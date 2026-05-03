#define BLYNK_TEMPLATE_ID "TMPL2j-qqsh4X"
#define BLYNK_TEMPLATE_NAME "New Template"
#define BLYNK_AUTH_TOKEN "ARIL-nB8SDK1xRNmK2o_erPTcESKTS8B"

#include <WiFiS3.h>
#include <BlynkSimpleWifi.h>
#include <ArduinoHttpClient.h>
#include <ArduinoJson.h>
#include <Servo.h>

// ================= WIFI =================
char ssid[] = "wadi3";
char pass[] = "00000000";

// ================= BLYNK =================
BlynkTimer timer;

// ================= SERVOS =================
Servo horizontal;
Servo vertical;

int servohori = 90;
int servovert = 90;   // départ au milieu

const int servoHoriPin = 9;
const int servoVertPin = 10;

const int servohoriLimitHigh = 175;
const int servohoriLimitLow  = 5;
const int servovertLimitHigh = 180;   // corrigé
const int servovertLimitLow  = 0;     // corrigé

// ================= LDR =================
const int ldrlt = A0;
const int ldrld = A1;
const int ldrrd = A2;
const int ldrrt = A3;

// ================= BUTTONS =================
const int buttonLeft  = 2;
const int buttonRight = 3;
const int buttonUp    = 4;
const int buttonDown  = 5;
const int buttonMode  = 7;

// ================= WEATHER =================
const char* weatherServer = "api.open-meteo.com";
const int weatherPort = 80;

String weatherPath =
"/v1/forecast?latitude=35&longitude=3&current=cloud_cover,wind_speed_10m,is_day&timezone=GMT";

WiFiClient wifi;
HttpClient client(wifi, weatherServer, weatherPort);

// ================= VARIABLES =================
int currentMode = 0; // 0 AUTO | 1 MANUAL/REPARATION | 2 WEATHER

bool repairNotified = false;

unsigned long lastWeatherCheck = 0;
const unsigned long weatherInterval = 60000;

// pour le bouton mode physique
bool lastModeButtonState = HIGH;
unsigned long lastDebounceTime = 0;
const unsigned long debounceDelay = 200;

// ================= UTILS =================
int clampValue(int value, int low, int high) {
  if (value < low) return low;
  if (value > high) return high;
  return value;
}

void writeServos() {
  servohori = clampValue(servohori, servohoriLimitLow, servohoriLimitHigh);
  servovert = clampValue(servovert, servovertLimitLow, servovertLimitHigh);

  horizontal.write(servohori);
  vertical.write(servovert);
}

// ================= BLYNK SEND =================
void sendDataToBlynk() {
  Blynk.virtualWrite(V0, servohori);
  Blynk.virtualWrite(V1, servovert);

  String modeStr;
  if (currentMode == 0) modeStr = "AUTO";
  else if (currentMode == 1) modeStr = "MANUAL";
  else modeStr = "WEATHER";

  Blynk.virtualWrite(V2, modeStr);

  // LED MODE
  Blynk.virtualWrite(V3, currentMode == 0 ? 255 : 0);
  Blynk.virtualWrite(V4, currentMode == 1 ? 255 : 0);
  Blynk.virtualWrite(V5, currentMode == 2 ? 255 : 0);
}

// ================= ALERT SYSTEM =================
void sendRepairAlert() {
  if (currentMode == 1 && !repairNotified) {
    Blynk.logEvent("repair_mode", "Attention Mode Reparation Active");
    repairNotified = true;
  }

  if (currentMode != 1) {
    repairNotified = false;
  }
}

// ================= CHANGE MODE =================
void changeMode() {
  currentMode = (currentMode + 1) % 3;
  sendRepairAlert();

  Serial.print("Mode changed to: ");
  Serial.println(currentMode);
}

// ================= BLYNK CONTROL =================
BLYNK_WRITE(V6) {
  if (param.asInt() == 1) {
    changeMode();
  }
}

// Servo horizontal gauche
BLYNK_WRITE(V7) {
  if (param.asInt() == 1 && currentMode == 1) {
    servohori -= 4;
    writeServos();
  }
}

// Servo horizontal droite
BLYNK_WRITE(V8) {
  if (param.asInt() == 1 && currentMode == 1) {
    servohori += 4;
    writeServos();
  }
}

// Servo vertical haut
BLYNK_WRITE(V9) {
  if (param.asInt() == 1 && currentMode == 1) {
    servovert += 4;
    writeServos();
  }
}

// Servo vertical bas
BLYNK_WRITE(V10) {
  if (param.asInt() == 1 && currentMode == 1) {
    servovert -= 4;
    writeServos();
  }
}

// ================= AUTO MODE =================
void autoTrackingMode() {
  int lt = analogRead(ldrlt);
  int rt = analogRead(ldrrt);
  int ld = analogRead(ldrld);
  int rd = analogRead(ldrrd);

  int dvert  = ((lt + rt) / 2) - ((ld + rd) / 2);
  int dhoriz = ((lt + ld) / 2) - ((rt + rd) / 2);

  if (abs(dvert) > 40) servovert += (dvert > 0) ? 1 : -1;
  if (abs(dhoriz) > 40) servohori += (dhoriz > 0) ? -1 : 1;

  writeServos();
  delay(15);
}

// ================= MANUAL MODE =================
void manualMode() {
  bool left  = (digitalRead(buttonLeft) == LOW);
  bool right = (digitalRead(buttonRight) == LOW);
  bool up    = (digitalRead(buttonUp) == LOW);
  bool down  = (digitalRead(buttonDown) == LOW);

  if (left)  servohori -= 4;
  if (right) servohori += 4;

  if (up)    servovert += 4;
  if (down)  servovert -= 4;

  writeServos();
  delay(100);
}

// ================= WEATHER MODE =================
void weatherMode() {
  if (millis() - lastWeatherCheck < weatherInterval) return;

  lastWeatherCheck = millis();

  client.get(weatherPath);
  int statusCode = client.responseStatusCode();
  String response = client.responseBody();

  if (statusCode == 200) {
    DynamicJsonDocument doc(2048);
    DeserializationError error = deserializeJson(doc, response);

    if (!error) {
      float wind = doc["current"]["wind_speed_10m"];
      int clouds = doc["current"]["cloud_cover"];
      int isDay  = doc["current"]["is_day"];

      if (wind > 10) {
        servohori = 90;
        servovert = 10;
      }
      else if (clouds > 80 || isDay == 0) {
        servohori = 175;
        servovert = 20;
      }

      writeServos();
    }
  }
}

// ================= READ PHYSICAL MODE BUTTON =================
void checkModeButton() {
  bool reading = digitalRead(buttonMode);

  if (reading != lastModeButtonState) {
    lastDebounceTime = millis();
  }

  if ((millis() - lastDebounceTime) > debounceDelay) {
    if (lastModeButtonState == HIGH && reading == LOW) {
      changeMode();
    }
  }

  lastModeButtonState = reading;
}

// ================= SETUP =================
void setup() {
  Serial.begin(115200);

  horizontal.attach(servoHoriPin);
  vertical.attach(servoVertPin);

  pinMode(buttonLeft, INPUT_PULLUP);
  pinMode(buttonRight, INPUT_PULLUP);
  pinMode(buttonUp, INPUT_PULLUP);
  pinMode(buttonDown, INPUT_PULLUP);
  pinMode(buttonMode, INPUT_PULLUP);

  writeServos();

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  timer.setInterval(1000L, sendDataToBlynk);

  Serial.println("System Ready");
}

// ================= LOOP =================
void loop() {
  Blynk.run();
  timer.run();

  checkModeButton();
  sendRepairAlert();

  if (currentMode == 0) {
    autoTrackingMode();
  }
  else if (currentMode == 1) {
    manualMode();
  }
  else {
    weatherMode();
  }
}