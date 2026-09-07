An automated, 3D-printed smart plant waterer powered by an ESP32 and environmental sensors. View the live project showcase at https://o-hashemi.github.io/Squirtle-Smart-Garden/ 


CODE:


// ================================= LIBRARIES =================================
#include <WiFi.h>
#include <Wire.h>
#include <Adafruit_SHT31.h>
#include <FirebaseESP32.h>
#include <NTPClient.h>    
#include <WiFiUdp.h>        


#include "addons/TokenHelper.h"
#include "addons/RTDBHelper.h"

Adafruit_SHT31 sht31 = Adafruit_SHT31();


WiFiUDP ntpUDP;

NTPClient timeClient(ntpUDP, "pool.ntp.org", -14400); 

// ================================= PINS =================================
#define SOIL_PIN 34
#define OUTPUT_PIN 23
#define FLOAT_PIN 18

// ================================= OUTPUT LOGIC =================================
#define OUTPUT_ON HIGH
#define OUTPUT_OFF LOW

// ================================= CALIBRATION =================================
int dryValue = 4000;
int wetValue = 900;

// ================================= SETTINGS =================================
const int waterThreshold = 45;
const unsigned long outputTime = 10000;
const unsigned long cooldownTime = 15000;
const unsigned long readInterval = 1000;

// ================================= NETWORK CREDENTIALS =================================
const char* ssid = "Coextro_TP-Link_56EE"; 
const char* password = "36348794";  

// ================================= FIREBASE CREDENTIALS =================================

#define FIREBASE_HOST "squirtle-smart-garden-default-rtdb.firebaseio.com"
#define FIREBASE_AUTH "qRwtDNqIxdnhz984j1GFKBy2YQ3OYZkjLgwQoDIh"

// ================================= FIREBASE OBJECTS =================================
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

// ================================= VARIABLES =================================
int moistureRaw = 0;
int moisturePercent = 0;

unsigned long lastOutputTime = 0;
unsigned long lastReadTime = 0;

bool sht31Found = false;

String pumpStatus = "OFF";
String waterLevelStatus = "OK";

// ================================= SOIL FUNCTION =================================
int readSoilMoisture() {
  long sum = 0;
  for (int i = 0; i < 30; i++) {
    sum += analogRead(SOIL_PIN);
    delay(10);
  }
  return sum / 30;
}

// ================================= SETUP =================================
void setup() {
  Serial.begin(115200);
  delay(500); 
  Serial.println("\n--- BOOTING SQUIRTLE SMART GARDEN ---");


  pinMode(OUTPUT_PIN, OUTPUT);
  digitalWrite(OUTPUT_PIN, OUTPUT_OFF);
  pinMode(FLOAT_PIN, INPUT_PULLUP);
  analogReadResolution(12);
  

  Wire.begin(21, 22);
  
  if (sht31.begin(0x44)) {
    sht31Found = true;
    Serial.println("SHT31 DETECTED");
  } else {
    sht31Found = false;
    Serial.println("SHT31 OFFLINE");
  }


  Serial.println("Attempting to connect to WiFi...");
  WiFi.begin(ssid, password);
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected successfully!");
    

   timeClient.begin();
    

  config.host = FIREBASE_HOST;
   config.signer.tokens.legacy_token = FIREBASE_AUTH;
    
   Firebase.reconnectWiFi(true);
 Firebase.begin(&config, &auth);
    Serial.println("Firebase Handshake Complete!");
  } else {
    Serial.println("\nWiFi Connection Failed. Checking local loops...");
  }

  Serial.println("SYSTEM READY");
}

// ================================= LOOP =================================
void loop() {

  timeClient.update();
  
  unsigned long currentTime = millis();
  
  if (currentTime - lastReadTime >= readInterval) {
    lastReadTime = currentTime;
    
   // ================================= FLOAT SWITCH =================================
    int waterLevel = digitalRead(FLOAT_PIN);
    
  if (waterLevel == LOW) {
      digitalWrite(OUTPUT_PIN, OUTPUT_OFF);
      pumpStatus = "LOCKED";
      waterLevelStatus = "LOW";
      Serial.println("NO WATER -> PUMP LOCKED");
      
 
   if (WiFi.status() == WL_CONNECTED && Firebase.ready()) {
        Firebase.setString(fbdo, "/sensorData/pumpStatus", pumpStatus);
        Firebase.setString(fbdo, "/sensorData/waterLevel", waterLevelStatus);
      }
    } else {
      waterLevelStatus = "OK";
    }
    
   // ================================= SOIL SENSOR =================================
    moistureRaw = readSoilMoisture();
    moisturePercent = map(moistureRaw, dryValue, wetValue, 0, 100);
    moisturePercent = constrain(moisturePercent, 0, 100);
    
   // ================================= SERIAL PRINT =================================
    Serial.print("Raw: ");
    Serial.print(moistureRaw);
    Serial.print(" | Moisture: ");
    Serial.print(moisturePercent);
    Serial.print("% | ");
    
   // ================================= TEMP + HUMIDITY =================================
    float temperature = 0.0;
    float humidity = 0.0;
    
   if (sht31Found) {
      temperature = sht31.readTemperature();
      humidity = sht31.readHumidity();
      
   Serial.print("Temp: ");
      Serial.print(temperature);
      Serial.print("C | Humidity: ");
      Serial.print(humidity);
      Serial.print("% | ");
    } else {
      Serial.print("SHT31 OFFLINE | ");
    }
        // ================================= WATER LOGIC =================================
    if (moisturePercent < waterThreshold) {
      if (currentTime - lastOutputTime >= cooldownTime) {
        Serial.println("DRY -> PUMP ON");
        pumpStatus = "ON";
        
  if (WiFi.status() == WL_CONNECTED && Firebase.ready()) {
            Firebase.setString(fbdo, "/lastWatered", timeClient.getFormattedTime());
        }
        
  digitalWrite(OUTPUT_PIN, OUTPUT_ON);
        delay(outputTime);
        digitalWrite(OUTPUT_PIN, OUTPUT_OFF);
        
  lastOutputTime = millis();
        pumpStatus = "OFF";
      } else {
        Serial.println("DRY -> WAITING");
        pumpStatus = "OFF";
      }
    } else {
      Serial.println("SOIL OK -> PUMP OFF");
      pumpStatus = "OFF";
      digitalWrite(OUTPUT_PIN, OUTPUT_OFF);
    }
    
  // ================================= FIREBASE UPLOAD =================================
    if (WiFi.status() == WL_CONNECTED && Firebase.ready()) {
      Firebase.setInt(fbdo, "/sensorData/moistureRaw", moistureRaw);
      Firebase.setInt(fbdo, "/sensorData/moisturePercent", moisturePercent);
      Firebase.setString(fbdo, "/sensorData/pumpStatus", pumpStatus);
      Firebase.setString(fbdo, "/sensorData/waterLevel", waterLevelStatus);
      
  if (sht31Found) {
        Firebase.setFloat(fbdo, "/sensorData/temperature", temperature);
        Firebase.setFloat(fbdo, "/sensorData/humidity", humidity);
      }
      Serial.println("-> Firebase Updated!");
    } else {
      Serial.println("-> Firebase Not Ready");
    }
  }
}
