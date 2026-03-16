```markdown
---
name: manpads-system-launcher-and-rocket
description: Low-cost proof-of-concept MANPADS-style guided rocket and launcher prototype using ESP32, MPU6050, folding fins, canard stabilization, and consumer electronics
triggers:
  - manpads rocket launcher
  - esp32 flight controller
  - guided rocket prototype
  - canard stabilization firmware
  - mpu6050 rocket imu
  - openrocket simulation
  - 3d printed rocket launcher
  - folding fin rocket design
---

# MANPADS System Launcher and Rocket Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection

A proof-of-concept guided rocket and launcher system built with consumer electronics and 3D-printed components. The rocket features folding fins, canard stabilization, an ESP32 flight computer, and an MPU6050 IMU. The launcher integrates GPS, compass, and barometric sensors for orientation and telemetry. Total hardware cost: ~$96.

---

## What It Does

- **Rocket flight computer**: ESP32-based controller reads IMU data, drives canard servos for stabilization, manages ignition sequencing
- **Launcher system**: GPS + compass + barometer integration for target orientation and telemetry
- **Mechanical design**: Fusion 360 CAD for rocket body, folding fins, canard assembly, and launcher tube
- **Simulation**: OpenRocket files for aerodynamic stability analysis and motor selection
- **Firmware**: Arduino/ESP-IDF firmware for both rocket and launcher subsystems

---

## Repository Structure

```
/
├── CAD/                    # Fusion 360 .f3d files for rocket and launcher
├── Firmware/
│   ├── Rocket/             # ESP32 flight controller firmware
│   └── Launcher/           # Launcher sensor and telemetry firmware
├── Simulation/             # OpenRocket .ork simulation files
└── Documentation/          # System flow diagrams, BOM, specs
```

---

## Hardware Stack

### Rocket
| Component | Purpose |
|---|---|
| ESP32 | Flight computer / main MCU |
| MPU6050 | 6-DOF IMU (accel + gyro) |
| Micro servos (x2) | Canard fin actuation |
| E-match / igniter | Motor ignition |
| LiPo cell | Power supply |

### Launcher
| Component | Purpose |
|---|---|
| ESP32 | Main controller |
| GPS module (NEO-6M or similar) | Position fix |
| HMC5883L / QMC5883 | Compass / heading |
| BMP280 / MS5611 | Barometric altitude |
| Relay module | Fire control circuit |

---

## Firmware: Rocket Flight Controller

### Core IMU Read + Canard Control Loop (ESP32 / Arduino framework)

```cpp
#include <Wire.h>
#include <MPU6050.h>
#include <ESP32Servo.h>

MPU6050 imu;
Servo canardPitch;
Servo canardYaw;

// PID state
float pitchIntegral = 0, yawIntegral = 0;
float prevPitchErr = 0, prevYawErr = 0;

const float Kp = 1.2f, Ki = 0.01f, Kd = 0.4f;
const int SERVO_CENTER = 90;
const int SERVO_RANGE  = 30; // ±30 degrees max deflection

void setup() {
  Wire.begin();
  imu.initialize();
  if (!imu.testConnection()) {
    Serial.println("MPU6050 connection failed");
    while (1);
  }

  canardPitch.attach(18); // GPIO18
  canardYaw.attach(19);   // GPIO19
  canardPitch.write(SERVO_CENTER);
  canardYaw.write(SERVO_CENTER);

  Serial.begin(115200);
}

float pidUpdate(float error, float &integral, float &prevError, float dt) {
  integral += error * dt;
  float derivative = (error - prevError) / dt;
  prevError = error;
  return Kp * error + Ki * integral + Kd * derivative;
}

void loop() {
  static unsigned long lastTime = micros();
  unsigned long now = micros();
  float dt = (now - lastTime) / 1e6f;
  lastTime = now;

  int16_t ax, ay, az, gx, gy, gz;
  imu.getMotion6(&ax, &ay, &az, &gx, &gy, &gz);

  // Convert raw gyro to deg/s (MPU6050 default ±250°/s scale)
  float pitchRate = gy / 131.0f;
  float yawRate   = gz / 131.0f;

  // Target: zero rotation rate (stabilization mode)
  float pitchCmd = pidUpdate(pitchRate, pitchIntegral, prevPitchErr, dt);
  float yawCmd   = pidUpdate(yawRate,   yawIntegral,   prevYawErr,   dt);

  // Clamp and apply
  pitchCmd = constrain(pitchCmd, -SERVO_RANGE, SERVO_RANGE);
  yawCmd   = constrain(yawCmd,   -SERVO_RANGE, SERVO_RANGE);

  canardPitch.write(SERVO_CENTER + (int)pitchCmd);
  canardYaw.write(SERVO_CENTER   + (int)yawCmd);

  delayMicroseconds(2000); // ~500 Hz loop
}
```

### Complementary Filter for Attitude Estimation

```cpp
// Fuse accelerometer angle with gyro integration
float compFilter(float accelAngle, float gyroRate, float prevAngle, float dt, float alpha = 0.98f) {
  return alpha * (prevAngle + gyroRate * dt) + (1.0f - alpha) * accelAngle;
}

float accelPitch(int16_t ax, int16_t ay, int16_t az) {
  return atan2f((float)ay, sqrtf((float)ax * ax + (float)az * az)) * RAD_TO_DEG;
}
```

---

## Firmware: Launcher System

### GPS + Compass + Barometer Initialization

```cpp
#include <Wire.h>
#include <TinyGPS++.h>
#include <QMC5883LCompass.h>
#include <Adafruit_BMP280.h>
#include <HardwareSerial.h>

TinyGPSPlus     gps;
QMC5883LCompass compass;
Adafruit_BMP280 baro;
HardwareSerial  gpsSerial(1); // UART1

void launcherSetup() {
  Wire.begin();
  gpsSerial.begin(9600, SERIAL_8N1, 16, 17); // RX=GPIO16, TX=GPIO17

  compass.init();
  compass.setCalibration(-500, 500, -500, 500, -500, 500); // calibrate per unit

  if (!baro.begin(0x76)) {
    Serial.println("BMP280 not found");
  }
  baro.setSampling(Adafruit_BMP280::MODE_NORMAL,
                   Adafruit_BMP280::SAMPLING_X2,
                   Adafruit_BMP280::SAMPLING_X16,
                   Adafruit_BMP280::FILTER_X4,
                   Adafruit_BMP280::STANDBY_MS_500);
}

struct LauncherTelemetry {
  double  lat, lng;
  float   altitudeMSL;
  float   heading;
  bool    gpsFix;
};

LauncherTelemetry readTelemetry() {
  LauncherTelemetry t = {};

  // Feed GPS
  while (gpsSerial.available()) gps.encode(gpsSerial.read());

  t.gpsFix = gps.location.isValid();
  if (t.gpsFix) {
    t.lat = gps.location.lat();
    t.lng = gps.location.lng();
  }

  t.altitudeMSL = baro.readAltitude(1013.25f); // standard sea-level pressure

  compass.read();
  t.heading = compass.getAzimuth(); // 0–360 degrees

  return t;
}
```

### Fire Control Relay

```cpp
const int RELAY_PIN    = 25;
const int ARM_PIN      = 26; // physical arm switch
const int LAUNCH_DELAY = 3000; // ms countdown

void fireControlSetup() {
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(ARM_PIN,   INPUT_PULLUP);
  digitalWrite(RELAY_PIN, LOW);
}

bool fireLaunch() {
  if (digitalRead(ARM_PIN) != LOW) {
    Serial.println("System not armed");
    return false;
  }
  Serial.println("FIRE in 3...");
  delay(LAUNCH_DELAY);
  digitalWrite(RELAY_PIN, HIGH);
  delay(500); // e-match pulse duration
  digitalWrite(RELAY_PIN, LOW);
  Serial.println("Launch complete");
  return true;
}
```

---

## OpenRocket Simulation Workflow

1. Open `.ork` file in [OpenRocket](https://openrocket.info/) (free, Java-based)
2. Verify stability margin (target: 1.0–2.0 calibers at launch)
3. Select motor from database matching the prototype's 24mm or 29mm mount
4. Run simulation → check apogee, max velocity, max acceleration
5. Export CSV for post-processing thrust curves

```python
# Parse OpenRocket CSV export
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv("simulation_export.csv", skiprows=5)
df.columns = df.columns.str.strip()

plt.figure(figsize=(10, 4))
plt.plot(df["Time (s)"], df["Altitude (m)"], label="Altitude")
plt.plot(df["Time (s)"], df["Vertical velocity (m/s)"], label="Velocity")
plt.xlabel("Time (s)")
plt.legend()
plt.title("OpenRocket Simulation")
plt.tight_layout()
plt.savefig("sim_plot.png")
```

---

## Configuration

### PID Tuning Constants (Rocket firmware)

```cpp
// Tune these for your specific airframe and motor
const float Kp = 1.2f;   // Proportional — increase for faster response
const float Ki = 0.01f;  // Integral     — increase to correct steady drift
const float Kd = 0.4f;   // Derivative   — increase to reduce overshoot
```

### MPU6050 Full-Scale Range

```cpp
// In setup(), optionally set higher range for high-G flight
imu.setFullScaleAccelRange(MPU6050_ACCEL_FS_16); // ±16g
imu.setFullScaleGyroRange(MPU6050_GYRO_FS_2000); // ±2000°/s
// Update scale factor: gyro = raw / 16.4f at 2000°/s range
```

### platformio.ini

```ini
[env:esp32dev]
platform  = espressif32
board     = esp32dev
framework = arduino
lib_deps  =
    electroniccats/MPU6050 @ ^1.3.0
    madhephaestus/ESP32Servo @ ^0.13.0
    mikalhart/TinyGPSPlus @ ^1.0.3
    mprograms/QMC5883LCompass @ ^1.2.0
    adafruit/Adafruit BMP280 Library @ ^2.6.8
monitor_speed = 115200
```

---

## Integration Patterns

### Rocket State Machine

```cpp
enum FlightState { IDLE, ARMED, BOOST, COAST, APOGEE, DESCENT };
FlightState state = IDLE;

void updateStateMachine(float accelMag, float altitudeDelta) {
  switch (state) {
    case IDLE:
      if (/* arm signal received */ false) state = ARMED;
      break;
    case ARMED:
      if (accelMag > 3.0f) state = BOOST; // 3g threshold for launch detect
      break;
    case BOOST:
      if (accelMag < 1.1f) state = COAST; // motor burnout
      break;
    case COAST:
      if (altitudeDelta < -0.5f) state = APOGEE; // descending
      break;
    case APOGEE:
      // deploy recovery, cut power to servos
      state = DESCENT;
      break;
    default:
      break;
  }
}
```

### Launcher–Rocket Communication (optional UART link)

```cpp
// Launcher sends launch command over serial before ignition
void sendLaunchPacket(HardwareSerial &link) {
  uint8_t packet[] = { 0xAA, 0x01, 0xFF }; // header, cmd, checksum
  link.write(packet, sizeof(packet));
}

// Rocket receives and validates
bool receiveLaunchCommand(HardwareSerial &link) {
  if (link.available() >= 3) {
    uint8_t buf[3];
    link.readBytes(buf, 3);
    return (buf[0] == 0xAA && buf[1] == 0x01 && buf[2] == 0xFF);
  }
  return false;
}
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| MPU6050 not detected | I2C address mismatch | Check AD0 pin; address is 0x68 (low) or 0x69 (high) |
| Servos jittering at rest | PID gains too high / noise | Lower Kp, add low-pass filter on gyro |
| GPS no fix indoors | Weak signal | Test outdoors with clear sky view; cold start takes 1–3 min |
| Compass heading drifts | Hard/soft iron interference | Run compass calibration routine, keep away from motors |
| Relay fires immediately | ARM_PIN floating | Ensure INPUT_PULLUP and arm switch wired correctly |
| Rocket unstable in sim | CG behind CP | Add nose weight or lengthen nose cone in CAD |
| High current draw on 3.3V | All peripherals active | Stagger initialization; use separate 3.3V LDO for sensors |

### MPU6050 I2C Scan

```cpp
void i2cScan() {
  for (uint8_t addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.printf("Device at 0x%02X\n", addr);
    }
  }
}
```

---

## Resources

- [Full development archive (Google Drive)](https://drive.google.com/drive/folders/17zpks6_R59H0iXJaGkTrtp1SzIFFAQtY?usp=drive_link) — BOM, assembly media, test footage, flow diagrams
- [OpenRocket](https://openrocket.info/) — free rocket simulation
- [ESP32 Arduino core](https://github.com/espressif/arduino-esp32)
- [MPU6050 datasheet](https://invensense.tdk.com/products/motion-tracking/6-axis/mpu-6050/)
- [30-second overview video](https://www.youtube.com/shorts/zFn__6_LdTc)
- [Full 5-minute system overview](https://www.youtube.com/watch?v=DDO2EvXyncE&t=59s)
```
