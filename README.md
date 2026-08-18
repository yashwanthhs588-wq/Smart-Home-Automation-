# Smart-Home-Automation-
Smart Home Automation is an IoT-based project that enables users to control and monitor home appliances remotely using ESP32, sensors, relays, and Wi-Fi. It improves convenience, energy efficiency, security, and provides a smart and automated home environment.
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>
#include <Servo.h>

#define DHTPIN 4
#define DHTTYPE DHT11

#define LDR_PIN A0
#define SOIL_PIN A1

#define TRIG_PIN 5
#define ECHO_PIN 6

#define BUZZER_PIN 8
#define LED_PIN 9
#define SERVO_PIN 10

#define PIR_PIN 2
#define RELAY_PIN 7

#define TEMP_THRESHOLD 35.0
#define DIST_THRESHOLD 20.0
#define LIGHT_THRESHOLD 375
#define SOIL_DRY_THRESHOLD 750

DHT dht(DHTPIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo myServo;

unsigned long previousMillis = 0;
const long interval = 2000;
int screenState = 0;

float readUltrasonicDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return -1.0;
  return (duration * 0.0343) / 2.0;
}

void setup() {
  Serial.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);

  myServo.attach(SERVO_PIN);
  myServo.write(0);

  dht.begin();
  Wire.begin();
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Smart Home Sys");
  lcd.setCursor(0, 1);
  lcd.print("Starting...");

  Serial.println(F("=========================================="));
  Serial.println(F("   SMART HOME MONITORING SYSTEM READY     "));
  Serial.println(F("=========================================="));
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long currentMillis = millis();

  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;

    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    int ldrVal = analogRead(LDR_PIN);
    int soilVal = analogRead(SOIL_PIN);
    float distance = readUltrasonicDistance();
    bool pirMotion = (digitalRead(PIR_PIN) == HIGH);

    bool dhtError = false;
    if (isnan(temp) || isnan(hum)) {
      dhtError = true;
      temp = 0.0;
      hum = 0.0;
    }

    int soilPercent = map(soilVal, 1023, 370, 0, 100);
    soilPercent = constrain(soilPercent, 0, 100);

    bool tempAlert = (temp > TEMP_THRESHOLD);
    bool distAlert = (distance > 0 && distance < DIST_THRESHOLD);
    bool soilAlert = (soilVal > SOIL_DRY_THRESHOLD);
    
    bool alertActive = tempAlert || distAlert || soilAlert || pirMotion;
    bool nightLightActive = (!alertActive && ldrVal < LIGHT_THRESHOLD);

    if (distAlert) {
      myServo.write(90);
    } else {
      myServo.write(0);
    }

    if (soilAlert) {
      digitalWrite(RELAY_PIN, HIGH);
    } else {
      digitalWrite(RELAY_PIN, LOW);
    }

    if (alertActive) {
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(LED_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(LED_PIN, nightLightActive ? HIGH : LOW);
    }

    Serial.println(F("\n--- [ SENSOR TELEMETRY REPORT ] ---"));
    if (dhtError) {
      Serial.println(F("DHT11 Sensor : ERROR (Check Wiring)"));
    } else {
      Serial.print(F("Temperature  : "));
      Serial.print(temp, 1);
      Serial.print(F(" C  "));
      Serial.println(tempAlert ? F("[WARNING: HIGH TEMP]") : F("[NORMAL]"));
      
      Serial.print(F("Humidity     : "));
      Serial.print(hum, 1);
      Serial.println(F(" %"));
    }

    Serial.print(F("Light Level  : "));
    Serial.print(ldrVal);
    Serial.print(F(" (ADC) -> "));
    Serial.println(ldrVal < LIGHT_THRESHOLD ? F("DARK") : F("BRIGHT"));

    Serial.print(F("Soil Moisture: "));
    Serial.print(soilPercent);
    Serial.print(F("% (ADC: "));
    Serial.print(soilVal);
    Serial.print(F(") -> "));
    Serial.println(soilAlert ? F("[ALERT: SOIL DRY]") : F("[WET/OPTIMAL]"));

    if (distance < 0) {
      Serial.println(F("Distance     : Out of Range / No Echo"));
    } else {
      Serial.print(F("Distance     : "));
      Serial.print(distance, 1);
      Serial.print(F(" cm  "));
      Serial.println(distAlert ? F("[ALERT: OBJECT NEAR]") : F("[CLEAR]"));
    }

    Serial.print(F("PIR Motion   : "));
    Serial.println(pirMotion ? F("[DETECTED]") : F("[CLEAR]"));

    Serial.print(F("Relay State  : "));
    Serial.println(soilAlert ? F("ON (Water Pump)") : F("OFF"));

    Serial.print(F("Servo Motor  : "));
    Serial.println(distAlert ? F("OPEN (90 deg)") : F("CLOSED (0 deg)"));

    Serial.print(F("Buzzer State : "));
    Serial.println(alertActive ? F("ACTIVE (ALARM)") : F("OFF"));
    
    Serial.print(F("LED State    : "));
    if (alertActive || nightLightActive) {
      Serial.print(F("ON ("));
      Serial.print(alertActive ? F("ALARM INDICATION") : F("NIGHT LIGHT"));
      Serial.println(F(")"));
    } else {
      Serial.println(F("OFF (STANDBY)"));
    }
    Serial.println(F("-----------------------------------"));

    lcd.clear();
    if (screenState == 0) {
      lcd.setCursor(0, 0);
      lcd.print("Temp: ");
      lcd.print(temp, 1);
      lcd.print(" C");
      
      lcd.setCursor(0, 1);
      lcd.print("Humidity: ");
      lcd.print(hum, 1);
      lcd.print("%");
      
      screenState = 1;
    } else if (screenState == 1) {
      lcd.setCursor(0, 0);
      lcd.print("LDR: ");
      lcd.print(ldrVal < LIGHT_THRESHOLD ? "Dark" : "Bright");
      
      lcd.setCursor(0, 1);
      if (distance < 0) {
        lcd.print("Dist: Out Range");
      } else {
        lcd.print("Dist: ");
        lcd.print(distance, 1);
        lcd.print(" cm");
      }
      
      screenState = 2;
    } else if (screenState == 2) {
      lcd.setCursor(0, 0);
      lcd.print("Soil: ");
      lcd.print(soilPercent);
      lcd.print("%");
      
      lcd.setCursor(0, 1);
      lcd.print("Status: ");
      lcd.print(soilAlert ? "Dry" : "Optimal");
      
      screenState = 3;
    } else {
      lcd.setCursor(0, 0);
      lcd.print("PIR: ");
      lcd.print(pirMotion ? "Motion" : "Clear");
      
      lcd.setCursor(0, 1);
      lcd.print("Relay: ");
      lcd.print(soilAlert ? "ON" : "OFF");
      
      screenState = 0;
    }
  }
}
