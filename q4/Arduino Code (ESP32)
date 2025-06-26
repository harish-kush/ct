#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <SoftwareSerial.h>
#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// WiFi credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Server endpoint
const char* serverURL = "https://your-server.com/api/airquality";

// Sensor pins
#define DHT_PIN 4
#define DHT_TYPE DHT22
#define MQ135_PIN 36
#define PMS_RX 16
#define PMS_TX 17

// Display settings
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1

// Initialize sensors
DHT dht(DHT_PIN, DHT_TYPE);
SoftwareSerial pmsSerial(PMS_RX, PMS_TX);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Data structure
struct AirQualityData {
  float temperature;
  float humidity;
  float co2;
  int pm25;
  int pm10;
  unsigned long timestamp;
};

void setup() {
  Serial.begin(115200);
  pmsSerial.begin(9600);
  dht.begin();
  
  // Initialize display
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 allocation failed");
    for(;;);
  }
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  
  // Connect to WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");
  
  // Warm up sensors
  delay(10000);
}

void loop() {
  AirQualityData data = readSensors();
  displayData(data);
  sendToServer(data);
  delay(30000); // Send data every 30 seconds
}

AirQualityData readSensors() {
  AirQualityData data;
  
  // Read DHT22
  data.temperature = dht.readTemperature();
  data.humidity = dht.readHumidity();
  
  // Read MQ-135 (CO2 approximation)
  int sensorValue = analogRead(MQ135_PIN);
  data.co2 = map(sensorValue, 0, 4095, 400, 5000); // Rough conversion
  
  // Read PMS7003
  if (pmsSerial.available()) {
    readPMSData(data);
  }
  
  data.timestamp = millis();
  return data;
}

void readPMSData(AirQualityData &data) {
  uint8_t buffer[32];
  int idx = 0;
  
  while (pmsSerial.available()) {
    buffer[idx++] = pmsSerial.read();
    if (idx >= 32) break;
  }
  
  if (buffer[0] == 0x42 && buffer[1] == 0x4d) {
    data.pm25 = (buffer[12] << 8) | buffer[13];
    data.pm10 = (buffer[10] << 8) | buffer[11];
  }
}

void displayData(AirQualityData data) {
  display.clearDisplay();
  display.setCursor(0, 0);
  
  display.println("Air Quality Monitor");
  display.println("==================");
  display.print("Temp: "); display.print(data.temperature); display.println("C");
  display.print("Humidity: "); display.print(data.humidity); display.println("%");
  display.print("CO2: "); display.print(data.co2); display.println(" ppm");
  display.print("PM2.5: "); display.print(data.pm25); display.println(" ug/m3");
  display.print("PM10: "); display.print(data.pm10); display.println(" ug/m3");
  
  display.display();
}

void sendToServer(AirQualityData data) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(serverURL);
    http.addHeader("Content-Type", "application/json");
    
    // Create JSON payload
    StaticJsonDocument<300> doc;
    doc["temperature"] = data.temperature;
    doc["humidity"] = data.humidity;
    doc["co2"] = data.co2;
    doc["pm25"] = data.pm25;
    doc["pm10"] = data.pm10;
    doc["timestamp"] = data.timestamp;
    doc["device_id"] = WiFi.macAddress();
    
    String jsonString;
    serializeJson(doc, jsonString);
    
    int httpResponseCode = http.POST(jsonString);
    
    if (httpResponseCode > 0) {
      Serial.print("HTTP Response: ");
      Serial.println(httpResponseCode);
    } else {
      Serial.print("Error: ");
      Serial.println(httpResponseCode);
    }
    
    http.end();
  }
}
