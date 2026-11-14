# sunfree-433mhz-blind-protocol
Reverse-engineered 433.92 MHz 2-FSK rolling-code protocol (SRJD) for Sunfree &amp; Allesin M130 motorized roller blinds. Includes Ghidra disassembly findings, XXTEA cipher &amp; Maxim CRC-8 specs, CMT2119A TWI init sequence, and an ESP32 baseband transceiver.

# Sunfree / Allesin M130 433.92 MHz Roller Blind Protocol (`SRJD`) — Reverse Engineering & ESP32 Transceiver

Complete over-the-air protocol specification, Ghidra firmware disassembly findings, XXTEA cryptographic reconstruction, and bare-metal ESP32 baseband transmitter implementation for **Sunfree Motors** (`SRJD` protocol) `433.92 MHz` tubular roller blind motors and handheld remotes.

> **Compatible Hardware & Search Aliases:** Sunfree Motors RF remotes & tubular motors, **Allesin M130** motorized roller blinds, "Tuya-compliant" 433.92 MHz 2-FSK smart shades/blinds (sold with external Tuya Wi-Fi-to-RF hubs), **HC32F002** + **CMT2119A** remote PCBs, `SRJD` 433 MHz rolling-code protocol.

---

## Table of Contents
1. [Overview & Why Off-the-Shelf RF Bridges Fail](#1-overview--why-off-the-shelf-rf-bridges-fail)
2. [Hardware Teardown, SWD Dump & Ghidra Findings](#2-hardware-teardown-swd-dump--ghidra-findings)
3. [Hardware Bus Hijacking & Wiring](#3-hardware-bus-hijacking--wiring)
4. [CMT2119A 2-Wire Interface (TWI) Initialization](#4-cmt2119a-2-wire-interface-twi-initialization)
5. [Physical Air Interface & Burst Structure](#5-physical-air-interface--burst-structure)
6. [Wire Frame, XXTEA Encryption & Maxim CRC-8 Specification](#6-wire-frame-xxtea-encryption--maxim-crc-8-specification)
7. [Command Opcodes & 3-Step Pairing Sequence](#7-command-opcodes--3-step-pairing-sequence)
8. [Minimal Standalone ESP32 Implementation](#8-minimal-standalone-esp32-implementation)

---

## 1. Overview & Why Off-the-Shelf RF Bridges Fail

Many motorized roller blinds (such as **Sunfree** and **Allesin M130**) are marketed as *"Tuya-compatible"*, which in practice requires buying the manufacturer's proprietary cloud-tethered RF gateway. Attempts to control these motors locally using standard universal RF bridges fail for three reasons:

1. **2-FSK Modulation (Not OOK):** Popular hubs like the **Broadlink RM4 Pro** only demodulate OOK/ASK signals and cannot receive or transmit Sunfree's `433.92 MHz` **2-FSK** carrier.
2. **16-Bit Monotonic Rolling Counter:** FSK-capable bridges (such as **Bond Bridge**) and raw SDR IQ capture/replay attacks (HackRF / RTL-SDR) fail because each paired transmitter ID has a strictly monotonic 16-bit counter tracked by the motor—replayed or stale counters are silently ignored.
3. **XXTEA Block Encryption:** Probing the baseband trace between the remote's microcontroller and RF transmitter shows high-entropy encrypted frames rather than static DIP-switch codes.

By dumping the remote's unprotected flash memory via SWD and reversing the firmware in **Ghidra**, the entire TWI radio configuration, XXTEA block cipher, Maxim CRC-8 checksum, and multi-burst air protocol were reconstructed.

---

## 2. Hardware Teardown, SWD Dump & Ghidra Findings

### Remote PCB Bill of Materials
* **Microcontroller:** HDSC (Huada Semiconductor) **HC32F002** (ARM Cortex-M0+, `3.3 V`).
* **RF Transmitter:** CMOSTEK **CMT2119A** sub-1 GHz 2-FSK/OOK transmitter (`240–960 MHz`, `< 20 nA` sleep current).

### Extracting the Firmware & Key via SWD + Ghidra
1. **Unprotected Flash:** Factory SWD readout protection (`SWD` lock) was left **disabled** on the HC32F002, allowing a full binary flash dump using standard SWD pads on the PCB (`SWDIO`, `SWCLK`, `NRST`, `GND`).
2. **Ghidra Disassembly:** Loading the raw ARM Cortex-M0+ binary into **Ghidra** reveals the radio initialization routine (which writes a 111-word configuration table to the CMT2119A over 2-wire TWI) and the packet-assembly function (`FUN_000019c8` in the analyzed revision).
3. **128-Bit XXTEA Key Location (`0x00000866`):** In the HC32F002 firmware dump, the 16-byte (`128-bit`) XXTEA key sits in flash at file/memory offset **`0x00000866`** (referenced as 4 little-endian `uint32_t` words by the encryption loop).
   * *Note: The key bytes are intentionally redacted (`0x00000000`) in this repository. If you own a Sunfree/Allesin remote, you can dump your own HC32F002 over SWD and read the 16 bytes at `0x00000866`.*

---

## 3. Hardware Bus Hijacking & Wiring

### Why Bus Hijacking Beats Button Simulation
The remote's front and back buttons are wired in a multiplexed scan matrix, and simulating button presses limits you to a single hardcoded transmitter ID and counter. Instead, pulling the HC32F002's active-low reset pin (`NRST`) **`LOW`** tri-states all stock MCU I/O pins, letting an ESP32 drive the CMT2119A's 2-wire interface (`SDIO/DATA` and `SCK/CLK`) directly and emulate an unlimited number of independent virtual remotes.

### Wiring (ESP32 ↔ Remote PCB)

| ESP32 Pin | Remote PCB Signal | Direction | Notes |
| :--- | :--- | :--- | :--- |
| `GPIO 18` | `CMT2119A SDIO / DATA` | Output | TWI serial data during init; NRZ baseband during TX |
| `GPIO 19` | `CMT2119A SCK / CLK` | Output | TWI shift clock (`falling-edge` latch); held `HIGH` during TX |
| `GPIO 13` | `HC32F002 NRST` | Output (Active Low) | Pull `LOW` to hold stock MCU in reset and tri-state its pins |
| `3V3` | `VDD (3.3 V)` | Power | Remove the remote's coin cell before powering from ESP32 |
| `GND` | `GND` | Ground | Common ground reference |

> **Critical Hardware Precaution:** Always drive `SDIO/DATA = 0` and `SCK/CLK = 1` on the ESP32 **before** pulling `HC32_RST` low. The CMT2119A has an internal pull-up on `DATA`; if `DATA` floats `0 -> 1` while `CLK` is idle, the chip interprets it as a direct-mode TX trigger and locks the RF power amplifier continuously on.

---

## 4. CMT2119A 2-Wire Interface (TWI) Initialization

Before transmitting baseband data, the CMT2119A must be reset, loaded with its 111-word indirect RAM configuration table (`433.92 MHz` PLL `N_INT`/`N_FRAC` and 2-FSK deviation), and switched from TWI mode into direct `DATA`-pin TX mode:

1. **`TWI_RST` (48-Clock Continuous Train):**
   * Hold `DATA = 0` and toggle `SCK` for **32 clock cycles**.
   * Immediately shift out the 16-bit word `0x8D00` MSB-first on `SCK` falling edges (`48` clocks total).
2. **`SOFT_RST`:**
   * Write register `0x3D = 0x01` (16-bit TWI frame `0xBD01`), then wait `>= 1 ms` (`2 ms` used).
3. **111-Word Register & Indirect RAM Load:**
   * Direct register writes use the 16-bit format `(0x80 | (reg & 0x3F)) << 8 | val`, latched on the **falling edge** of `SCK` (`~0.9 us` high / `~0.9 us` low).
   * Indirect configuration RAM slots `0x00..0x14` are programmed by writing address to `0x18`, low byte to `0x19`, high byte to `0x1A`, and strobing `0x25 = 0x01`.
4. **`TWI_OFF` (`0x8D02`):**
   * Write register `0x0D = 0x02` (`0x8D02`) to disable TWI and arm rising-edge direct-mode transmission on `DATA`, followed by a `54 ms` settle delay (`SCK = 1`, `DATA = 0`).
5. **Return to Deep Sleep (`t_STOP`):**
   * After transmission finishes, holding `DATA = 0` with `SCK = 1` for `> 20 ms` (`t_STOP` configured to `20 ms`) returns the CMT2119A to `< 20 nA` sleep.

---

## 5. Physical Air Interface & Burst Structure

* **Center Frequency:** `433.92 MHz`
* **Modulation:** `2-FSK` (NRZ baseband on `DATA`, MSB-first)
* **Bit Rate / Bit Period:** `25.115 us/bit` (`~39.817 kbps`, `6,028` cycles at `240 MHz`, matching the HC32F002 internal RC oscillator)

Every button press transmits **4 bursts** separated by `30 ms` (`> t_STOP = 20 ms`) low gaps:

| Burst | Preamble | Sync + Payload | Low Gap After Burst | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Burst 1 (Wake)** | `808 x 0xAA` (`6,464` bits / `~162.3 ms`) | `21-byte` frame (`168` bits / `~4.22 ms`) | `30 ms` (`DATA = 0`) | Wakes duty-cycled motor receiver & locks AGC |
| **Burst 2 (Repeat 1)** | `8 x 0xAA` (`64` bits / `~1.61 ms`) | `21-byte` frame (`168` bits / `~4.22 ms`) | `30 ms` (`DATA = 0`) | Redundant frame delivery |
| **Burst 3 (Repeat 2)** | `8 x 0xAA` (`64` bits / `~1.61 ms`) | `21-byte` frame (`168` bits / `~4.22 ms`) | `30 ms` (`DATA = 0`) | Redundant frame delivery |
| **Burst 4 (Repeat 3)** | `8 x 0xAA` (`64` bits / `~1.61 ms`) | `21-byte` frame (`168` bits / `~4.22 ms`) | `25 ms` (`DATA = 0`) | Final repeat + `t_STOP` sleep entry |

---

## 6. Wire Frame, XXTEA Encryption & Maxim CRC-8 Specification

### 21-Byte Wire Frame Layout

| Wire Offset | Size | Field | Value / Encoding |
| :--- | :--- | :--- | :--- |
| `0..3` | `4 B` | Magic Header | ASCII `"SRJD"` (`0x53 0x52 0x4A 0x44`) |
| `4` | `1 B` | Payload Length | `0x10` (`16` bytes) |
| `5..20` | `16 B` | Payload | `payload[0..15]` (see below) |

### 16-Byte Payload Structure (`payload[0..15]`)

| Payload Offset | Size | Field | Endianness | Encrypted? | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0` | `1 B` | Protocol Type | — | **Yes (XXTEA)** | Constant `0x05` |
| `1..2` | `2 B` | Rolling Counter | **Little-Endian** | **Yes (XXTEA)** | 16-bit monotonic counter (`1..65535`) |
| `3..6` | `4 B` | Transmitter ID | **Big-Endian** | **Yes (XXTEA)** | Unique 32-bit remote/channel ID |
| `7..10` | `4 B` | Padding | — | **Yes (XXTEA)** | Constant `0x00 0x00 0x00 0x00` |
| `11..12` | `2 B` | Channel Bitmask | **Little-Endian** | **No (Cleartext)** | `0x01 0x00` = Channel 1 |
| `13` | `1 B` | Reserved | — | **No (Cleartext)** | Constant `0x00` |
| `14` | `1 B` | Command Opcode | — | **No (Cleartext)** | Button / calibration opcode (see Section 7) |
| `15` | `1 B` | Checksum | — | **No (Cleartext)** | Maxim CRC-8 over `payload[0..14]` post-encryption |

### Cryptographic & Integrity Algorithms (Reconstructed via Ghidra)

1. **XXTEA (Corrected Block TEA) over `payload[0..11]`:**
   * The first `12` bytes (`payload[0..11]`) are treated as `n = 3` unsigned 32-bit little-endian words (`v[0]`, `v[1]`, `v[2]`).
   * Encrypted in place using standard XXTEA with `rounds = 1 + 52 / 3 = 18`, delta `0x9E3779B9`, and the 128-bit key located at offset **`0x00000866`** in the HC32F002 firmware (`uint32_t key[4]`).
2. **Maxim CRC-8 (`CRC-8/MAXIM`) over `payload[0..14]`:**
   * Computed **after** XXTEA encryption across the `12` encrypted bytes (`payload[0..11]`) plus the `3` cleartext bytes (`payload[12..14]`), and written to `payload[15]`.
   * Parameters: Reflected polynomial `0x8C` (normal `0x31`), initial value `0x00`, final XOR `0x00`.

```c
// XXTEA block cipher (n = 3 words = 12 bytes, 18 rounds)
void xxtea_encrypt(uint32_t *v, int n, uint32_t const key[4]) {
    uint32_t y, z, sum = 0;
    uint32_t p, rounds = 1 + 52 / n, e;
    z = v[n - 1];
    do {
        sum += 0x9E3779B9;
        e = (sum >> 2) & 3;
        for (p = 0; p < (uint32_t)(n - 1); p++) {
            y = v[p + 1];
            z = v[p] += (((z >> 5 ^ y << 2) + (y >> 3 ^ z << 4)) ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z)));
        }
        y = v[0];
        z = v[n - 1] += (((z >> 5 ^ y << 2) + (y >> 3 ^ z << 4)) ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z)));
    } while (--rounds);
}

// Maxim 1-Wire CRC-8 (reflected poly 0x8C, init 0x00)
uint8_t crc8_maxim(const uint8_t *data, uint8_t len) {
    uint8_t crc = 0;
    for (uint8_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t j = 0; j < 8; j++) {
            crc = (crc & 1) ? ((crc >> 1) ^ 0x8C) : (crc >> 1);
        }
    }
    return crc;
}
```

### Security Observations
* **Unauthenticated Cleartext Opcode:** Because XXTEA only encrypts `payload[0..11]` while `payload[14]` (opcode) sits in the clear and `payload[15]` is an unkeyed CRC-8, a captured `UP` (`0x01`) packet can be modified to `DOWN` (`0x03`) with a recalculated CRC-8 and replayed before the counter increments—even without the XXTEA key.
* **Disabled SWD Readout Protection:** Leaving SWD unlocked on the HC32F002 exposes the hardcoded 128-bit XXTEA key at `0x00000866`.

---

## 7. Command Opcodes & 3-Step Pairing Sequence

### Opcode Table (`payload[14]`)

| Opcode | Button / Function | Behavior |
| :--- | :--- | :--- |
| `0x01` | **UP** (Normal Mode) | Continuous open until upper limit or `STOP` |
| `0x02` | **STOP** (Normal Mode) | Halt motor movement immediately |
| `0x03` | **DOWN** (Normal Mode) | Continuous close until lower limit or `STOP` |
| `0x41` | **Calibration Step UP** | `~500 ms` upward step during limit setup |
| `0x42` | **Confirm / Set Limit** | Saves current upper/lower end-limit position |
| `0x43` | **Calibration Step DOWN** | `~500 ms` downward step during limit setup |
| `0x4E` | **Back Arrow UP** | Micro-jog upward for fine limit trimming |
| `0x4F` | **Back Arrow DOWN** | Micro-jog downward for fine limit trimming |
| `0x4D` | **Back `S` (Stage 0)** | Initial `S` / Prog button press (`t = 0.0 s`) |
| `0x5D` | **Back `S` (Stage 1)** | Held `S` / Prog button stage 1 (`t + 1.3 s`) |
| `0x6D` | **Back `S` (Stage 2)** | Held `S` / Prog button stage 2 (`t + 2.8 s`) |

### 3-Step Pairing Sequence
To pair a new 32-bit transmitter ID to a motor:
1. Put the blind motor into learn/pairing mode (hold the physical button on the motor head until it jogs/beeps).
2. Transmit **4 bursts** with opcode `0x4D` at `counter = N`.
3. Wait **`~1.3 s`** start-to-start (`~1,025 ms` after Burst 4 completes) and transmit **4 bursts** with opcode `0x5D` at `counter = N + 1`.
4. Wait **`~1.5 s`** start-to-start (`~1,225 ms` after Burst 4 completes) and transmit **4 bursts** with opcode `0x6D` at `counter = N + 2`.

---

## 8. Minimal Standalone ESP32 Implementation

This self-contained Arduino/ESP-IDF sketch demonstrates cycle-accurate (`240 MHz`) CMT2119A TWI initialization, XXTEA encryption, Maxim CRC-8 calculation, and `4`-burst `433.92 MHz` baseband transmission.

```cpp
#include <Arduino.h>
#include "driver/gpio.h"
#include "soc/gpio_reg.h"
#include "soc/rtc_wdt.h"
#include "soc/timer_group_reg.h"

// --- PIN DEFINITIONS ---
#define PIN_SDIO      18  // CMT2119A DATA / SDIO
#define PIN_SCK       19  // CMT2119A CLK / SCK
#define PIN_HC32_RST  13  // HC32F002 Active-Low NRST

// --- COMMAND OPCODES ---
#define CMD_UP        0x01
#define CMD_STOP      0x02
#define CMD_DOWN      0x03
#define CMD_PAIR_S0   0x4D
#define CMD_PAIR_S1   0x5D
#define CMD_PAIR_S2   0x6D

// REDACTED: Populate with the 16-byte little-endian XXTEA key located at 0x00000866
// in the HC32F002 firmware dump.
const uint32_t XXTEA_KEY[4] = {0x00000000, 0x00000000, 0x00000000, 0x00000000};

// Example virtual transmitter ID and monotonic rolling counter
uint32_t transmitterId  = 0x12488DF9;
uint16_t rollingCounter = 100;

struct TwiCmd {
    uint8_t reg;
    uint8_t val;
};

// Exact 111-word CMT2119A register & indirect RAM configuration table for 433.92 MHz 2-FSK
const TwiCmd CMT_TWI_INIT[] = {
    {0x02, 0x78}, {0x2F, 0x80}, {0x35, 0xCA}, {0x36, 0xEB},
    {0x37, 0x37}, {0x38, 0x82}, {0x12, 0x10}, {0x12, 0x00},
    {0x24, 0x07}, {0x1D, 0x20},
    // Indirect RAM slots 0x00..0x14 (0x18=Addr, 0x19=Low, 0x1A=High, 0x25=0x01 Strobe)
    {0x18, 0x00}, {0x19, 0x7F}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x01}, {0x19, 0x00}, {0x1A, 0x54}, {0x25, 0x01},
    {0x18, 0x02}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x03}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x04}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x05}, {0x19, 0x00}, {0x1A, 0xF0}, {0x25, 0x01},
    {0x18, 0x06}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x07}, {0x19, 0xC5}, {0x1A, 0xC1}, {0x25, 0x01},
    {0x18, 0x08}, {0x19, 0x08}, {0x1A, 0x42}, {0x25, 0x01},
    {0x18, 0x09}, {0x19, 0xC9}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x0A}, {0x19, 0x01}, {0x1A, 0x24}, {0x25, 0x01},
    {0x18, 0x0B}, {0x19, 0x81}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x0C}, {0x19, 0x00}, {0x1A, 0x80}, {0x25, 0x01},
    {0x18, 0x0D}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x0E}, {0x19, 0xFF}, {0x1A, 0xFF}, {0x25, 0x01},
    {0x18, 0x0F}, {0x19, 0x20}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x10}, {0x19, 0x8B}, {0x1A, 0x4F}, {0x25, 0x01}, // 433.92 MHz PLL N_INT
    {0x18, 0x11}, {0x19, 0xD6}, {0x1A, 0xA2}, {0x25, 0x01}, // 433.92 MHz PLL N_FRAC
    {0x18, 0x12}, {0x19, 0x13}, {0x1A, 0x0E}, {0x25, 0x01},
    {0x18, 0x13}, {0x19, 0x19}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x14}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    // Post-calibration indirect updates
    {0x18, 0x07}, {0x19, 0xC5}, {0x1A, 0xC1}, {0x25, 0x01},
    {0x18, 0x08}, {0x19, 0x08}, {0x1A, 0x42}, {0x25, 0x01},
    {0x18, 0x0D}, {0x19, 0x00}, {0x1A, 0x00}, {0x25, 0x01},
    {0x18, 0x10}, {0x19, 0xC9}, {0x1A, 0x5F}, {0x25, 0x01},
    // TWI_OFF (0x8D02): Disables TWI & arms DATA pin rising-edge TX trigger
    {0x0D, 0x02}
};

// =====================================================================================
// 1. CRYPTOGRAPHY & FRAME ASSEMBLY
// =====================================================================================
void xxtea_encrypt(uint32_t *v, int n, uint32_t const key[4]) {
    uint32_t y, z, sum = 0;
    uint32_t p, rounds = 1 + 52 / n, e; // 18 rounds for n = 3 words
    z = v[n - 1];
    do {
        sum += 0x9E3779B9;
        e = (sum >> 2) & 3;
        for (p = 0; p < (uint32_t)(n - 1); p++) {
            y = v[p + 1];
            z = v[p] += (((z >> 5 ^ y << 2) + (y >> 3 ^ z << 4)) ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z)));
        }
        y = v[0];
        z = v[n - 1] += (((z >> 5 ^ y << 2) + (y >> 3 ^ z << 4)) ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z)));
    } while (--rounds);
}

uint8_t crc8_maxim(const uint8_t *data, uint8_t len) {
    uint8_t crc = 0;
    for (uint8_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t j = 0; j < 8; j++) {
            crc = (crc & 1) ? ((crc >> 1) ^ 0x8C) : (crc >> 1);
        }
    }
    return crc;
}

void build_sunfree_frame(uint8_t opcode, uint32_t txId, uint16_t counter, uint8_t frame21[21]) {
    alignas(4) uint8_t payload[16] = {0};

    // Encrypted block (bytes 0..11): Header (0x05), LE Counter, BE Transmitter ID, 4B zero pad
    payload[0] = 0x05;
    payload[1] = counter & 0xFF;
    payload[2] = (counter >> 8) & 0xFF;
    payload[3] = (txId >> 24) & 0xFF;
    payload[4] = (txId >> 16) & 0xFF;
    payload[5] = (txId >> 8)  & 0xFF;
    payload[6] = txId & 0xFF;

    // Cleartext block (bytes 11..14): Channel 1 bitmask (0x0001) + Command Opcode
    payload[11] = 0x01;
    payload[12] = 0x00;
    payload[14] = opcode;

    // Encrypt first 12 bytes (3 x uint32_t) in-place using 4-byte aligned buffer
    uint32_t v[3];
    memcpy(v, payload, 12);
    xxtea_encrypt(v, 3, XXTEA_KEY);
    memcpy(payload, v, 12);

    // Compute Maxim CRC-8 over encrypted bytes [0..11] + cleartext bytes [12..14]
    payload[15] = crc8_maxim(payload, 15);

    // Prepend 5-byte "SRJD" + 0x10 wire header
    frame21[0] = 'S';
    frame21[1] = 'R';
    frame21[2] = 'J';
    frame21[3] = 'D';
    frame21[4] = 0x10;
    memcpy(&frame21[5], payload, 16);
}

// =====================================================================================
// 2. CYCLE-ACCURATE CMT2119A TWI & BASEBAND DRIVER (240 MHz)
// =====================================================================================
static inline uint32_t IRAM_ATTR getCycleCount() {
    uint32_t ccount;
    __asm__ __volatile__("rsr %0, ccount" : "=a"(ccount));
    return ccount;
}

static inline void IRAM_ATTR delayCycles(uint32_t cycles) {
    uint32_t start = getCycleCount();
    while ((uint32_t)(getCycleCount() - start) < cycles) {}
}

static inline void IRAM_ATTR fastPinWrite(int gpio, uint8_t level) {
    if (level) REG_WRITE(GPIO_OUT_W1TS_REG, (1UL << gpio));
    else       REG_WRITE(GPIO_OUT_W1TC_REG, (1UL << gpio));
}

static inline void IRAM_ATTR feedWatchdogs() {
    REG_WRITE(TIMG_WDTWPROTECT_REG(0), TIMG_WDT_WKEY_VALUE);
    REG_WRITE(TIMG_WDTFEED_REG(0), 1);
    REG_WRITE(TIMG_WDTWPROTECT_REG(0), 0);
    rtc_wdt_feed();
}

void IRAM_ATTR cmt_twi_shift16(uint16_t word16) {
    for (int i = 15; i >= 0; i--) {
        fastPinWrite(PIN_SDIO, (word16 >> i) & 1);
        delayCycles(216);              // 0.9 us setup
        fastPinWrite(PIN_SCK, 0);      // Falling edge latches bit
        delayCycles(216);              // 0.9 us hold
        fastPinWrite(PIN_SCK, 1);      // Rising edge
    }
    delayCycles(144);
    fastPinWrite(PIN_SDIO, 0);
    delayCycles(144);
}

void IRAM_ATTR cmt_twi_write_reg(uint8_t reg, uint8_t val) {
    uint16_t word16 = ((uint16_t)(0x80 | (reg & 0x3F)) << 8) | val;
    fastPinWrite(PIN_SCK, 1);
    fastPinWrite(PIN_SDIO, 0);
    delayCycles(144);
    cmt_twi_shift16(word16);
}

void IRAM_ATTR cmt_twi_init_433mhz() {
    // 48-clock TWI_RST (32 clocks with DATA=0 + 16 clocks of 0x8D00)
    fastPinWrite(PIN_SCK, 1);
    fastPinWrite(PIN_SDIO, 0);
    delayCycles(240);
    for (int i = 0; i < 32; i++) {
        fastPinWrite(PIN_SCK, 0);
        delayCycles(216);
        fastPinWrite(PIN_SCK, 1);
        delayCycles(216);
    }
    cmt_twi_shift16(0x8D00);

    // SOFT_RST (0xBD01) + 2 ms settle
    cmt_twi_write_reg(0x3D, 0x01);
    delayCycles(480000);
    feedWatchdogs();

    // Load 111-word RF & indirect RAM configuration table + TWI_OFF (0x8D02)
    for (size_t i = 0; i < sizeof(CMT_TWI_INIT) / sizeof(TwiCmd); i++) {
        cmt_twi_write_reg(CMT_TWI_INIT[i].reg, CMT_TWI_INIT[i].val);
    }
    feedWatchdogs();

    // 54 ms post-TWI_OFF settle with SCK=1, SDIO=0
    fastPinWrite(PIN_SCK, 1);
    fastPinWrite(PIN_SDIO, 0);
    for (int ms = 0; ms < 54; ms++) {
        delayCycles(240000);
        feedWatchdogs();
    }
}

void IRAM_ATTR cmt_tx_bytes_nrz(const uint8_t *buf, size_t len, uint32_t &nextEdgeCycle) {
    const uint32_t bitCycles = 6028; // 25.117 us/bit @ 240 MHz
    for (size_t b = 0; b < len; b++) {
        uint8_t cur = buf[b];
        for (int bit = 7; bit >= 0; bit--) {
            while ((int32_t)(getCycleCount() - nextEdgeCycle) < 0) {}
            fastPinWrite(PIN_SDIO, (cur >> bit) & 1);
            nextEdgeCycle += bitCycles;
        }
        feedWatchdogs();
    }
}

void sendSunfreePacket(uint8_t opcode) {
    uint8_t frame21[21];
    build_sunfree_frame(opcode, transmitterId, rollingCounter, frame21);

    // Drive SDIO=0 & SCK=1 BEFORE holding HC32 in reset so CMT2119A DATA never floats HIGH
    fastPinWrite(PIN_SDIO, 0);
    fastPinWrite(PIN_SCK, 1);
    pinMode(PIN_SDIO, OUTPUT);
    pinMode(PIN_SCK, OUTPUT);
    fastPinWrite(PIN_HC32_RST, 0);
    pinMode(PIN_HC32_RST, OUTPUT);
    delayCycles(24000); // 100 us reset settle

    portDISABLE_INTERRUPTS();
    feedWatchdogs();

    cmt_twi_init_433mhz();

    // Burst 1: 808-byte 0xAA wake preamble + 21-byte frame
    uint8_t aa = 0xAA;
    uint32_t nextEdge = getCycleCount();
    for (int i = 0; i < 808; i++) cmt_tx_bytes_nrz(&aa, 1, nextEdge);
    cmt_tx_bytes_nrz(frame21, 21, nextEdge);

    while ((int32_t)(getCycleCount() - nextEdge) < 0) {}
    fastPinWrite(PIN_SDIO, 0);
    for (int ms = 0; ms < 30; ms++) { delayCycles(240000); feedWatchdogs(); }

    // Bursts 2..4: 8-byte 0xAA sync preamble + 21-byte frame
    for (int rep = 0; rep < 3; rep++) {
        nextEdge = getCycleCount();
        for (int i = 0; i < 8; i++) cmt_tx_bytes_nrz(&aa, 1, nextEdge);
        cmt_tx_bytes_nrz(frame21, 21, nextEdge);

        while ((int32_t)(getCycleCount() - nextEdge) < 0) {}
        fastPinWrite(PIN_SDIO, 0);
        int gapMs = (rep < 2) ? 30 : 25; // 25 ms (> t_STOP = 20 ms) enters <20 nA sleep
        for (int ms = 0; ms < gapMs; ms++) { delayCycles(240000); feedWatchdogs(); }
    }

    portENABLE_INTERRUPTS();

    Serial.printf("TX Opcode=0x%02X | ID=%08lX | CNT=%u\n",
                  opcode, (unsigned long)transmitterId, rollingCounter);
    rollingCounter++;
}

void pairBlind() {
    sendSunfreePacket(CMD_PAIR_S0); // 0x4D at t = 0
    delay(1025);                    // ~1.3 s start-to-start spacing
    sendSunfreePacket(CMD_PAIR_S1); // 0x5D
    delay(1225);                    // ~1.5 s start-to-start spacing
    sendSunfreePacket(CMD_PAIR_S2); // 0x6D
}

void setup() {
    setCpuFrequencyMhz(240);
    Serial.begin(115200);

    // Hold stock HC32F002 in reset while keeping CMT2119A bus idle (SDIO=0, SCK=1)
    fastPinWrite(PIN_SDIO, 0);
    fastPinWrite(PIN_SCK, 1);
    pinMode(PIN_SDIO, OUTPUT);
    pinMode(PIN_SCK, OUTPUT);
    fastPinWrite(PIN_HC32_RST, 0);
    pinMode(PIN_HC32_RST, OUTPUT);

    delay(1000);
    sendSunfreePacket(CMD_UP);
}

void loop() {}
```

