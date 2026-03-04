# MAX30102 SpO₂ Mode: Data Path, Internal FIFO Storage, Data Format, and Core Read Protocol

**Version:** 1.0  
**Date:** 2026-03-04

---

## 1. Purpose

This document explains, from the processor/core point of view, how **MAX30102** sends SpO₂-mode measurement data to the core:

- Which **protocol** is used to communicate with the core
- How the core knows **where to read** the measurement bytes from (register map concept)
- How samples are **stored internally** inside the sensor (FIFO)
- The exact **6-byte sample format** in SpO₂ mode
- How the core reconstructs **integer RED/IR values** from the bytes
- What the data means (raw PPG) and what is computed in software (HR/SpO₂)

> Note: This describes the **sensor → core data path** and **raw data format**. The HR/SpO₂ algorithm is executed on the host (core/MCU), not inside the MAX30102.

---

## 2. Communication Protocol

### 2.1 Protocol used
The MAX30102 communicates with the processor using **I²C** (two-wire serial bus).

- **SCL**: clock line  
- **SDA**: data line  

**Roles**
- Core / RISC‑V system: **I²C Master**
- MAX30102 sensor: **I²C Slave**

### 2.2 I²C address
The sensor has a fixed 7-bit slave address:

- 7-bit: `0b1010111`

Common 8-bit forms (as often shown in documentation):
- Write: `0xAE`
- Read:  `0xAF`

> Many I²C controllers use the **7-bit address** (`0x57`) and a separate R/W bit automatically. If your driver uses 8-bit addressing, it will use `0xAE/0xAF`.

---

## 3. How the Core Knows Where to Read Measurement Data

### 3.1 Register-based interface
MAX30102 is **register-based**. The core reads/writes registers by register address.

Key registers for measurement flow:

| Register | Address | Purpose |
|---|---:|---|
| Interrupt Status | `0x00` | Indicates events (e.g., new FIFO data) |
| FIFO Write Pointer | `0x04` | Where sensor will write next |
| FIFO Read Pointer | `0x06` | Where host will read next |
| **FIFO Data** | `0x07` | **Measurement bytes are read here** |

### 3.2 Measurement register to read
In SpO₂ mode, the measurement stream is read from:

- **`FIFO_DATA` register = `0x07`**

This register behaves like a **data port** into the FIFO:
- Each read returns the next byte from the FIFO.
- Sequential multi-byte reads pull consecutive FIFO bytes.

---

## 4. Internal Storage Inside the Sensor (FIFO)

### 4.1 FIFO memory overview
MAX30102 contains an internal **FIFO buffer**, which can hold up to **32 samples**.

### 4.2 What constitutes a “sample” in SpO₂ mode
In **SpO₂ mode**, each sample contains **two channels**:

- **RED** LED channel measurement
- **IR** LED channel measurement

Each channel measurement is derived from an **18-bit ADC** conversion (unsigned).

Therefore, per sample:

- RED = 18-bit value (sent using 3 bytes)
- IR  = 18-bit value (sent using 3 bytes)

Total per sample:

- **6 bytes/sample** = 3 bytes (RED) + 3 bytes (IR)

---

## 5. Data Format on the Wire (6 Bytes per Sample)

### 5.1 Per-channel packing: 3 bytes for an 18-bit value
Even though ADC resolution is **18 bits**, the FIFO stores and outputs each channel in **3 bytes (24 bits)**.

- Data type: **unsigned integer**
- Valid bits: **18**
- Unused bits: upper bits in the 24-bit word

### 5.2 Bit layout (conceptual)
For each channel (RED or IR):

- **Byte1** = bits `[23:16]`
- **Byte2** = bits `[15:8]`
- **Byte3** = bits `[7:0]`

But only **18 bits** are valid. Conceptually:

- Byte1: `000000 D17 D16`
- Byte2: `D15 ... D8`
- Byte3: `D7  ... D0`

So the payload is effectively a 24-bit container with:
- top 6 bits unused/zero
- lower 18 bits carrying the ADC result

### 5.3 6-byte sample ordering in SpO₂ mode
One FIFO sample is transmitted as:

| Byte index | Meaning |
|---:|---|
| 0 | RED `[23:16]` |
| 1 | RED `[15:8]` |
| 2 | RED `[7:0]` |
| 3 | IR `[23:16]` |
| 4 | IR `[15:8]` |
| 5 | IR `[7:0]` |

---

## 6. How the Core Reads the FIFO_DATA Register (I²C Transaction)

Reading a sample requires two phases:

### 6.1 Phase 1: Point to the register
Core writes the register address it wants to read from:

1. `START`
2. Slave address + **Write** (`0xAE` in 8-bit form)
3. Send register address `0x07` (FIFO_DATA)

This tells the sensor:
- “Next read should begin from FIFO_DATA register.”

### 6.2 Phase 2: Read measurement bytes
Core reads bytes sequentially:

1. `RESTART`
2. Slave address + **Read** (`0xAF` in 8-bit form)
3. Read **N bytes** (e.g., 6 bytes for one SpO₂ sample)
4. `STOP`

---

## 7. Reconstructing Integers in the Core (RED and IR)

### 7.1 Combine 3 bytes into a 24-bit word
For RED:

```c
uint32_t red24 = ( (uint32_t)byte0 << 16 ) |
                 ( (uint32_t)byte1 <<  8 ) |
                   (uint32_t)byte2;
```

For IR:

```c
uint32_t ir24  = ( (uint32_t)byte3 << 16 ) |
                 ( (uint32_t)byte4 <<  8 ) |
                   (uint32_t)byte5;
```

### 7.2 Mask to keep only 18 valid bits
Only 18 bits are valid, so mask the result:

```c
uint32_t red18 = red24 & 0x3FFFF;  // keep bits [17:0]
uint32_t ir18  = ir24  & 0x3FFFF;
```

### 7.3 Example
If FIFO provides:

```text
03 A2 F1  02 F8 AA
```

Then:

- RED bytes = `03 A2 F1` → `RED = 0x03A2F1` → `RED18 = 0x03A2F1 & 0x3FFFF`
- IR  bytes = `02 F8 AA` → `IR  = 0x02F8AA` → `IR18  = 0x02F8AA & 0x3FFFF`

---

## 8. What the Data Means (Raw PPG) vs What Software Computes (HR/SpO₂)

### 8.1 What the sensor provides
The MAX30102 does **not** send heart rate or SpO₂ directly.

It sends only:
- **raw PPG measurements**
  - RED PPG integer samples
  - IR PPG integer samples

These represent the photodiode signal intensity corresponding to light absorption/reflection through tissue.

### 8.2 What the core computes
The core/firmware algorithm computes:

- **Heart Rate (HR)**:
  - typically uses IR channel to detect pulse peaks (AC component)
- **SpO₂**:
  - uses the **ratio-of-ratios** approach based on AC/DC components of both RED and IR signals

---

## 9. How the Core Knows When Data Is Ready

The sensor provides an **INT** pin (interrupt output).

Typical behavior:
- When new FIFO data is available, **INT asserts** (often active-low).
- The core ISR (or polling loop) then reads FIFO_DATA (`0x07`) to drain available samples.

---

## 10. Summary Table (Core View)

| Item | Value |
|---|---|
| Protocol | I²C |
| Slave address (7-bit) | `0x57` (binary `1010111`) |
| 8-bit address (Write/Read) | `0xAE` / `0xAF` |
| Data source register | `FIFO_DATA = 0x07` |
| Internal storage | FIFO (up to 32 samples) |
| SpO₂ sample size | **6 bytes** |
| Channels per sample | RED + IR |
| Bytes per channel | 3 bytes |
| ADC resolution | 18-bit |
| Data type | unsigned integer |
| Host action | Read bytes → rebuild integers → run HR/SpO₂ algorithm |

---

## 11. Reference Implementation Snippet (One Sample Read)

```c
// Read 6 bytes from FIFO_DATA (0x07) and convert into RED/IR 18-bit integers.

uint8_t b[6]; // b[0..5] read from FIFO_DATA sequentially

uint32_t red24 = ((uint32_t)b[0] << 16) | ((uint32_t)b[1] << 8) | b[2];
uint32_t ir24  = ((uint32_t)b[3] << 16) | ((uint32_t)b[4] << 8) | b[5];

uint32_t red18 = red24 & 0x3FFFF;
uint32_t ir18  = ir24  & 0x3FFFF;
```

---

## 12. Notes / Integration Checklist

- Ensure the sensor is configured into **SpO₂ mode** before reading FIFO.
- Use FIFO pointers and interrupt/status flags to avoid underflow/overflow.
- Always mask to **18 bits** to avoid unused bits polluting calculations.
- Store RED/IR samples in a ring buffer for filtering and HR/SpO₂ computation.
