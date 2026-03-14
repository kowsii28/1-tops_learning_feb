# AMBA, AXI, and Bus Architecture: Complete Guide

## Table of Contents
1. [AMBA Overview](#amba-overview)
2. [AMBA Bus Protocols](#amba-bus-protocols)
3. [AXI Protocol](#axi-protocol)
4. [AXI4 vs AXI4-Lite](#axi4-vs-axi4-lite)
5. [AHB vs APB](#ahb-vs-apb)
6. [Burst Concept in AXI](#burst-concept-in-axi)
7. [Data Transfer Calculations](#data-transfer-calculations)
8. [LFSR (Linear Feedback Shift Register)](#lfsr)
9. [Homogeneous Slaves](#homogeneous-slaves)
10. [Summary Tables](#summary-tables)

---

## AMBA Overview

### What is AMBA?

**AMBA = Advanced Microcontroller Bus Architecture**

- A standard bus architecture created by **Arm** to connect components inside a System-on-Chip (SoC)
- Defines how modules communicate inside a chip
- Provides a set of bus protocols for interconnecting hardware blocks

### Components Connected by AMBA

- **CPU** (RISC-V / ARM core)
- **Memory** (RAM, Flash)
- **UART** (Serial communication)
- **GPIO** (General Purpose I/O)
- **SPI** (Serial Peripheral Interface)
- **I2C** (Inter-Integrated Circuit)
- **DMA** (Direct Memory Access)
- **Sensors**
- **High-speed peripherals**

---

## AMBA Bus Protocols

AMBA provides several protocols for different performance requirements:

| Protocol | Full Name | Purpose | Speed | Use Case |
|----------|-----------|---------|-------|----------|
| **AXI** | Advanced eXtensible Interface | High-performance advanced bus | Very High | CPU, Memory, DMA |
| **AHB** | Advanced High-performance Bus | High-performance system bus | High | System bus, Memory controllers |
| **APB** | Advanced Peripheral Bus | Simple peripheral bus | Low | Peripherals, Control registers |

---

## AXI Protocol

### What is AXI?

**AXI = Advanced eXtensible Interface**

- A high-performance bus protocol defined in AMBA
- Designed for connecting high-bandwidth components
- Latest versions: **AXI3** and **AXI4** (current standard)

### AXI Components Connected

- **CPU** cores
- **High-speed memory** (DDR)
- **DMA** controllers
- **GPU**
- **High-bandwidth peripherals**

### Key Features of AXI

| Feature | Description |
|---------|-------------|
| **Separate Channels** | Independent read and write channels |
| **Burst Transfers** | Multiple data transfers per address |
| **High Throughput** | Excellent bandwidth utilization |
| **Multiple Outstanding Transactions** | Parallel operations support |
| **Pipelined Communication** | Overlapping operations |

### AXI Channels (5 Independent Channels)

| Channel | Full Name | Purpose |
|---------|-----------|---------|
| **AW** | Write Address | Carries write address information |
| **W** | Write Data | Carries write data |
| **B** | Write Response | Write response from slave |
| **AR** | Read Address | Carries read address information |
| **R** | Read Data | Carries read data and response |

#### Visual Representation:
```
Master                              Slave
   │                                  │
   ├─── AW (Write Address) ──────────→│
   │                                  │
   ├─── W (Write Data) ──────────────→│
   │                                  │
   │←─── B (Write Response) ─────────┤
   │                                  │
   ├─── AR (Read Address) ───────────→│
   │                                  │
   │←─── R (Read Data + Response) ───┤
   │                                  │
```

---

## AXI4 vs AXI4-Lite

### AXI4

**AXI4** is the improved version of AXI (AXI3).

#### Main Improvements over AXI3:
- **Burst length increased** (16 beats → 256 beats)
- **Simplified signals**
- **Better performance**
- **AXLEN field increased** (4 bits → 8 bits)

#### AXI4 Features:
- Burst transfers up to **256 beats**
- High-bandwidth memory access
- Used for:
  - **DDR controllers**
  - **DMA engines**
  - **GPUs**
- Example use: CPU ↔ DDR Memory (Large blocks of data move quickly)

### AXI4-Lite

**AXI4-Lite** is a simplified version of AXI4.

#### Purpose:
- Control registers
- Simple peripherals
- Low-bandwidth accesses

#### Example Peripherals:
- **GPIO**
- **UART**
- **Timer**
- **RTC** (Real-Time Clock)
- **I2C controller**

#### AXI4-Lite Characteristics:
- **NO burst transfers** ⚠️
- **Single data transfer only** (1 beat maximum)
- **Simpler hardware** (smaller logic gates)
- **Lower complexity** (easier to implement)
- **Lower power consumption**

### AXI4 vs AXI4-Lite Comparison

| Feature | AXI4 | AXI4-Lite |
|---------|------|-----------|
| **Full Name** | Advanced eXtensible Interface 4 | Advanced eXtensible Interface 4 Lite |
| **Burst Transfers** | ✓ Yes | ✗ No |
| **Max Burst Length** | 256 beats | 1 transfer (single) |
| **Data Transfers** | Multiple per address | Single only |
| **AXLEN bits** | 8 bits | 8 bits (but only value 0 used) |
| **Complexity** | High | Very Low |
| **Throughput** | Very High | Low |
| **Use Case** | Memory / DMA / High-bandwidth | Control registers / Peripherals |
| **Hardware Logic** | Large | Small |
| **Pipelining** | Supported | Limited |
| **Interleaved Transactions** | Yes | No |

### Example SoC Architecture

```
              CPU (RISC-V)
                  │
            AXI Interconnect
            /              \
         AXI4            AXI4-Lite
         /                  \
    DDR RAM            Peripherals
                      ┌───┬────┬────┐
                    UART GPIO RTC  SPI

High-speed memory uses AXI4
Simple peripherals use AXI4-Lite
```

---

## AHB vs APB

### AHB (Advanced High-performance Bus)

**AHB = Advanced High-performance Bus**

#### Purpose:
- High-speed communication inside processor system
- Main system bus in many AMBA-based designs

#### Components Connected:
- **CPU** cores
- **SRAM**
- **DMA** controllers
- **High-speed peripherals**
- **Memory controllers**

#### Key Features:
- ✓ High-speed bus
- ✓ Supports burst transfers
- ✓ Pipelined operation
- ✓ Single clock edge protocol
- ✓ Multiple masters possible

### APB (Advanced Peripheral Bus)

**APB = Advanced Peripheral Bus**

#### Purpose:
- Low-speed peripheral connections
- Simple and low-power design

#### Peripherals Connected:
- **UART** (Serial communication)
- **GPIO** (General Purpose I/O)
- **Timer**
- **RTC** (Real-Time Clock)
- **I2C** controller
- **SPI** controller
- **Watchdog**

#### Key Features:
- ✓ Very simple protocol
- ✓ No burst transfers
- ✓ No pipelining
- ✓ Low power consumption
- ✓ Low hardware complexity

### How AHB and APB Work Together

Typically, there is a **bridge** connecting them:

```
        CPU
         │
       AHB BUS
         │
   AHB-APB Bridge
         │
       APB BUS
     ┌───┼───┬───┐
   UART GPIO TIMER RTC
```

#### Process Flow:
1. CPU communicates on **AHB**
2. Bridge **converts AHB → APB**
3. APB **accesses low-speed peripherals**

### AHB vs APB Comparison

| Feature | AHB | APB |
|---------|-----|-----|
| **Full Name** | Advanced High-performance Bus | Advanced Peripheral Bus |
| **Speed** | High | Low |
| **Burst Transfer** | ✓ Yes | ✗ No |
| **Pipelining** | ✓ Yes | ✗ No |
| **Complexity** | Moderate | Very Simple |
| **Use Case** | Memory / System bus | Peripherals / Control registers |
| **Power Consumption** | Higher | Lower |
| **Masters** | Multiple possible | Typically single |
| **Clock Edge** | Single edge | Simple timing |

---

## Relationship: AXI vs AHB vs APB

### Historical Evolution

```
Early designs:  CPU ── AHB ── Memory
                        │
                   AHB-APB Bridge
                        │
                   APB ── Peripherals

Modern designs: CPU ── AXI ── Memory
                        │
                   AXI-APB Bridge
                        │
                   APB ── Peripherals
```

**Note:** AXI was developed later to replace AHB in high-performance systems.

### Overall Speed Hierarchy

| Bus | Speed | Bandwidth | Use |
|-----|-------|-----------|-----|
| **AXI** | **Very High** | **Best** | CPU, DDR, DMA |
| **AHB** | High | Good | System bus, Memory controllers |
| **APB** | Low | Limited | Simple peripherals, Control registers |

---

## Burst Concept in AXI

### Understanding Bursts

#### What is a Burst?

A burst means:
- **One address phase** followed by **multiple data transfers**
- Instead of sending address for every data word, master sends address **once**
- Then **many data items** are transferred sequentially

#### Visual Example:

```
Without Burst (Multiple Transactions):
Address → 0x1000 → Data
Address → 0x1004 → Data
Address → 0x1008 → Data
Address → 0x100C → Data

With Burst (Single Transaction):
Address → 0x1000
         → Data0
         → Data1
         → Data2
         → Data3
```

### What is a Beat?

**A beat** means:
- **One data transfer** on the data bus during **one clock cycle**
- Simple formula: **1 beat = 1 data transfer**

#### Why "Beat"?

The word comes from **clock rhythm**:

```
Clock:   ↑      ↑      ↑      ↑
         │      │      │      │
Cycle:   1      2      3      4
Data:   D1     D2     D3     D4

Each clock cycle = one beat (one data transfer)
```

### AXI3 vs AXI4 Burst Lengths

| Protocol | Max Burst Length | AXLEN Bits | Formula |
|----------|------------------|-----------|---------|
| **AXI3** | 16 beats | 4 bits | AXLEN[3:0] → Number of beats = AXLEN + 1 |
| **AXI4** | 256 beats | 8 bits | AXLEN[7:0] → Number of beats = AXLEN + 1 |

#### Detailed Values:

| AXLEN Value | Number of Beats |
|-------------|-----------------|
| 0 | 1 beat |
| 1 | 2 beats |
| 2 | 3 beats |
| ... | ... |
| 15 | 16 beats |
| 255 | 256 beats (AXI4 only) |

### Why AXI4 Increased Burst Length?

#### Advantages:

| Advantage | Description |
|-----------|-------------|
| **Higher Performance** | More data transferred per transaction |
| **Better Memory Bandwidth** | Reduced address overhead |
| **Less Address Overhead** | Fewer address phases needed |
| **Efficient for Large Transfers** | DMA and large memory operations |

#### Example: Transferring 1 KB of Data

| Protocol | Beats per Burst | Bursts Needed | Total Time |
|----------|-----------------|---------------|-----------|
| **AXI3** | 16 | Many (64÷16=4) | Slower |
| **AXI4** | 256 | Few (256÷1) | Faster ✓ |

### Visual Burst Example

#### AXI3 Burst (Max 16 beats):
```
Address Phase (once):
    ARADDR = 0x1000

Data Phase (16 beats maximum):
    Beat1  → Data[31:0]
    Beat2  → Data[31:0]
    Beat3  → Data[31:0]
    ...
    Beat16 → Data[31:0]
```

#### AXI4 Burst (Max 256 beats):
```
Address Phase (once):
    ARADDR = 0x1000

Data Phase (256 beats maximum):
    Beat1   → Data[31:0]
    Beat2   → Data[31:0]
    ...
    Beat256 → Data[31:0]
```

### Real-World Timing Example (32-bit bus)

Suppose CPU wants to read 4 data words:

```
Address Phase:
    ARADDR = 0x80000000
    ARLEN  = 3  (This means 4 beats, because beats = ARLEN + 1)

Data Phase (4 clock cycles):
Clock   Beat    Data Bus (32-bit)    Memory Location
─────────────────────────────────────────────────────
 1      Beat 1  0xA3F192BC  ← 0x80000000
 2      Beat 2  0x12345678  ← 0x80000004
 3      Beat 3  0xDEADBEEF  ← 0x80000008
 4      Beat 4  0xCAFEBABE  ← 0x8000000C
```

### Train Analogy (Very Helpful)

```
Address Phase → Train station assignment
Burst        → Entire train
Beat         → One train compartment

Example:
- Train departs from Station (address phase) = 0x1000
- Compartment 1 passes (Beat 1)
- Compartment 2 passes (Beat 2)
- Compartment 3 passes (Beat 3)
- Compartment 4 passes (Beat 4)

Only ONE station announcement, but FOUR compartments pass!
```

### Important: AXI4-Lite Does NOT Support Bursts ⚠️

| Feature | AXI4 | AXI4-Lite |
|---------|------|-----------|
| **Burst Support** | ✓ Yes | ✗ **NO** |
| **Max Burst Length** | 256 beats | 1 beat (single transfer only) |
| **Multiple Data Transfers** | ✓ Yes | ✗ Single transfer only |

---

## Data Transfer Calculations

### Formula for Total Data Transfer

```
Total Data Transferred = Number of Beats × Data per Beat

Where:
- Number of Beats = AXLEN + 1
- Data per Beat = Data Bus Width / 8 (in bytes)
```

### Data Bus Widths

| Bus Width | Bytes per Beat | Example |
|-----------|----------------|---------|
| 32-bit | 4 bytes | 0xA3F192BC (4 bytes) |
| 64-bit | 8 bytes | 0xA3F192BCCAFEBABE (8 bytes) |
| 128-bit | 16 bytes | 16 bytes of data |
| 256-bit | 32 bytes | 32 bytes of data |

### Example 1: 32-bit AXI4 Bus

**Bus Width:** 32 bits = 4 bytes per beat

**Maximum Burst:**
- Maximum beats = 256
- Data per beat = 4 bytes
- **Total transferred = 256 × 4 = 1024 bytes = 1 KB**

#### Detailed Example:
```
Address Phase:
    ARADDR = 0x80000000
    ARLEN  = 255  (0xFF → 256 beats)

Data Phase (256 clock cycles):
    Beat 1   → 4 bytes from 0x80000000
    Beat 2   → 4 bytes from 0x80000004
    Beat 3   → 4 bytes from 0x80000008
    ...
    Beat 256 → 4 bytes from 0x800003FC
    
Total: 1024 bytes transferred after ONE address!
```

### Example 2: 64-bit AXI4 Bus

**Bus Width:** 64 bits = 8 bytes per beat

**Maximum Burst:**
- Maximum beats = 256
- Data per beat = 8 bytes
- **Total transferred = 256 × 8 = 2048 bytes = 2 KB**

```
Each clock cycle transfers 8 bytes instead of 4 bytes
So 2 KB can be transferred after one address
```

### Example 3: 128-bit AXI4 Bus

**Bus Width:** 128 bits = 16 bytes per beat

**Maximum Burst:**
- Maximum beats = 256
- Data per beat = 16 bytes
- **Total transferred = 256 × 16 = 4096 bytes = 4 KB**

### Example 4: 256-bit AXI4 Bus

**Bus Width:** 256 bits = 32 bytes per beat

**Maximum Burst:**
- Maximum beats = 256
- Data per beat = 32 bytes
- **Total transferred = 256 × 32 = 8192 bytes = 8 KB**

### Comparison: AXI3 vs AXI4 (32-bit bus)

| Protocol | Max Beats | Bytes per Beat | Max Total |
|----------|-----------|----------------|-----------|
| **AXI3** | 16 | 4 | 64 bytes |
| **AXI4** | 256 | 4 | 1024 bytes |
| **Increase** | 16× | Same | **16× more data** |

### Important: What Does "256 Beats" Actually Mean?

#### ❌ WRONG:
"256 beats means 256 bytes are sent"

#### ✓ CORRECT:
"256 beats means 256 data transfers, where each transfer carries data equal to the bus width"

**Key Points:**
- 256 beats ≠ 256 bytes always
- 256 beats depends on bus width
- Larger bus width = more data in same beats

### Visualization of Data Transfer

```
32-bit bus:                64-bit bus:
Address (once)             Address (once)
    │                          │
    ▼                          ▼
Beat1 → 4 bytes            Beat1 → 8 bytes
Beat2 → 4 bytes            Beat2 → 8 bytes
Beat3 → 4 bytes            Beat3 → 8 bytes
...                        ...
Beat256 → 4 bytes          Beat256 → 8 bytes
    │                          │
Total: 1 KB                Total: 2 KB
```

---

## LFSR (Linear Feedback Shift Register)

### What is LFSR?

**LFSR = Linear Feedback Shift Register**

A digital circuit that:
- Generates **pseudo-random numbers**
- Uses simple hardware (flip-flops and XOR gates)
- Operates deterministically (predictable sequence)
- Widely used in hardware design and testing

### LFSR Basic Concept

#### How it Works:
1. **Bits shift** through registers (left or right)
2. **New bit** is calculated from **XOR** of selected bits (taps)
3. **New bit** is inserted into register
4. **Sequence repeats** → produces pseudo-random pattern

#### Why "Linear"?
- Uses **linear operations** (XOR only)
- XOR in binary is a **linear operation** over GF(2) (Galois Field)

### 4-bit LFSR Example

```
Register: [ Q3  Q2  Q1  Q0 ]

Feedback: new_bit = Q3 XOR Q2

Clock Cycle    Register State
    0              1001
    1              1100
    2              1110
    3              1111
    4              0111
    ...           ...continues
```

### Hardware Structure

```
 +----+    +----+    +----+    +----+
 |FF0 | -> |FF1 | -> |FF2 | -> |FF3 |
 +----+    +----+    +----+    +----+
     ^                         |
     |________ XOR ____________|
     
Feedback taps generate new input bit
```

### LFSR Applications

| Application | Purpose |
|-------------|---------|
| **Random Number Generation** | Generate pseudo-random sequences |
| **Built-in Self Test (BIST)** | Test hardware functionality |
| **CRC/Checksum** | Error detection |
| **Encryption/Scrambling** | Data security |
| **Test Pattern Generation** | Hardware verification |
| **Memory Testing** | Test memory arrays |

### LFSR in Your SoC Context

#### AXI4-attached LFSR Example:

```
"AXI4-attached LFSR; read returns pseudo-random data, 
 writes are compressed into a checksum"
```

**Meaning:**

| Operation | What Happens |
|-----------|--------------|
| **Read** | Returns LFSR pseudo-random value from current state |
| **Write** | Compresses written data using LFSR polynomial → updates state |

#### Use Cases:
- **Test peripheral** in SoC
- **Random data generator** for verification
- **Signature generator** for design testing
- **CPU verification** via AXI4 bus

### Simple Verilog LFSR Example

```verilog
module lfsr (
    input clk,
    input reset,
    output reg [7:0] lfsr
);

wire feedback;

// Feedback polynomial: tap from bits 7, 5, 4, 3
assign feedback = lfsr[7] ^ lfsr[5] ^ lfsr[4] ^ lfsr[3];

always @(posedge clk or posedge reset) begin
    if (reset)
        lfsr <= 8'h1;  // Initial value
    else
        lfsr <= {lfsr[6:0], feedback};  // Shift left, insert feedback
end

endmodule
```

---

## Homogeneous Slaves

### What are Slaves in Bus Architecture?

In bus protocols like AXI4:

| Component | Role | Examples |
|-----------|------|----------|
| **Master** | Initiates transactions | CPU, DMA controller |
| **Slave** | Responds to requests | RAM, UART, GPIO, RTC |

### What Does "Homogeneous" Mean?

**Homogeneous** = Same type / Uniform structure

**Homogeneous slaves** means:
- All slave devices on the bus have **same interface type**
- All slaves use **same protocol**
- All slaves have **same signal structure**
- All slaves follow **same transaction rules**

### Homogeneous Slave System

```
        RISC-V CPU (Master)
            │
        AXI Interconnect
       /      │       \
     RAM     UART     GPIO
    (AXI)    (AXI)    (AXI)

All slaves speak AXI → Homogeneous System
```

### Heterogeneous Slave System (Opposite)

```
        CPU (Master)
         │
   AXI Interconnect
     |         |
   AXI RAM   APB Bridge
   (AXI)        |
             UART (APB)
             GPIO (APB)

Different protocols → Heterogeneous System
```

### Advantages of Homogeneous Slaves

| Advantage | Benefit |
|-----------|---------|
| **Simpler Bus Design** | No complex logic needed |
| **No Protocol Converters** | Reduced area and power |
| **Higher Performance** | No conversion overhead |
| **Easier Verification** | Consistent interfaces |
| **Faster Development** | Reusable IP blocks |

### Example in Your RISC-V SoC

```
Homogeneous AXI4-Lite Slave Peripherals:

├── RTC (Real-Time Clock)     [AXI4-Lite Slave]
├── I2C Controller            [AXI4-Lite Slave]
├── SPI Controller            [AXI4-Lite Slave]
├── GPIO                      [AXI4-Lite Slave]
└── UART                      [AXI4-Lite Slave]

All use same AXI4-Lite interface signals
```

---

## Summary Tables

### Quick Reference: All Bus Protocols

| Feature | AMBA | AXI | AXI4 | AXI4-Lite | AHB | APB |
|---------|------|-----|------|-----------|-----|-----|
| **Type** | Architecture | Protocol | Full Protocol | Lite Protocol | Protocol | Protocol |
| **Speed** | N/A | Very High | Very High | Low | High | Low |
| **Bursts** | N/A | Yes | 256 beats | NO | Yes | No |
| **Use** | Standard | High-bandwidth | CPU/Memory | Peripherals | System Bus | Simple Peripherals |
| **Complexity** | N/A | High | High | Very Low | Moderate | Very Simple |
| **Power** | N/A | Higher | Higher | Lower | Higher | Lowest |

### Bandwidth Comparison

| Protocol | Max Bandwidth (32-bit) | Max Bandwidth (64-bit) |
|----------|------------------------|-----------------------|
| **AXI4** | 256 × 4 = 1 KB/transaction | 256 × 8 = 2 KB/transaction |
| **AHB** | 16 × 4 = 64 bytes/transaction | N/A |
| **APB** | 1 × 4 = 4 bytes/transaction | N/A |

### SoC Hierarchy Reference

```
┌─────────────────────────────────────┐
│      CPU (Master)                   │
│      RISC-V / ARM Core              │
└──────────────┬──────────────────────┘
               │
         ┌─────▼─────┐
         │ AXI / AHB │  High-speed system bus
         └─────┬─────┘
        ┌──────┴──────┐
        │             │
    ┌───▼──┐    ┌─────▼──────┐
    │ DDR  │    │ AXI-APB    │
    │Memory│    │ Bridge     │
    └──────┘    └─────┬──────┘
                      │
                  ┌───▼───┐
                  │  APB  │  Low-speed peripheral bus
                  └───┬───┘
            ┌─────────┼─────────┐
            │         │         │
        ┌───▼──┐ ┌───▼──┐ ┌───▼──┐
        │UART  │ │GPIO  │ │ RTC  │
        └──────┘ └──────┘ └──────┘
```

### Feature Comparison Matrix

| Aspect | AXI4 | AXI4-Lite | AHB | APB |
|--------|------|-----------|-----|-----|
| **Intended Use** | Memory / DMA / High-BW | Control Registers | System Bus | Peripherals |
| **Burst Support** | ✓ Full | ✗ None | ✓ Full | ✗ None |
| **Max Burst Length** | 256 | 1 | 16 | 1 |
| **Pipelining** | ✓ Yes | Limited | ✓ Yes | ✗ No |
| **Outstanding Tx** | Multiple | Single | Multiple | Single |
| **Hardware Complexity** | High | Low | Moderate | Very Low |
| **Typical Data Width** | 64-256 bit | 32 bit | 32-64 bit | 32 bit |
| **Frequency** | High (>100 MHz) | Medium (50-100 MHz) | High (>100 MHz) | Low (<100 MHz) |

---

## Key Takeaways for RISC-V SoC Design

### 1. Architecture Selection
```
For High Bandwidth (Memory/DMA):
    → Use AXI4

For Low Bandwidth (Peripherals):
    → Use AXI4-Lite or APB

For System Bus:
    → Use AXI or AHB
```

### 2. Burst Understanding
```
Remember: 1 beat = 1 clock cycle = 1 data transfer

For AXI4: Up to 256 beats = Up to (256 × bus_width) bytes

For AXI4-Lite: 1 beat only = Single transfer
```

### 3. Data Transfer Calculation
```
Total Data = Number of Beats × (Bus Width / 8)

Example (32-bit, 256 beats):
Total Data = 256 × 4 = 1024 bytes = 1 KB
```

### 4. Protocol Selection for Peripherals
```
Simple Peripherals (UART, GPIO, RTC):
    ├── Use APB (simplest, lowest power)
    └── OR use AXI4-Lite (if need AXI ecosystem)

High-Speed Interfaces (DDR, High-speed Sensors):
    └── Use AXI4 (full burst support)
```

---

## Visual Architecture Example

```
                    RISC-V CPU
                    (Master)
                        │
                        │ 64-bit
                  ┌─────▼─────┐
                  │ AXI Switch │
                  │(Interconn)│
                  └────┬─────┬┘
                       │     │
                   64-bit  32-bit
                       │     │
        ┌──────────────┘     └──────────────┐
        │                                   │
    ┌───▼────┐                        ┌─────▼─────┐
    │ AXI4   │                        │AXI-APB    │
    │ DDR    │                        │ Bridge    │
    │ Ctrl   │                        └─────┬─────┘
    │(Slave) │                              │
    └────────┘                          32-bit APB
                                            │
                        ┌───────────────────┼───────────────┐
                        │                   │               │
                    ┌───▼──┐            ┌──▼───┐        ┌──▼────┐
                    │UART  │            │GPIO  │        │ RTC   │
                    │(APB) │            │(APB) │        │(APB)  │
                    └──────┘            └──────┘        └───────┘

High-speed path: CPU → DDR (AXI4, 64-bit, burst support)
Low-speed path:  CPU → Peripherals (APB, 32-bit, no burst)
```

---

## Reference Signal Names

### AXI Write Signals
- `AWVALID` / `AWREADY` - Write address valid/ready
- `AWADDR` - Write address (typically 32-64 bit)
- `AWLEN` - Write burst length (8 bits in AXI4, 4 bits in AXI3)
- `AWSIZE` - Write burst size (bytes per beat)
- `WVALID` / `WREADY` - Write data valid/ready
- `WDATA` - Write data
- `WLAST` - Last write data transfer in burst
- `BRESP` - Write response

### AXI Read Signals
- `ARVALID` / `ARREADY` - Read address valid/ready
- `ARADDR` - Read address
- `ARLEN` - Read burst length
- `ARSIZE` - Read burst size
- `RVALID` / `RREADY` - Read data valid/ready
- `RDATA` - Read data
- `RLAST` - Last read data transfer in burst
- `RRESP` - Read response

---

## Additional Resources

### For More Information:
- ARM AMBA Specification documents
- AXI Protocol specification sheets
- Your SoC design documentation
- AXI Interconnect IP documentation

### Testing LFSR:
- Use in verification testbenches
- Implement CRC checkers
- Build memory BIST structures

---

**Last Updated:** 2026-03-14  
**Author Notes:** This guide covers AMBA architecture, AXI protocols, burst concepts, and related SoC design elements for RISC-V implementation.
