```markdown
---
name: aeris10-plfm-radar
description: Open-source 10.5 GHz Pulse Linear Frequency Modulated phased array radar system (AERIS-10) with FPGA signal processing, STM32 control, and Python GUI
triggers:
  - "set up AERIS-10 radar"
  - "configure PLFM radar firmware"
  - "radar beamforming phased array"
  - "FPGA radar signal processing"
  - "STM32 radar control firmware"
  - "pulse compression doppler radar"
  - "ADAR1000 phase shifter configuration"
  - "radar chirp waveform generation"
---

# AERIS-10 PLFM Phased Array Radar

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

AERIS-10 is an open-source 10.5 GHz Pulse Linear Frequency Modulated (PLFM) phased array radar. It comes in two versions: **AERIS-10N (Nexus)** with 3 km range using an 8×16 patch antenna array, and **AERIS-10E (Extended)** with 20 km range using a 32×16 slotted waveguide array and 10 W GaN amplifiers. The system uses an XC7A100T FPGA for signal processing, an STM32F746xx MCU for system management, and a Python GUI for visualization.

---

## Repository Structure

```
PLFM_RADAR/
├── 4_Schematics and Boards Layout/
│   └── 4_7_Production Files/   # Gerber files, BOM
├── 9_Firmware/
│   ├── 9_1_FPGA/               # VHDL/Verilog (Vivado project)
│   ├── 9_2_STM32/              # STM32F746 C firmware
│   └── 9_3_GUI/                # Python GUI
├── 10_docs/
│   ├── assembly_guide.md
│   └── Hardware/Enclosure/     # 3D printable files
└── 8_Utils/                    # Images, utilities
```

---

## Hardware Overview

| Subsystem | Key ICs |
|---|---|
| Clock distribution | AD9523-1 |
| TX/RX frequency synthesis | ADF4382 |
| Chirp generation | High-speed DAC |
| Phase shifting (beamforming) | ADAR1000 (4× 4-ch) |
| Front-end LNA/PA | ADTR1107 (16×) |
| Extended-range PA | QPA2962 GaN (16× 10 W) |
| FPGA | Xilinx XC7A100T (Artix-7) |
| MCU | STM32F746xx |
| ADC (power monitoring) | ADS7830 |
| DAC (bias control) | DAC5578 |

---

## Getting Started

### Prerequisites

```bash
# Python GUI
python --version   # 3.8+ required

# FPGA toolchain
# Install Xilinx Vivado (2022.x or later recommended)
# https://www.xilinx.com/support/download.html

# STM32 firmware
# Install STM32CubeIDE or arm-none-eabi-gcc toolchain
arm-none-eabi-gcc --version
```

### Python GUI Setup

```bash
git clone https://github.com/NawfalMotii79/PLFM_RADAR.git
cd PLFM_RADAR/9_Firmware/9_3_GUI

pip install -r requirements.txt
python radar_gui.py
```

### Building STM32 Firmware (GCC)

```bash
cd 9_Firmware/9_2_STM32

# Using Make (if Makefile present)
make all

# Flash via ST-Link
make flash
# or
st-flash write build/aeris10.bin 0x08000000
```

### Building FPGA Bitstream (Vivado TCL)

```bash
cd 9_Firmware/9_1_FPGA

# Non-interactive build
vivado -mode batch -source build.tcl

# Open project interactively
vivado aeris10_fpga.xpr
```

---

## STM32 Firmware — Key Patterns

### Power Sequencing

The STM32 controls power-up order to protect components. Follow the sequencing defined in the Power Management Excel file.

```c
/* power_seq.h */
typedef enum {
    PWR_STATE_OFF = 0,
    PWR_STATE_DIGITAL,   /* 3.3V / 1.8V digital rails */
    PWR_STATE_SYNTH,     /* ADF4382 VCC */
    PWR_STATE_RF,        /* ADTR1107 / ADAR1000 */
    PWR_STATE_PA,        /* QPA2962 GaN (Extended only) */
    PWR_STATE_READY
} PowerState_t;

void PowerSeq_Up(void);
void PowerSeq_Down(void);
```

```c
/* power_seq.c */
#include "power_seq.h"
#include "stm32f7xx_hal.h"

#define DELAY_MS(x)  HAL_Delay(x)

/* GPIO bank aliases — match your schematic net names */
#define EN_3V3_PIN   GPIO_PIN_0
#define EN_3V3_PORT  GPIOB
#define EN_RF_PIN    GPIO_PIN_4
#define EN_RF_PORT   GPIOC
#define EN_PA_PIN    GPIO_PIN_5
#define EN_PA_PORT   GPIOC

static PowerState_t current_state = PWR_STATE_OFF;

void PowerSeq_Up(void) {
    /* Step 1: Digital rails */
    HAL_GPIO_WritePin(EN_3V3_PORT, EN_3V3_PIN, GPIO_PIN_SET);
    DELAY_MS(50);

    /* Step 2: Initialize clock generator */
    AD9523_Init();
    DELAY_MS(10);

    /* Step 3: Frequency synthesizers */
    ADF4382_Init(ADF4382_TX);
    ADF4382_Init(ADF4382_RX);
    DELAY_MS(20);

    /* Step 4: RF front-end */
    HAL_GPIO_WritePin(EN_RF_PORT, EN_RF_PIN, GPIO_PIN_SET);
    DELAY_MS(30);

    /* Step 5: Phase shifters */
    for (uint8_t i = 0; i < 4; i++) {
        ADAR1000_Init(i);
    }

    /* Step 6: PA boards (Extended version only) */
#ifdef AERIS10_EXTENDED
    HAL_GPIO_WritePin(EN_PA_PORT, EN_PA_PIN, GPIO_PIN_SET);
    DELAY_MS(100);
    PA_SetBias(); /* DAC5578 Vg control */
#endif

    current_state = PWR_STATE_READY;
}

void PowerSeq_Down(void) {
    /* Reverse order */
#ifdef AERIS10_EXTENDED
    HAL_GPIO_WritePin(EN_PA_PORT, EN_PA_PIN, GPIO_PIN_RESET);
    DELAY_MS(50);
#endif
    HAL_GPIO_WritePin(EN_RF_PORT, EN_RF_PIN, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(EN_3V3_PORT, EN_3V3_PIN, GPIO_PIN_RESET);
    current_state = PWR_STATE_OFF;
}
```

---

### ADAR1000 Phase Shifter Driver

The ADAR1000 is a 4-channel X/Ku-band beamformer IC controlled over SPI.

```c
/* adar1000.h */
#define ADAR1000_COUNT       4
#define ADAR1000_CHANNELS    4

/* Register addresses (from ADAR1000 datasheet) */
#define ADAR1000_CH1_TX_GAIN   0x001
#define ADAR1000_CH1_TX_PHASE  0x002
#define ADAR1000_CH1_RX_GAIN   0x010
#define ADAR1000_CH1_RX_PHASE  0x011
#define ADAR1000_TX_ENABLES    0x038
#define ADAR1000_RX_ENABLES    0x039
#define ADAR1000_SW_CTRL       0x042

void ADAR1000_Init(uint8_t device_idx);
void ADAR1000_SetTxPhase(uint8_t dev, uint8_t ch, uint16_t phase_deg_x10);
void ADAR1000_SetRxPhase(uint8_t dev, uint8_t ch, uint16_t phase_deg_x10);
void ADAR1000_SetTxGain(uint8_t dev, uint8_t ch, uint8_t gain);
void ADAR1000_SetRxGain(uint8_t dev, uint8_t ch, uint8_t gain);
void ADAR1000_ApplyBeam(uint8_t dev);
```

```c
/* adar1000.c */
#include "adar1000.h"
#include "spi.h"

/* SPI chip-select lines per device */
static GPIO_TypeDef* cs_ports[ADAR1000_COUNT] = {GPIOA, GPIOA, GPIOB, GPIOB};
static uint16_t      cs_pins[ADAR1000_COUNT]  = {GPIO_PIN_4, GPIO_PIN_5,
                                                   GPIO_PIN_0, GPIO_PIN_1};

static void ADAR1000_Write(uint8_t dev, uint16_t reg, uint8_t data) {
    uint8_t tx[3];
    /* 3-byte SPI: [addr_high][addr_low][data] — write bit = 0 */
    tx[0] = (reg >> 8) & 0x7F;
    tx[1] = reg & 0xFF;
    tx[2] = data;

    HAL_GPIO_WritePin(cs_ports[dev], cs_pins[dev], GPIO_PIN_RESET);
    HAL_SPI_Transmit(&hspi1, tx, 3, HAL_MAX_DELAY);
    HAL_GPIO_WritePin(cs_ports[dev], cs_pins[dev], GPIO_PIN_SET);
}

void ADAR1000_Init(uint8_t dev) {
    /* Software reset */
    ADAR1000_Write(dev, 0x000, 0x81);
    HAL_Delay(1);
    /* Default: all channels enabled, TR switch to TX */
    ADAR1000_Write(dev, ADAR1000_TX_ENABLES, 0x0F);
    ADAR1000_Write(dev, ADAR1000_RX_ENABLES, 0x0F);
}

/* phase_deg_x10: phase in units of 0.1 degrees, 0–3599 */
void ADAR1000_SetTxPhase(uint8_t dev, uint8_t ch, uint16_t phase_deg_x10) {
    /* ADAR1000 phase word = phase(deg) / 360 * 128  (7-bit) */
    uint8_t phase_word = (uint8_t)((phase_deg_x10 * 128UL) / 3600UL);
    uint16_t reg = ADAR1000_CH1_TX_PHASE + (ch * 0x10);
    ADAR1000_Write(dev, reg, phase_word);
}

void ADAR1000_SetRxPhase(uint8_t dev, uint8_t ch, uint16_t phase_deg_x10) {
    uint8_t phase_word = (uint8_t)((phase_deg_x10 * 128UL) / 3600UL);
    uint16_t reg = ADAR1000_CH1_RX_PHASE + (ch * 0x10);
    ADAR1000_Write(dev, reg, phase_word);
}

void ADAR1000_ApplyBeam(uint8_t dev) {
    /* Latch all pending phase/gain updates */
    ADAR1000_Write(dev, ADAR1000_SW_CTRL, 0x01);
}
```

---

### Beamforming — Computing Phase Shifts

```c
/* beamforming.c
 * Computes per-element phase shifts for a linear array
 * to steer to azimuth angle `theta_deg`.
 */
#include <math.h>
#include <stdint.h>
#include "adar1000.h"

#define FREQ_HZ        10.5e9
#define C_MPS          3.0e8
#define LAMBDA_M       (C_MPS / FREQ_HZ)   /* ~0.02857 m */
#define ELEMENT_SPACING_M  (LAMBDA_M / 2.0) /* half-wavelength */
#define NUM_TX_ELEMENTS    16

/*
 * Steer all TX elements to angle theta_deg (azimuth).
 * Call after ADAR1000_Init() for all devices.
 */
void Beamform_SteerTx(float theta_deg) {
    float theta_rad = theta_deg * (float)M_PI / 180.0f;
    float sin_theta = sinf(theta_rad);

    for (int elem = 0; elem < NUM_TX_ELEMENTS; elem++) {
        /* Phase progression across array */
        float phase_rad = 2.0f * (float)M_PI
                          * ELEMENT_SPACING_M / LAMBDA_M
                          * (float)elem * sin_theta;

        /* Normalise to [0, 360) degrees */
        float phase_deg = fmodf(phase_rad * 180.0f / (float)M_PI, 360.0f);
        if (phase_deg < 0.0f) phase_deg += 360.0f;

        uint16_t phase_x10 = (uint16_t)(phase_deg * 10.0f);

        /* Map element index to device and channel */
        uint8_t dev = elem / ADAR1000_CHANNELS;
        uint8_t ch  = elem % ADAR1000_CHANNELS;

        ADAR1000_SetTxPhase(dev, ch, phase_x10);
    }

    /* Latch all devices simultaneously */
    for (uint8_t dev = 0; dev < ADAR1000_COUNT; dev++) {
        ADAR1000_ApplyBeam(dev);
    }
}
```

---

### ADF4382 Frequency Synthesizer Init

```c
/* adf4382.h */
typedef enum { ADF4382_TX = 0, ADF4382_RX = 1 } ADF4382_ID;

void ADF4382_Init(ADF4382_ID id);
void ADF4382_SetFreq(ADF4382_ID id, uint64_t freq_hz);
```

```c
/* adf4382.c — simplified register write sequence */
#include "adf4382.h"
#include "spi.h"

/* Reference from AD9523-1 output: 100 MHz */
#define REF_FREQ_HZ    100000000ULL
#define LO_FREQ_HZ     10500000000ULL  /* 10.5 GHz */

static void ADF4382_WriteReg(ADF4382_ID id, uint16_t addr, uint32_t val) {
    /* 3-byte SPI write: addr[15:8], addr[7:0], data[7:0] */
    uint8_t buf[3] = {
        (addr >> 8) & 0xFF,
         addr       & 0xFF,
         val        & 0xFF
    };
    /* Select correct CS based on id */
    SPI_CS_Assert(id);
    HAL_SPI_Transmit(&hspi2, buf, 3, HAL_MAX_DELAY);
    SPI_CS_Deassert(id);
}

void ADF4382_Init(ADF4382_ID id) {
    /* Soft reset */
    ADF4382_WriteReg(id, 0x000, 0x81);
    HAL_Delay(5);

    /* Configure reference path: REF_FREQ = 100 MHz */
    ADF4382_WriteReg(id, 0x004, 0x00); /* REF doubler off */

    /* Integer-N: N = LO / REF = 10500000000 / 100000000 = 105 */
    uint32_t N = (uint32_t)(LO_FREQ_HZ / REF_FREQ_HZ);
    ADF4382_WriteReg(id, 0x010, (N >> 8) & 0xFF);
    ADF4382_WriteReg(id, 0x011,  N       & 0xFF);

    /* Enable VCO, charge pump, output */
    ADF4382_WriteReg(id, 0x020, 0x01);
    HAL_Delay(10); /* Wait for lock */
}
```

---

### GaN PA Bias Control (QPA2962 via DAC5578)

```c
/* pa_bias.c
 * DAC5578 is an 8-channel 8-bit I2C DAC.
 * Used to set gate voltage (Vg) of QPA2962 for proper Idq.
 */
#include "stm32f7xx_hal.h"

#define DAC5578_ADDR_A   0x4C  /* PA board group A */
#define DAC5578_ADDR_B   0x4E  /* PA board group B */
#define DAC5578_CMD_WRITE_UPDATE  0x30

extern I2C_HandleTypeDef hi2c1;

static void DAC5578_SetChannel(uint8_t i2c_addr, uint8_t ch, uint8_t val) {
    /* Command byte: 0x3n where n = channel (0-7) */
    uint8_t buf[2] = {DAC5578_CMD_WRITE_UPDATE | (ch & 0x07), val};
    HAL_I2C_Master_Transmit(&hi2c1, i2c_addr << 1, buf, 2, HAL_MAX_DELAY);
}

/* Vg_mv: gate voltage in millivolts (e.g. -200 mV = 0 for GaN depletion) */
/* Map Vg range [-500, 0] mV -> DAC [0, 255]                               */
static uint8_t VgToDacCode(int32_t vg_mv) {
    /* Assumes resistor divider sets: DAC_out=0V -> Vg=-500mV,
                                       DAC_out=Vcc -> Vg=0V           */
    int32_t clamped = vg_mv < -500 ? -500 : vg_mv > 0 ? 0 : vg_mv;
    return (uint8_t)(((clamped + 500) * 255) / 500);
}

void PA_SetBias(void) {
    /* Nominal Idq target: ~100 mA per device -> Vg ~ -200 mV */
    int32_t vg_target_mv = -200;
    uint8_t dac_code = VgToDacCode(vg_target_mv);

    for (uint8_t ch = 0; ch < 8; ch++) {
        DAC5578_SetChannel(DAC5578_ADDR_A, ch, dac_code);
        DAC5578_SetChannel(DAC5578_ADDR_B, ch, dac_code);
    }
}

/* Read drain current via ADS7830 ADC (8-ch single-ended I2C ADC) */
#define ADS7830_ADDR_A   0x48
#define ADS7830_CMD(ch)  (0x84 | ((ch & 0x07) << 4))  /* single-ended */

uint8_t PA_ReadIdqRaw(uint8_t i2c_addr, uint8_t ch) {
    uint8_t cmd = ADS7830_CMD(ch);
    uint8_t result = 0;
    HAL_I2C_Master_Transmit(&hi2c1, i2c_addr << 1, &cmd, 1, HAL_MAX_DELAY);
    HAL_I2C_Master_Receive(&hi2c1,  i2c_addr << 1, &result, 1, HAL_MAX_DELAY);
    return result; /* 0-255, scale by Vref/shunt resistor */
}
```

---

### Stepper Motor Control (360° Mechanical Scan)

```c
/* stepper.c — step/dir driver for 360° scan */
#include "stm32f7xx_hal.h"

#define STEP_PIN   GPIO_PIN_6
#define STEP_PORT  GPIOD
#define DIR_PIN    GPIO_PIN_7
#define DIR_PORT   GPIOD

#define STEPS_PER_REV   200    /* 1.8° stepper */
#define MICROSTEP       16     /* 1/16 microstepping */
#define FULL_ROT_STEPS  (STEPS_PER_REV * MICROSTEP)  /* 3200 */

void Stepper_Step(uint32_t steps, uint8_t direction, uint32_t delay_us) {
    HAL_GPIO_WritePin(DIR_PORT, DIR_PIN,
                      direction ? GPIO_PIN_SET : GPIO_PIN_RESET);

    for (uint32_t i = 0; i < steps; i++) {
        HAL_GPIO_WritePin(STEP_PORT, STEP_PIN, GPIO_PIN_SET);
        /* Use TIM-based microsecond delay in production */
        HAL_Delay(1);
        HAL_GPIO_WritePin(STEP_PORT, STEP_PIN, GPIO_PIN_RESET);
        HAL_Delay(1);
    }
}

void Stepper_FullRotation(uint8_t clockwise) {
    Stepper_Step(FULL_ROT_STEPS, clockwise, 500);
}
```

---

## FPGA Signal Processing Pipeline

The FPGA (XC7A100T) implements the full radar DSP chain. Key module roles:

| Module | Function |
|---|---|
| `chirp_gen.v` | DDS-based LFM chirp via DAC |
| `iq_demod.v` | Baseband I/Q down-conversion |
| `cic_decimator.v` | Sample rate reduction |
| `pulse_compress.v` | Range FFT + matched filter |
| `doppler_fft.v` | Slow-time FFT for velocity |
| `mti_filter.v` | Moving Target Indicator (clutter cancel) |
| `cfar_detect.v` | Constant False Alarm Rate detection |
| `usb_interface.v` | Host data transfer |

### Example: Pulse Compression Concept (C pseudocode matching FPGA logic)

```c
#include <complex.h>
#include <math.h>
#include <stdint.h>

/* Matched filter for LFM pulse compression.
 * range_fft[]:  FFT of received IF signal (N points)
 * ref_fft[]:    FFT of reference chirp (pre-computed, conjugate)
 * output[]:     compressed range profile
 */
void PulseCompress(float complex *range_fft,
                   const float complex *ref_fft,
                   float complex *output,
                   uint32_t N)
{
    /* Multiply spectrum by conjugate of reference = matched filter */
    for (uint32_t i = 0; i < N; i++) {
        output[i] = range_fft[i] * conjf(ref_fft[i]);
    }
    /* IFFT gives compressed range profile — call your FFT library */
    /* ifft(output, N); */
}

/* LFM reference chirp generation (for pre-computing ref_fft) */
void GenerateLFMChirp(float complex *chirp, uint32_t N,
                      float bw_hz, float t_pulse_s, float fs_hz)
{
    float k = bw_hz / t_pulse_s;  /* chirp rate Hz/s */
    for (uint32_t n = 0; n < N; n++) {
        float t = (float)n / fs_hz;
        float phase = (float)M_PI * k * t * t;
        chirp[n] = cosf(phase) + I * sinf(phase);
    }
}
```

---

## Python GUI

```python
# 9_Firmware/9_3_GUI/radar_gui.py (usage pattern)
import serial
import numpy as np
import struct

SERIAL_PORT = "/dev/ttyUSB0"   # or "COM3" on Windows
BAUD_RATE   = 921600

class AerisRadar:
    def __init__(self, port: str = SERIAL_PORT):
        self.ser = serial.Serial(port, BAUD_RATE, timeout=1)

    def send_command(self, cmd_id: int, payload: bytes = b"") -> None:
        """Send framed command to STM32."""
        frame = struct.pack(">BB", cmd_id, len(payload)) + payload
        self.ser.write(frame)

    def set_beam_angle(self, azimuth_deg: float, elevation_deg: float) -> None:
        """Command beam steering angles."""
        payload = struct.pack(">ff", azimuth_deg, elevation_deg)
        self.send_command(0x10, payload)

    def read_detections(self) -> list[dict]:
        """Read CFAR detections from FPGA via USB/UART."""
        detections = []
        raw = self.ser.read(256)
        # Parse fixed-size detection records: range(m), velocity(m/s), az, el
        rec_size = 16  # 4 floats
        for i in range(0, len(raw) - rec_size + 1, rec_size):
            r, v, az, el = struct.unpack_from(">ffff", raw, i)
            detections.append({"range_m": r, "velocity_mps": v,
                                "azimuth_deg": az, "elevation_deg": el})
        return detections

    def close(self):
        self.ser.close()


if __name__ == "__main__":
    radar = AerisRadar()
    radar.set_beam_angle(0.0, 0.0)  # Broadside
    while True:
        targets = radar.read_detections()
        for t in targets:
            print(f"Range: {t['range_m']:.1f} m  "
                  f"Vel: {t['velocity_mps']:.1f} m/s  "
                  f"Az: {t['azimuth_deg']:.1f}°")
```

---

## GPS / IMU Integration

```c
/* gps_imu.c — reads GY-85 IMU for pitch/roll correction */
#include "stm32f7xx_hal.h"
#include <math.h>

/* GY-85 contains ADXL345 accel + ITG3205 gyro + HMC5883L mag */
#define ADXL345_ADDR   0x53
#define ADXL345_DATA_X0 0x32

extern I2C_HandleTypeDef hi2c2;

typedef struct { float pitch; float roll; } AttitudeAngles;

AttitudeAngles IMU_GetAttitude(void) {
    uint8_t buf[6];
    uint8_t reg = ADXL345_DATA_X0;
    HAL_I2C_Master_Transmit(&hi2c2, ADXL345_ADDR << 1, &reg, 1, HAL_MAX_DELAY);
    HAL_I2C_Master_Receive(&hi2c2,  ADXL345_ADDR << 1, buf, 6, HAL_MAX_DELAY);

    int16_t ax = (int16_t)(buf[1] << 8 | buf[0]);
    int16_t ay = (int16_t)(buf[3] << 8 | buf[2]);
    int16_t az = (int16_t)(buf[5] << 8 | buf[4]);

    AttitudeAngles att;
    att.pitch = atan2f((float)ay, sqrtf((float)(ax*ax + az*az))) * 180.0f / M_PI;
    att.roll  = atan2f(-(float)ax, (float)az) * 180.0f / M_PI;
    return att;
}

/* Correct target elevation for platform tilt */
float CorrectTargetElevation(float measured_el, AttitudeAngles att) {
    return measured_el - att.pitch;
}
```

---

## Configuration Reference

| Parameter | Location | Notes |
|---|---|---|
| LO frequency | `adf4382.c` → `LO_FREQ_HZ` | Default 10.5 GHz |
| Reference clock | `adf4382.c` → `REF_FREQ_HZ` | From AD9523-1, default 100 MHz |
| Element spacing | `beamforming.c` → `ELEMENT_SPACING_M` | λ/2 default |
| PA gate voltage | `pa_bias.c` → `vg_target_mv` | Tune for Idq target |
| Stepper microstep | `stepper.c` → `MICROSTEP` | Match driver DIP switches |
| Serial port (GUI) | `radar_gui.py` → `SERIAL_PORT` | Set per host OS |

---

## Version Differences (N vs E)

| Feature | AERIS-10N | AERIS-10E |
|---|---|---|
| Range | 3 km | 20 km |
| Antenna | 8×16 patch | 32×16 slotted waveguide |
| PA boards | No (ADTR1107 only) | 16× QPA2962 10 W GaN |
| Build flag | (default) | `#define AERIS10_EXTENDED` |

Enable Extended version in firmware:

```c
/* In your project-wide config header */
#define AERIS10_EXTENDED   1
```

---

## Troubleshooting

### No PLL Lock (ADF4382)
- Verify AD9523-1 is outputting 100 MHz reference before ADF4382 init
- Check SPI wiring: MOSI, SCK, CS polarity
- Confirm N divider value matches `LO_FREQ_HZ / REF_FREQ_HZ`
- Add `HAL_Delay(10)` after register writes if
