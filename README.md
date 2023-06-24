# Practica - Base de datos con MySQL y Node-red

## Introducción

### Descripción

En esta práctica se utilizó una tarjeta **ESP32**, un sensor ultrasónico de distancia y un sensor **DHT22**, se utilizó una base de datos para la captura de valores de estos sensores y así obtener un historial.

## Material Necesario

Para realizar esta práctica se ocuparon las siguientes herramientas y componentes:

- phpMyAdmin
- [Wokwi](https://wokwi.com/)
- [Node-red](https://nodejs.org/en)
- ESP32
- Sensor ultrasónico
- DHT22


## Instrucciones

### Ajustes de phpMyAdmin

1. Instalar [XAMPP](https://www.apachefriends.org/) y ejecutar el archivo descargado. Una vez abierto, dar click en *Start* en los apartados *Apache* y *MySQL*, posteriormente, dar click en *Admin* del apartado *MySQL*.

![](https://github.com/Michellecg/Base-de-datos/blob/main/XAMPP.PNG)

2. Se crea una nueva base de datos con un cotejamiento ```utf8mb4_general_ci```.

3. Crear una nueva tabla de 6 columnas con los siguientes criterios:

![](https://github.com/Michellecg/Base-de-datos/blob/main/Tabla.PNG)

4. Dentro de Node-red, colocar un bloque ```mqtt in```, este hará la conexión mediante una IP.

5. Copiar la IP ```3.82.39.163```, porteriormente dirigirse al ```localhost:1880```, hacer *doble click* en **mqtt in**, en el apartado *Server*, hacer click en el ícono de lápiz y pegar en el *Server*. Cambiar el nombre del bloque en *Topic*, en este caso se utilizó el nombre de "MichelleCG".

6. En la esquina superior derecha (debajo del ícono de propiedades), se encuentra un ícono de triángulo, hacer click en el apartado _dashboard_, seleccionar _tab_>_group_>_spacer_. Cambiar el nombre de Tap a _Base de datos_.

7. Añadir un bloque de _function_ para la transferencia de datos entre Wokwi y phpMyAdmin. Hacer *doble click* en el bloque y añadir el siguiente código:
```
var query = "INSERT INTO `trabajo 1`(`ID`, `FECHA`, `DEVICE`, `TEMPERATURA`, `HUMEDAD`, `DISTANCIA`) VALUES (NULL, current_timestamp(), '";
query = query + msg.payload.DEVICE + "','";
query = query + msg.payload.TEMPERATURA + "','";
query = query + msg.payload.HUMEDAD + "','";
query = query + msg.payload.DISTANCIA + "');'";
msg.topic = query;
return msg;
``` 

8. Añadir un bloque _mysql_, hacer doble click y en la parte _Database_ se agrega el nombre de la base de datos y en el ícono de lápiz, verificar que el servidor sea el mismo que en la base de datos, esto se puede ver en la pestaña de _privilegios_ de _phpMyAdmin_. Por último, en _User_ escribir _root_.

9. Añadir otros 3 bloques de _function_ para _Temperatura_, _Humedad_ y _Distancia_.
Hacer doble click en cada uno de los bloques y escribir los siguientes códigos, respectivamente:
```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
```
```
msg.payload = msg.payload.HUMEDAD;
msg.topic = "HUMEDAD";
return msg;
```
```
msg.payload = msg.payload.DISTANCIA;
msg.topic = "DISTANCIA";
return msg;
```

10. Una vez hecho, ingresar a la página de [WOKWI](https://wokwi.com/) se selecciona la tarjeta **ESP32**, un sensor ultrasónico y un sensor **DHT22**.

11. Una vez seleccionado la tarjeta ```Esp32``` junto a los componentes, en la parte izquierda se encuentra la pestaña de código donde se agrega lo siguiente:
```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 15;
DHTesp dhtSensor;

const int Trigger=13; //Pin digital 13 para el Trigger del sensor
const int Echo=12; //Pin digital 3 para el Echo del sensor
// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "44.195.202.69";
String username_mqtt="MichelleCG";
String password_mqtt="12345678";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger,OUTPUT); //Pin como salida
  pinMode(Echo,INPUT); //Pin como entrada
  digitalWrite(Trigger, LOW);
}

void loop() {

delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
long t; //Tiempo de demora en llegar el eco
long d; //Distancia en centimetros
digitalWrite(Trigger, HIGH);
delayMicroseconds(10);
digitalWrite(Trigger, LOW);

t=pulseIn(Echo,HIGH);
d=t/59;

  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    //doc["Anho"] = 2022;
    //doc["Empresa"] = "Educatronicos";
    doc["DISTANCIA"] = String(d);
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("MichelleCG", output.c_str());
  }
}

``` 
7. En la pestaña de *Library Manager*, instalar las librerías de **ArduinoJson**, **PubSubClient** y **DHT sensor library for ESPx** como se muestra en la siguente imagen.

![](https://github.com/Michellecg/Node-Red_con-DHT22-y-ultrasonico/blob/main/Lib_Ult_DHT22.PNG)

8. Hacer la conexión del **sensor ultrasónico de distancia** y **DHT22** a la tarjeta **ESP32** como se muestra en la siguente imagen.

![](https://github.com/Michellecg/Node-Red_con-DHT22-y-ultrasonico/blob/main/Conex_Ult_DHT22.PNG)

9. En Node-red, se debe observar de la siguiente manera el diagrama:

![](https://github.com/Michellecg/Base-de-datos/blob/main/Diagrama_m.PNG)

## Resultados

En phpMyAdmin se pueden observar los siguientes resultados de los datos monitoreados en Wokwi.

![](https://github.com/Michellecg/Base-de-datos/blob/main/Res_base_de_datos.PNG)

Mientras que en el dashboard se puede ver de la siguiente manera:

![](https://github.com/Michellecg/Base-de-datos/blob/main/Resultados_m.PNG)

# Créditos

Desarrollado por Michelle Cuatlapantzi González

- [GitHub](https://github.com/Michellecg/)