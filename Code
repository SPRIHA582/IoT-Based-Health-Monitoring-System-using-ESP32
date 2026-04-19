// -------- LIBRARIES --------
#include <Wire.h>
#include "MAX30105.h"
#include "spo2_algorithm.h"
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// -------- OLED --------
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// -------- MQ2 --------
#define MQ2_PIN 34

// -------- DS18B20 --------
#define ONE_WIRE_BUS 4
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// -------- MAX30102 --------
MAX30105 particleSensor;
bool maxOK = false;

#define BUFFER_SIZE 50
uint32_t irBuffer[BUFFER_SIZE];
uint32_t redBuffer[BUFFER_SIZE];

int32_t spo2 = 0;
int8_t validSPO2 = 0;
int32_t heartRate = 0;
int8_t validHeartRate = 0;

// -------- VARIABLES --------
float tempC = 0;
int alcohol = 0;
String sugarStatus = "";

// -------- GAS AVERAGE --------
int readGasAverage() {
  int sum = 0;
  for (int i = 0; i < 10; i++) {
    sum += analogRead(MQ2_PIN);
    delay(10);
  }
  return sum / 10;
}

void setup()
{
  Serial.begin(115200);
  delay(3000); // stabilize power

  // OLED INIT
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED not found");
    while(1);
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(WHITE);
  display.setCursor(0,0);
  display.println("System Starting...");
  display.display();
  delay(1500);

  sensors.begin();

  // MAX30102 INIT
  if (particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30102 detected");
    particleSensor.setup();
    particleSensor.setPulseAmplitudeRed(0x0A);
    particleSensor.setPulseAmplitudeIR(0x0A);
    maxOK = true;
  } else {
    Serial.println("MAX30102 not found");
  }
}

void loop()
{
  // -------- MQ SENSOR --------
  alcohol = readGasAverage();

  if (alcohol < 2900) {
    sugarStatus = "LOW VOC";
  }
  else if (alcohol < 3500) {
    sugarStatus = "MEDIUM VOC";
  }
  else {
    sugarStatus = "HIGH VOC";
  }

  // -------- TEMPERATURE --------
  sensors.requestTemperatures();
  tempC = sensors.getTempCByIndex(0);

  String tempStatus;
  if (tempC < 36) tempStatus = "LOW";
  else if (tempC <= 37.5) tempStatus = "NORMAL";
  else tempStatus = "HIGH";

  // -------- MAX30102 (SAFE BLOCKING) --------
  if (maxOK) {
    for (byte i = 0; i < BUFFER_SIZE; i++) {

      unsigned long startTime = millis();

      while (!particleSensor.available()) {
        particleSensor.check();

        // timeout protection
        if (millis() - startTime > 1000) {
          validHeartRate = 0;
          validSPO2 = 0;
          break;
        }
      }

      redBuffer[i] = particleSensor.getRed();
      irBuffer[i] = particleSensor.getIR();
      particleSensor.nextSample();
    }

    maxim_heart_rate_and_oxygen_saturation(
      irBuffer, BUFFER_SIZE, redBuffer,
      &spo2, &validSPO2,
      &heartRate, &validHeartRate
    );
  }

  // -------- FORMAT VALUES --------
  String hrText = (validHeartRate) ? String(heartRate) : "N/A";
  String spo2Text = (validSPO2) ? String(spo2) : "N/A";

  // -------- SYSTEM STATUS --------
  String systemStatus = "SAFE";

  if (validHeartRate && (heartRate < 60 || heartRate > 100))
    systemStatus = "UNSAFE";

  if (validSPO2 && spo2 < 92)
    systemStatus = "UNSAFE";

  if (alcohol > 3500)
    systemStatus = "UNSAFE";

  // -------- OLED DISPLAY --------
  display.clearDisplay();

  display.setCursor(0,0);
  display.print("Gas:");
  display.print(alcohol);
  display.print(" ");
  display.println(sugarStatus);

  display.setCursor(0,10);
  display.print("HR:");
  display.println(hrText);

  display.setCursor(0,20);
  display.print("SpO2:");
  display.print(spo2Text);
  display.println("%");

  display.setCursor(0,30);
  display.print("Temp:");
  display.print(tempC);
  display.print(" ");
  display.println(tempStatus);

  display.setCursor(0,45);
  display.print("Status:");
  display.println(systemStatus);

  if (!maxOK) {
    display.setCursor(0,55);
    display.println("Pulse OFF");
  }

  display.display();

  // -------- SERIAL --------
  Serial.println("----- HEALTH DATA -----");
  Serial.print("Gas: "); Serial.println(alcohol);
  Serial.print("HR: "); Serial.println(hrText);
  Serial.print("SpO2: "); Serial.println(spo2Text);
  Serial.print("Temp: "); Serial.println(tempC);
  Serial.print("Status: "); Serial.println(systemStatus);
  Serial.println("----------------------");

  delay(1500);
}
