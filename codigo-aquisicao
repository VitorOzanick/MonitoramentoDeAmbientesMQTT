#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <BME280I2C.h>
#include <Wire.h>

#define SERIAL_BAUD 115200

// Identificação do local
//#define CAMPUS "campus1"
#define BLOCO  "A"
#define SALA   "101"

// Pinos
#define LDR_PIN A0
#define PIR_PIN D4
#define LED_PIN D3


#define TOPIC_SENSORES "campus/bloco/" BLOCO "/sala/" SALA "/sensores"
#define TOPIC_CMD_LUZ  "campus/bloco/" BLOCO "/sala/" SALA "/comandos/luz"

// Wi-Fi
const char* ssid     = "";
const char* password = "";

// MQTT
const char* mqtt_server = "broker.hivemq.com";
WiFiClient espClient;
PubSubClient client(espClient);




// Envio periódico
const unsigned long SEND_INTERVAL_MS = 2000; // 2s
unsigned long lastSend = 0;

// PIR debounce
const unsigned long PIR_DEBOUNCE_MS = 200; // 200 ms
bool motionDetected = false;
unsigned long lastPirChange = 0;
int lastPirRaw = LOW;

// LDR leitura
int ldrValue = 0;

// LED estado
bool ledState = false;

// BME280
BME280I2C bme;

void ensureWifi() {
  if (WiFi.status() == WL_CONNECTED) return;

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  Serial.print("Conectando Wi-Fi");
  unsigned long t0 = millis();

  while (WiFi.status() != WL_CONNECTED && millis() - t0 < 20000) {
    delay(500);
    Serial.print(".");
    yield(); // IMPORTANTE
  }

  Serial.println(WiFi.status() == WL_CONNECTED ? "\nWi-Fi OK" : "\nWi-Fi FALHOU");
}


void reconnectMQTT() {
  while (!client.connected() && WiFi.status() == WL_CONNECTED) {
    Serial.print("Conectando ao broker MQTT...");
    char clientId[32];
    snprintf(clientId, sizeof(clientId), "ESP8266-%06X", ESP.getChipId());

    if (client.connect(clientId)) {
      Serial.println("Conectado!");
      client.subscribe(TOPIC_CMD_LUZ);
      Serial.print("Inscrito: ");
      Serial.println(TOPIC_CMD_LUZ);
    } else {
      Serial.print("Falha MQTT, estado: ");
      Serial.print(client.state());
      Serial.println(" (tentando novamente em 5s)");
      delay(5000);
    }
  }
}


void mqttCallback(char* topic, byte* payload, unsigned int length) {
  if (strcmp(topic, TOPIC_CMD_LUZ) != 0) return;

  if (length == 1 && payload[0] == 'L') {
    ledState = true;
    digitalWrite(LED_PIN, HIGH);
  } else if (length == 1 && payload[0] == 'D') {
    ledState = false;
    digitalWrite(LED_PIN, LOW);
  }
}


bool bmeOk = false;

void setupBME() {
  Wire.begin();
  if (bme.begin()) {
    bmeOk = true;
    Serial.println("BME280 OK");
  } else {
    Serial.println("BME280 NAO encontrado (continuando sem ele)");
  }
}


void setup() {
  Serial.begin(SERIAL_BAUD);

  pinMode(PIR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  ensureWifi();

  setupBME();

  client.setServer(mqtt_server, 1883);
  client.setCallback(mqttCallback);
  reconnectMQTT();
}

void loop() {
  ensureWifi();
  if (!client.connected()) reconnectMQTT();
  client.loop();

  // PIR debounce
  int pirRaw = digitalRead(PIR_PIN);
  unsigned long now = millis();
  if (pirRaw != lastPirRaw) {
    lastPirChange = now;
    lastPirRaw = pirRaw;
  }
  if (now - lastPirChange > PIR_DEBOUNCE_MS) {
    motionDetected = (pirRaw == HIGH);
  }

  
  bool lightCurrentlyOn = ledState; // usa o estado lógico, não o pino

  // LDR
  ldrValue = analogRead(LDR_PIN);

  float temp = NAN, hum = NAN, pres = NAN;
  
  if (bmeOk) {
    bme.read(pres, temp, hum,
             BME280::TempUnit_Celsius,
             BME280::PresUnit_Pa);
  }

  // Tratamento de NaN -> valores default/últimos válidos (simples)
  if (isnan(temp)) temp = -100.0;       // sentinel
  if (isnan(hum))  hum  = -1.0;
  if (isnan(pres)) pres = -1.0;

  // Envio periódico
  if (now - lastSend >= SEND_INTERVAL_MS) {
    lastSend = now;
    // Monta JSON
    StaticJsonDocument<256> doc;
    doc["presenca"]     = motionDetected ? 1 : 0;
    doc["luz"]          = lightCurrentlyOn ? "ligada" : "desligada";
    doc["temperatura"]  = temp;
    doc["umidade"]      = hum;
    doc["pressao"]      = pres;

    char buf[256];
    size_t n = serializeJson(doc, buf, sizeof(buf));

    // Publica no tópico padronizado por sala
    bool ok = client.publish(TOPIC_SENSORES, buf, n);
    Serial.print("Publish [");
    Serial.print(TOPIC_SENSORES);
    Serial.print("] ");
    Serial.println(ok ? "OK" : "FALHA");

    Serial.print("Payload: "); Serial.println(buf);
  }

  yield();

}
