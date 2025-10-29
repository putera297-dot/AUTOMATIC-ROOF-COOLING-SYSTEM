#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>

#define DHTPIN_INDOOR 32
#define DHTPIN_OUTDOOR 33
#define DHTTYPE DHT22

#define WATER_PUMP_PIN 25

// WiFi credentials
const char* ssid = "ESP32put";
const char* password = "88888888";

// MQTT Broker
const char* mqtt_server = "broker.hivemq.com";  
const char* status_topic = "roof/status";
const char* indoor_topic = "roof/temp/indoor";
const char* outdoor_topic = "roof/temp/outdoor";
const char* alert_topic = "roof/alert";
const char* manual_control_topic = "roof/manual";

WiFiClient espClient;
PubSubClient client(espClient);

DHT dhtIndoor(DHTPIN_INDOOR, DHTTYPE);
DHT dhtOutdoor(DHTPIN_OUTDOOR, DHTTYPE);

bool manualPumpState = false;
bool autoCoolingTriggered = false;
float tempThreshold = 30.0; // Suhu ambang

void setup_wifi() {
  delay(10);
  Serial.println("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("WiFi connected!");
}

void callback(char* topic, byte* message, unsigned int length) {
  String msg;
  for (int i = 0; i < length; i++) {
    msg += (char)message[i];
  }

  if (String(topic) == manual_control_topic) {
    if (msg == "ON") {
      manualPumpState = true;
      digitalWrite(WATER_PUMP_PIN, HIGH);
      client.publish(status_topic, "Water Cooling is Running (Manual)");
    } else if (msg == "OFF") {
      manualPumpState = false;
      digitalWrite(WATER_PUMP_PIN, LOW);
      client.publish(status_topic, "Water Cooling is Standby (Manual)");
    }
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    if (client.connect("ESP32Client")) {
      Serial.println("connected");
      client.subscribe(manual_control_topic);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(WATER_PUMP_PIN, OUTPUT);
  digitalWrite(WATER_PUMP_PIN, LOW);

  dhtIndoor.begin();
  dhtOutdoor.begin();

  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  float tempIndoor = dhtIndoor.readTemperature();
  float tempOutdoor = dhtOutdoor.readTemperature();

  if (isnan(tempIndoor) || isnan(tempOutdoor)) {
    Serial.println("Gagal baca sensor DHT22");
    return;
  }

  // Hantar data suhu ke Node-RED
  char buffer[10];
  dtostrf(tempIndoor, 4, 1, buffer);
  client.publish(indoor_topic, buffer);
  dtostrf(tempOutdoor, 4, 1, buffer);
  client.publish(outdoor_topic, buffer);

  bool highTempDetected = (tempIndoor > tempThreshold || tempOutdoor > tempThreshold);

  if (highTempDetected && !manualPumpState) {
    digitalWrite(WATER_PUMP_PIN, HIGH);
    if (!autoCoolingTriggered) {
      client.publish(alert_topic, "🔥 Suhu tinggi dikesan! Pam diaktifkan.");
      client.publish(status_topic, "Water Cooling is Running (Auto)");
      autoCoolingTriggered = true;
    }
  } else if (!manualPumpState) {
    digitalWrite(WATER_PUMP_PIN, LOW);
    if (autoCoolingTriggered) {
      client.publish(status_topic, "Water Cooling is Standby (Auto)");
      autoCoolingTriggered = false;
    }
  }

  delay(5000);
}
