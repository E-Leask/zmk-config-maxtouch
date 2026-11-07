# ZMK maXTouch Configuration & Hardware Debugging Guide

This repository contains the ZMK firmware configuration and shield definitions for testing Microchip / Atmel **maXTouch** capacitive touch controllers on the **nice!nano** (nRF52840).

---

## 1. Serial Console & Interactive Shell

The firmware includes USB CDC ACM serial logging and the Zephyr interactive I2C shell enabled in [maxtouch_test.conf](file:///c:/Users/Eric/Documents/GitHub/zmk-config-maxtouch/config/boards/shields/maxtouch_test/maxtouch_test.conf).

### Connecting with SimplySerial (`ss`)
```powershell
ss -com:17 -baud:115200
```
*(Press <kbd>Ctrl</kbd> + <kbd>X</kbd> to exit the session).*

---

## 2. Hardware Architecture & Addressing

* **I2C Bus Controller**: `i2c@40003000` (Nordic TWIM0 on-chip peripheral mapped to `0x40003000`).
* **maXTouch Sensor Address**: `0x4B` on the I2C bus (`touchpad: maxtouch@4b`).
* **Register Addressing**: Microchip maXTouch uses **16-bit little-endian** addressing:
  * Address = `0xHHL`
  * First byte transmitted = **LSB** (`0xLL`)
  * Second byte transmitted = **MSB** (`0xHH`)

### How to Read Registers in the Zephyr I2C Shell
Because Zephyr's `i2c read` command is hardcoded to an 8-bit register parameter, 16-bit addresses are accessed using a 2-step sequence:
1. **Set 16-bit address pointer**:
   ```text
   i2c write i2c@40003000 4b <LSB> <MSB>
   ```
2. **Read bytes (repeating LSB as register argument)**:
   ```text
   i2c read i2c@40003000 4b <LSB> <num_bytes>
   ```

---

## 3. Information Block (`0x0000`)

The 7-byte ID Information Block is located at address `0x0000`.

### Read Commands
```text
i2c write i2c@40003000 4b 0x00 0x00
i2c read i2c@40003000 4b 0x00 7
```

### Verified Output
```text
00000000: a6 0a 11 aa 0c 0c 22
```

### Field Breakdown

| Byte | Value | Field | Description |
| :---: | :---: | :--- | :--- |
| **0** | `0xA6` | `family_id` | maXTouch Family (e.g., mXT640T / mXT1066T family) |
| **1** | `0x0A` | `variant_id` | Silicon variant / revision (10) |
| **2** | `0x11` | `version` | Firmware version (v1.1) |
| **3** | `0xAA` | `build` | Firmware build number (170) |
| **4** | `0x0C` | `matrix_x_size` | 12 X sensor lines |
| **5** | `0x0C` | `matrix_y_size` | 12 Y sensor lines |
| **6** | `0x22` | `num_objects` | **34 objects** in the Object Table (`0x22` = 34 decimal) |

---

## 4. Complete Object Table & I2C Commands (34 Objects)

The chip exposes 34 functional objects starting after the Information Block:

| Obj | Type | Address | Size | Inst | Rep | Description | I2C Write (Set Pointer) | I2C Read |
| :---: | :--- | :---: | :---: | :---: | :---: | :--- | :--- | :--- |
| **0** | **T37** | `0x00D6` | 130 | 1 | 0 | Diagnostic Debug Data | `i2c write i2c@40003000 4b 0xd6 0x00` | `i2c read i2c@40003000 4b 0xd6 16` |
| **1** | **T44** | `0x0158` | 1 | 1 | 0 | Message Count | `i2c write i2c@40003000 4b 0x58 0x01` | `i2c read i2c@40003000 4b 0x58 1` |
| **2** | **T5** | `0x0159` | 11 | 1 | 0 | Message Processor (FIFO) | `i2c write i2c@40003000 4b 0x59 0x01` | `i2c read i2c@40003000 4b 0x59 11` |
| **3** | **T34** | `0x0164` | 7 | 53 | 0 | Sensor Config | `i2c write i2c@40003000 4b 0x64 0x01` | `i2c read i2c@40003000 4b 0x64 7` |
| **4** | **T6** | `0x02D7` | 7 | 1 | 1 | Command Processor | `i2c write i2c@40003000 4b 0xd7 0x02` | `i2c read i2c@40003000 4b 0xd7 7` |
| **5** | **T68** | `0x02DE` | 73 | 1 | 1 | Serial Data / GPIO | `i2c write i2c@40003000 4b 0xde 0x02` | `i2c read i2c@40003000 4b 0xde 16` |
| **6** | **T38** | `0x0327` | 64 | 1 | 0 | User Data Storage | `i2c write i2c@40003000 4b 0x27 0x03` | `i2c read i2c@40003000 4b 0x27 16` |
| **7** | **T71** | `0x0367` | 200 | 1 | 0 | Dynamic Configuration | `i2c write i2c@40003000 4b 0x67 0x03` | `i2c read i2c@40003000 4b 0x67 16` |
| **8** | **T110** | `0x042F` | 24 | 4 | 0 | Sensor Configuration | `i2c write i2c@40003000 4b 0x2f 0x04` | `i2c read i2c@40003000 4b 0x2f 16` |
| **9** | **T7** | `0x048F` | 5 | 1 | 0 | Power Configuration | `i2c write i2c@40003000 4b 0x8f 0x04` | `i2c read i2c@40003000 4b 0x8f 5` |
| **10** | **T8** | `0x0494` | 15 | 1 | 0 | Acquisition Configuration | `i2c write i2c@40003000 4b 0x94 0x04` | `i2c read i2c@40003000 4b 0x94 15` |
| **11** | **T15** | `0x04A3` | 11 | 1 | 1 | Key Array | `i2c write i2c@40003000 4b 0xa3 0x04` | `i2c read i2c@40003000 4b 0xa3 11` |
| **12** | **T18** | `0x04AE` | 2 | 1 | 0 | Communication Config | `i2c write i2c@40003000 4b 0xae 0x04` | `i2c read i2c@40003000 4b 0xae 2` |
| **13** | **T24** | `0x04B0` | 19 | 1 | 4 | One-Touch Gestures | `i2c write i2c@40003000 4b 0xb0 0x04` | `i2c read i2c@40003000 4b 0xb0 16` |
| **14** | **T25** | `0x04C3` | 16 | 1 | 1 | Self Test | `i2c write i2c@40003000 4b 0xc3 0x04` | `i2c read i2c@40003000 4b 0xc3 16` |
| **15** | **T27** | `0x04D3` | 7 | 1 | 1 | Stretch Filter | `i2c write i2c@40003000 4b 0xd3 0x04` | `i2c read i2c@40003000 4b 0xd3 7` |
| **16** | **T40** | `0x04DA` | 5 | 1 | 0 | Grip Suppression | `i2c write i2c@40003000 4b 0xda 0x04` | `i2c read i2c@40003000 4b 0xda 5` |
| **17** | **T42** | `0x04DF` | 14 | 1 | 0 | Proximity Sensing | `i2c write i2c@40003000 4b 0xdf 0x04` | `i2c read i2c@40003000 4b 0xdf 14` |
| **18** | **T46** | `0x04ED` | 12 | 1 | 1 | CTE Configuration | `i2c write i2c@40003000 4b 0xed 0x04` | `i2c read i2c@40003000 4b 0xed 12` |
| **19** | **T51** | `0x04F9` | 13 | 1 | 1 | Touch Suppression | `i2c write i2c@40003000 4b 0xf9 0x04` | `i2c read i2c@40003000 4b 0xf9 13` |
| **20** | **T56** | `0x0506` | 16 | 1 | 1 | Shieldless Integration | `i2c write i2c@40003000 4b 0x06 0x05` | `i2c read i2c@40003000 4b 0x06 16` |
| **21** | **T61** | `0x0516` | 5 | 6 | 1 | Timer Configuration | `i2c write i2c@40003000 4b 0x16 0x05` | `i2c read i2c@40003000 4b 0x16 5` |
| **22** | **T65** | `0x0534` | 23 | 3 | 1 | Lens Bending / Position | `i2c write i2c@40003000 4b 0x34 0x05` | `i2c read i2c@40003000 4b 0x34 16` |
| **23** | **T70** | `0x0579` | 10 | 20 | 1 | Dynamic Gesture | `i2c write i2c@40003000 4b 0x79 0x05` | `i2c read i2c@40003000 4b 0x79 10` |
| **24** | **T72** | `0x0641` | 89 | 1 | 1 | Noise Suppression | `i2c write i2c@40003000 4b 0x41 0x06` | `i2c read i2c@40003000 4b 0x41 16` |
| **25** | **T78** | `0x069A` | 12 | 1 | 0 | Gesture Processor | `i2c write i2c@40003000 4b 0x9a 0x06` | `i2c read i2c@40003000 4b 0x9a 12` |
| **26** | **T80** | `0x06A6` | 14 | 1 | 1 | Retransmission Config | `i2c write i2c@40003000 4b 0xa6 0x06` | `i2c read i2c@40003000 4b 0xa6 14` |
| **27** | **T100**| `0x06B4` | 62 | 1 | 8 | Multiple Touch Touchscreen| `i2c write i2c@40003000 4b 0xb4 0x06` | `i2c read i2c@40003000 4b 0xb4 16` |
| **28** | **T104**| `0x06F2` | 11 | 1 | 0 | Aux Touch Screen | `i2c write i2c@40003000 4b 0xf2 0x06` | `i2c read i2c@40003000 4b 0xf2 11` |
| **29** | **T108**| `0x06FD` | 75 | 1 | 1 | Dual Config | `i2c write i2c@40003000 4b 0xfd 0x06` | `i2c read i2c@40003000 4b 0xfd 16` |
| **30** | **T109**| `0x0748` | 9 | 1 | 1 | Touch Screen Status | `i2c write i2c@40003000 4b 0x48 0x07` | `i2c read i2c@40003000 4b 0x48 9` |
| **31** | **T111**| `0x0751` | 30 | 2 | 0 | Hover / Stylus Support | `i2c write i2c@40003000 4b 0x51 0x07` | `i2c read i2c@40003000 4b 0x51 16` |
| **32** | **T113**| `0x078D` | 3 | 1 | 0 | Low Power Gesture | `i2c write i2c@40003000 4b 0x8d 0x07` | `i2c read i2c@40003000 4b 0x8d 3` |
| **33** | **T121**| `0x0790` | 3 | 1 | 0 | Active Stylus | `i2c write i2c@40003000 4b 0x90 0x07` | `i2c read i2c@40003000 4b 0x90 3` |

---

## 5. Reading Touch Reports from Message Processor (T5)

Object **T5** (`0x0159`) is the FIFO message buffer where touch event reports are delivered.

```text
# 1. Set pointer to T5
i2c write i2c@40003000 4b 0x59 0x01

# 2. Read report (e.g., 8 bytes)
i2c read i2c@40003000 4b 0x59 8
```

* **When FIFO is empty**: Returns `0xFF 0x00 0x00 0x00...` (`0xFF` indicates report ID 255 / invalid / empty).
* **When a touch event occurs**: Byte 0 is the report ID (assigned to T100 / T9), followed by status flags, $X/Y$ coordinates, and touch size.
