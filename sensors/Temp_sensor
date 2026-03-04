# MLX90614 Temperature Computation, Internal Storage (RAM), and Core Data Format Specification

**Version:** 1.0  
**Date:** 2026-03-04

---

## 1. Purpose

This document specifies:

- What temperatures the **MLX90614** measures (ambient and object)
- How the MLX90614 internally computes temperature and stores the computed results in its **RAM registers**
- In what data format the temperature reaches the processor core over **SMBus (I²C-compatible)**
- How software on the core converts the raw register value into **Kelvin** and **Celsius**

---

## 2. Scope

### 2.1 In scope

- MLX90614 RAM temperature registers:
  - `0x06` (Ta: ambient temperature)
  - `0x07` (To1: object temperature)
- RAM layout and **16-bit temperature encoding**
- SMBus/I²C read transaction for fetching the temperature words
- Raw-to-physical temperature conversion in core software

### 2.2 Out of scope

- Full SMBus packet error checking (PEC/CRC) and all command variants
- EEPROM calibration coefficients and advanced configuration options
- Exact optical/thermal design constraints (FOV, emissivity tuning, mechanical placement)

---

## 3. Terms and Definitions

**Ta (Ambient temperature)**  
Temperature of the sensor chip/package itself (die temperature).

**To (Object temperature)**  
Temperature of the observed external object, computed from IR radiation sensed by the thermopile and corrected using Ta and internal calibration.

**LSB**  
Least Significant Bit of the stored temperature code.

**SMBus**  
System Management Bus; an I²C-compatible two-wire bus used by MLX90614.

---

## 4. Temperature Outputs Provided by the Sensor

The sensor provides two main temperatures that the processor can read:

| Temperature | Meaning | RAM Address |
|---|---|---|
| Ta (Ambient) | Temperature of the sensor chip itself | `0x06` |
| To (Object) | Temperature of the external object | `0x07` |

**Note**
- In typical applications, the processor reads **To** (object temperature) from RAM address `0x07`.
- **Ta** (`0x06`) is often used for compensation, diagnostics, or logging.

---

## 5. Internal Computation Chain (How Temperature Is Computed Inside MLX90614)

The MLX90614 computes object temperature internally. The high-level data flow is:

1. IR radiation from object  
2. Thermopile sensing element converts IR energy to an electrical signal  
3. Analog amplifier conditions the thermopile signal  
4. 17-bit ADC digitizes the conditioned signal  
5. DSP performs temperature computation (including compensation based on ambient Ta and internal calibration)  
6. Computed temperatures are written into internal RAM registers (Ta at `0x06`, To at `0x07`)  
7. SMBus interface exposes RAM registers to the host processor  
8. RISC-V core reads raw register data and converts it to Kelvin/°C in software  

**Key point**
- The host does **not** compute To from raw IR ADC codes in the normal read flow; the MLX90614 DSP computes To and stores it as a ready-to-use value in RAM.

---

## 6. Internal Storage Model (RAM Registers)

### 6.1 RAM organization

- The MLX90614 contains RAM with **32 registers**.
- Each RAM register is **16 bits** wide.

### 6.2 Relevant RAM register addresses

| Address | Content |
|---|---|
| `0x04` | Raw IR sensor data |
| `0x05` | Raw IR sensor data (sensor2) |
| `0x06` | Ambient temperature (Ta) |
| `0x07` | Object temperature (To1) |
| `0x08` | Object temperature (To2) |

**Host usage guidance**
- Use `0x07` for object temperature **To1** unless the design explicitly uses the second object channel (`0x08`).

---

## 7. Stored Value Data Format (16-bit Word Layout)

### 7.1 Bit structure

Each temperature RAM register holds a 16-bit value:

- `bit15`: **error flag**
- `bit14..bit0`: **temperature code**

```
bit15            bit14..bit0
[error flag]     [temperature code]
```

### 7.2 Resolution (scaling)

- **1 LSB = 0.02 Kelvin**

Interpretation:
- The temperature code represents an absolute temperature in Kelvin, scaled by **0.02 K/LSB**.

---

## 8. Representation (Binary / Hex / Decimal)

The stored quantity is fundamentally a 16-bit binary word. The same word can be expressed as:
- Binary (bit pattern)
- Hexadecimal (datasheet / debugging representation)
- Decimal (numeric representation)

Example representation of the same 16-bit word:

| Format | Value |
|---|---|
| Binary | `0011101011110111` |
| Hex | `0x3AF7` |
| Decimal | `15095` |

All three represent the identical register content.

---

## 9. Host-side Temperature Conversion (Core Software)

### 9.1 Conversion formula

Let `raw_value` = `(bit14..bit0)` interpreted as an integer temperature code.

**Temperature in Kelvin**

\[
T(K) = raw\_value \times 0.02
\]

**Temperature in Celsius**

\[
T(°C) = (raw\_value \times 0.02) - 273.15
\]

**Important implementation note**
- `bit15` is an error flag. The core should check `bit15` first.
- If `bit15` indicates an error, treat the reading as invalid (system policy: discard sample, raise fault, etc.).
- When computing temperature, use only the temperature code bits (`bit14..bit0`).

### 9.2 Worked example

Given RAM value read from `0x07`:

- Hex word: `0x3AF7`
- Decimal: `15095`

Kelvin:

- `15095 × 0.02 = 301.9 K`

Celsius:

- `301.9 − 273.15 = 28.75 °C`

---

## 10. Data Transfer to the Core (SMBus / I²C Read)

### 10.1 Bus type and address

- Interface: **SMBus (I²C-compatible)**
- Default 7-bit slave address: `0x5A`

### 10.2 Register read sequence (object temperature at `0x07`)

Conceptual SMBus/I²C transaction:

1. `START`
2. Send slave address (`0x5A`) + **Write**
3. Send command/register address `0x07`
4. `RESTART`
5. Send slave address (`0x5A`) + **Read**
6. Read **LSB**
7. Read **MSB**
8. `STOP`

Result at the core:
- The processor receives a 16-bit word assembled from two bytes:
  - First byte read: **LSB**
  - Second byte read: **MSB**

Therefore, the data reaches the core as:
- A 2-byte little-endian payload on the wire (**LSB then MSB**)
- Typically assembled into a 16-bit integer in the driver

---

## 11. End-to-End Data Flow Summary (What Reaches the Core, and in What Format)

### 11.1 What is stored in sensor RAM

- Computed temperatures are stored in MLX90614 RAM:
  - Ta at `0x06`
  - To1 at `0x07`
- Each stored value is a 16-bit word:
  - `bit15`: error flag
  - `bit14..0`: temperature code (`0.02 K/LSB`)

### 11.2 What is transferred over SMBus to the core

- The core reads two bytes from the sensor:
  - Byte0: LSB
  - Byte1: MSB
- The driver combines them into a 16-bit value:

```c
raw_word = (MSB << 8) | LSB;
```

### 11.3 What format software should store internally (recommended)

To avoid floating point in firmware, it is recommended to store temperature as either:
- milli-degree Celsius (`int32 temp_mC`), or
- centi-Kelvin / centi-degree units matching the **0.02 K** LSB

Example fixed-point approach:
- Since `1 LSB = 0.02 K = 20 mK`:

```text
Temperature(mK)  = raw_value * 20
Temperature(m°C) = raw_value * 20 - 273150
```

---

## 12. Key Points / Checklist

- The MLX90614 outputs two main computed temperatures:
  - `0x06`: Ta (ambient, sensor die temperature)
  - `0x07`: To1 (object temperature, typically read by host)
- The sensor computes temperature internally (thermopile → amplifier → 17-bit ADC → DSP).
- Computed results are stored in internal RAM registers as 16-bit words.
- Data encoding:
  - `bit15` is an error flag
  - `bit14..0` is the temperature code
  - resolution is `0.02 K` per LSB
- The core reads the data over SMBus/I²C (address `0x5A`) as **LSB then MSB**.
- The core converts `raw_value` to °C using:

```text
Temperature(°C) = raw_value × 0.02 − 273.15
```
