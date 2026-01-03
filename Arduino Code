#include <Wire.h>
#include "Adafruit_SHTC3.h"
#include "MAX30105.h"     // from SparkFun MAX3010x library
#include "heartRate.h"

// ---- Sensors ----
Adafruit_SHTC3 shtc3 = Adafruit_SHTC3();
MAX30105 particleSensor;

// ---- LED pins ----
const int LED_RED    = 5;  // Stress
const int LED_BLUE   = 6;  // Fatigue
const int LED_YELLOW = 7;  // Excited/Tired

// ---- Globals ----
float bpm = 0;
float tempC = 0, humidity = 0;
unsigned long lastBeat = 0;   
void setup() {
  Serial.begin(115200);
  while (!Serial) {}

  // LEDs
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_BLUE, OUTPUT);
  pinMode(LED_YELLOW, OUTPUT);
  digitalWrite(LED_RED, LOW);
  digitalWrite(LED_BLUE, LOW);
  digitalWrite(LED_YELLOW, LOW);

  Wire.begin();                 // safe (SparkFun lib can call it too)
  Wire.setClock(100000);        // calm I2C

  // ---- Start SHTC3 ----
  if (!shtc3.begin()) {
    Serial.println("Couldn't find SHTC3 (check 3V3/GND/SDA(A4)/SCL(A5)).");
  } else {
    Serial.println("SHTC3 ready");
  }

  // ---- Start MAX30102 ----
  if (!particleSensor.begin(Wire, I2C_SPEED_STANDARD)) {
    Serial.println("MAX30102 not found. Check wiring (3V3/GND/SDA/SCL)!");
    while (1) { delay(1000); }
  }
  // Low-power, safe config
  particleSensor.setup();                 // load library defaults
  particleSensor.setLEDMode(2);           // Red + IR (no Green)
  particleSensor.setSampleRate(100);      // 100 Hz
  particleSensor.setPulseWidth(69);       // shortest pulse (lower energy)
  particleSensor.setPulseAmplitudeIR(0x10);  // << IR amplitude REQUIRED for getIR()
  particleSensor.setPulseAmplitudeRed(0x10); // visible red kept low
  particleSensor.setPulseAmplitudeGreen(0x00);
}

void loop() {
  // ---- Read Pulse Sensor (IR) ----
  long irValue = particleSensor.getIR();    // raw IR
  if (checkForBeat(irValue)) {
    unsigned long now = millis();
    unsigned long delta = now - lastBeat;
    lastBeat = now;
    if (delta > 0) {
      float candidate = 60.0 * 1000.0 / (float)delta;
      if (candidate > 30 && candidate < 220) bpm = candidate;  // sanity clamp
    }
  }

  // ---- Read SHTC3 ----
  sensors_event_t humidityEvent, tempEvent;
  if (shtc3.getEvent(&humidityEvent, &tempEvent)) {
    tempC = tempEvent.temperature;
    humidity = humidityEvent.relative_humidity;
  }

  // ---- Map to emotions ----
  //  - STRESSED: high BPM AND high humidity
  //  - EXCITED:  high BPM AND NOT high humidity
  //  - TIRED:    low BPM
  //  - CALM:     otherwise
  String emotion = "CALM";

  // turn all off by default
  digitalWrite(LED_RED, LOW);
  digitalWrite(LED_BLUE, LOW);
  digitalWrite(LED_YELLOW, LOW);

  if (bpm > 100 && humidity > 70) {
    emotion = "STRESSED";
    digitalWrite(LED_RED, HIGH);
  } else if (bpm > 100 && humidity <= 70) {
    emotion = "EXCITED";
    digitalWrite(LED_YELLOW, HIGH);
  } else if (bpm > 0 && bpm < 60) {
    emotion = "TIRED";
    // simple "breathing" effect on BLUE
    static uint32_t t0 = millis();
    float phase = (millis() - t0) / 1000.0;
    int duty = 30 + (int)(25 * (0.5 + 0.5 * sinf(phase))); // 30–55
    analogWrite(LED_BLUE, duty);   // Nano 33 IoT supports PWM on D6
  } else {
    emotion = "CALM";
  }

  // ---- Print Debug (1 Hz) ----
  static uint32_t lastPrint = 0;
  if (millis() - lastPrint >= 1000) {
    lastPrint = millis();
    Serial.print("IR="); Serial.print(irValue);
    Serial.print("  BPM="); Serial.print(bpm, 1);
    Serial.print("  Temp="); Serial.print(tempC, 2); Serial.print("C");
    Serial.print("  Hum="); Serial.print(humidity, 1); Serial.print("%");
    Serial.print("  -> "); Serial.println(emotion);
  }

  delay(5);
}





