# SmartishFairy

> **Status - work in progress.** The KiCad 10 schematics describe the current design. The PCB layout (`SmartishFairy.kicad_pcb`) is the **previous hardware revision**, upgraded to KiCad 10 format only: it does not match the schematic (different regulator, enable circuit, power input, GPIO assignment and button) and, on that layout, the data/power terminals are wired straight to the USB VBUS. **Do not fabricate it.** Run *Update PCB from Schematic*, re-route, and pass DRC with schematic parity before ordering boards. There is also no firmware in this repository yet (see Firmware).

## Overview
SmartishFairy is a smart controller for inexpensive consumer fairy lights that provides advanced control features at an affordable price. This project allows you to transform ordinary fairy light strings into programmable, dynamic lighting installations with various effects and control options.

### Key Features
- Control up to 12 output channels (6 pairs of lights)
- Single 12-50V main input; strings up to about 48V (run each output at its light string's rated voltage)
- Multiple operating modes for different control schemes, configured over USB (no DIP switches)
- Compatible with common lighting protocols (WS2812, SPI)
- Based on the powerful RP2040 microcontroller
- Cost-effective solution for home or commercial lighting displays

## Hardware Specifications

### Power and Output
- **Single main input, 12-50 V DC** (`MAIN_IN`): Data_In pin 4 (V+ over the lighting-control cable) or the Main_In terminal, diode-OR-ed for reverse-polarity protection. Data_Out pin 4 passes the input power on through a 3A PTC fuse. A TVS footprint is provided but unfitted; fit SMBJ28A only on 12-24 V builds (no TVS can protect the 60 V-rated parts above about 33 V), and connect the supply to the board before plugging it into the wall.
- **High-voltage bus (`HVBUS`)** feeds the per-driver supplies. Build options: **pass-through** (SS310 from MAIN_IN: MAIN_IN is already the string voltage, 24-50 V) and/or a **boost converter** for a "12 V mode" (XL6019E1, software-set 26-50 V, about 30 W from 12 V, MAIN_IN <= ~24 V on boost builds (a boost cannot output less than its input minus the diode drop, and the XL6019E1 VIN absolute maximum is 45 V); the diode must be >= 60 V).
- **Per-driver supply:** either a **software-set LM5164 buck** (HVBUS -> VSn, about 24-50 V, 1 A each) or a 0R link from HVBUS to VSn (shared voltage). Fit one or the other for each driver.
- **Software control:** an I2C DAC (MCP4728, one channel per converter) injects current into each converter's feedback pin; the ADC reads VS1-VS3 and HVBUS back so the firmware can trim to the exact voltage. Every converter enable has a pull-down, so nothing runs until the firmware has set the DAC. Set the DAC's EEPROM power-up values to the maximum code (lowest voltage) for the three bucks and to 0 V for the boost, so nothing is injected into the boost feedback pin while MAIN_IN is absent (USB-only). The DAC is powered from 3.3 V, so keep codes at or below 3.2 V. Keep each driver's VSn at least about 8 V below HVBUS: the buck's ripple injection depends on that headroom.
- **Build straps:** an I2C expander (PCA9538) drives the converter enables and reads one strap per optional block (boost, buck 1-3): a 0R to GND in the block means it is fitted, a 100K pull-up means it is not. One firmware therefore adapts to whatever was assembled.
- Output channels: 12 channels total (6 strings, each wired as one H-bridge output pair), 3 x L6226QTR dual full-bridge drivers. The L6226 can drive both outputs of a bridge high at once, which the independent output modes rely on. Sized for one 1000-LED string per output (about 6-12 W at 31 V).
- Overcurrent trip: R_CL = 22K gives about 0.5A (I = 11050 / R_CL). Each driver's EN/OCD pins form one fault node (pull-up + 5.6nF); a fault disables both bridges of that driver and is readable on its EN_FAULT GPIO.
- Logic power: a 5 V LM5164 buck (about 320 kHz, 1 A) from MAIN_IN, set to 5.3 V so the rail still has at least 4.5 V behind the output diodes, feeds the system 5 V rail (diode-OR-ed with USB 5 V, so USB-only bench configuration works) and, separately, the optional ESP module. 3.3 V comes from an AP2112K LDO off the 5 V rail. The 5 V DOUT buffer is only guaranteed on main power; on USB-only power the rail can sag below the buffer's 4.5 V minimum.
- Supply voltage monitoring: each VS rail is divided by about 17 (100K/6.2K) into an RP2040 ADC pin (52 V reads 3.0 V). A 10k + 10nF filter after each divider limits injection into the ADC pins while 3.3 V is still starting.

### Build variants (default population)
The schematic marks the optional converter blocks DNP, so the **default build is the simple one**: pass-through diode D25 plus 0R links R44-R46 from HVBUS to VS1-VS3, with all three drivers sharing the MAIN_IN voltage (24-50 V).

| Variant | Fit | Do not fit |
|---------|-----|------------|
| Default (shared voltage) | D25, R44-R46 | boost block (U12, L5, D26, R75-R79, C62-C64) and per-driver buck blocks (U15-U17 and their parts) |
| Per-driver voltage | the buck block for a driver (U15 = driver 1, U16 = 2, U17 = 3, with its parts and 0R strap) | that driver's 0R link (R44 = 1, R45 = 2, R46 = 3) - never both |
| 12 V mode (low-power) | boost block U12; MAIN_IN <= ~24 V | - |

The unfitted blocks' 0R strap resistors are also left off, so the firmware reads them as "not fitted".

### Optional ESP32 / WLED module
A Wemos D1 mini (generic pinout) pair of 1x8 sockets, with the module stacked above the board (low parts can sit underneath; allow roughly 8 mm of clearance), plus a 3-pin header (5V / DATA / GND) for any other module. External-antenna module variants avoid any WiFi keep-out concern.
- **Power:** from the module's 5V pin, from the main-power buck only (not USB). Flash it through its own USB. Its 3V3 pin is not connected.
- **Data:** the module shares the board's data-in pins, on the assumption that no external data source is connected when it is fitted. A single-wire stream (WS2812 timing) from the 3-pin header DATA pin or D7 goes through the same buffer as Data_In. The full SPI link uses the RP2040's SPI0 slave pins with no extra components: D5 SCK -> CLK (GPIO18), D7 MOSI -> DIN (GPIO16), D6 MISO <- DOUT (GPIO19), D8 CS -> ESP_CS (GPIO17). Leave GPIO17 without a pull-up: an ESP8266 D8 must be low at boot.

### Data Interface
- Data_In / Data_Out terminals carry GND, CLK, DIN/DOUT and V+ (MAIN_IN, passed through). DIN and CLK are buffered into the 3.3V domain (74LVC2G17, 5V-tolerant inputs); DOUT is buffered up to 5V (74AHCT1G125) from the 5 V rail. CLK is input-only and is passed through to Data_Out.
- USB D+/D- are protected by a USBLC6-2SC6 ESD array.

### Pin Allocation (RP2040)
| GPIO | Function |
|------|----------|
| 0, 1 | UART TX/RX (debug header) |
| 2-13 | CH0-CH11 driver inputs. Each bridge's IN1/IN2 are an even/odd pair so they share a PWM slice (CH0/CH1 are on GPIO12/13) |
| 14, 15, 23 | EN_FAULT1-3 (one per L6226; open-drain style - the board pulls up, firmware drives low to disable, reads low as a fault) |
| 16, 17, 18, 19 | SPI0 pins: DIN (MOSI), ESP_CS, CLK (SCK), DOUT (MISO) |
| 20-22 | Status LEDs: PWR, DATA, FAULT (left to right on the board) |
| 26-28 | VS1-VS3 voltage sense (ADC0-2) |
| 24, 25 | I2C0 SDA/SCL (DAC and expander) |
| 29 | HV_SENSE - HVBUS voltage (ADC3) |

### Test pads
VS1-VS3, HVBUS, MAIN_IN, ESP_5V, +5V, +3V3 and two GND pads.

## Configuration
There are no DIP switches. Settings (output modes and input mode below) are stored in flash and changed over the USB-C port (USB serial console), and a BOOT/SETUP push button is fitted on the QSPI chip-select line (hold it while resetting or powering up to enter the USB bootloader; the firmware can also read it as a user button, using the standard Pico-SDK BOOTSEL technique, to step through presets with the status LEDs showing the selection).

## Operating Modes

### Output Modes (2 bits per driver, 6 bits total)
| Mode | Description |
|------|-------------|
| 0 | Switching - Simple on/off control |
| 1 | TBD (Reserved for future implementation) |
| 2 | Independent high is on - Active high control |
| 3 | Independent low is on - Active low control |

### Input Modes (2 bits)
| Mode | Protocol | Description |
|------|----------|-------------|
| 0 | WS2812 | Neopixel-compatible protocol |
| 1 | SPI | Standard SPI interface |
| 2 | RGB | RGB values converted to brightness (12 virtual RGB lights map to 12 outputs) |
| 3 | Single | Each input channel controls an output (4 virtual RGB elements map to 12 output channels) |

## Installation and Setup

### Hardware Requirements
- The SmartishFairy board (RP2040 + 3 x L6226QTR dual full-bridge drivers)
- Appropriate power supply (rated for your fairy lights, up to 50V)
- Connecting wires/jumpers
- Fairy light strings (compatible with the voltage of your setup)

### Wiring
1. Check which build variant your board is (see Build variants) and, for per-driver buck builds, that each driver's voltage is at or below the string's rating.
2. Connect the main power supply (12-50 V) to Data_In pin 4 or to the Main_In terminal, with its ground, **before** switching it on at the wall.
3. Connect the fairy-light strings to the output terminals.
4. Connect USB-C and set the operating modes (see Configuration).

Wiring diagrams have not been written yet.

## Firmware
The firmware is **not part of this repository yet** - it currently holds the hardware design only. The hardware expects firmware that:
- reads the build straps (PCA9538 inputs) to learn which converter blocks are fitted;
- keeps every converter disabled until the DAC (MCP4728) has been set to a safe code, then enables and trims it from the VS/HVBUS ADC readings;
- uses the GPIO map above (SPI0 slave on GPIO16-19, I2C0 on GPIO24/25, ADC0-3 for VS1-3 and HVBUS);
- stores its configuration in flash and exposes it over USB serial.

## Usage Examples

### Basic Control
To control the lights with basic on/off functionality, set the output mode to 0 (switching).

### RGB Control
To use RGB color control:
1. Set the input mode to 2 (RGB)
2. Send RGB color values via the configured input protocol
3. The controller automatically maps these values to brightness levels on the physical outputs

### Independent Channel Control
For direct control of individual channels:
1. Set the input mode to 3 (Single)
2. Control each output channel independently

## Troubleshooting

| Problem | Possible Solution |
|---------|------------------|
| No lights turn on | Check power supply and connections |
| Lights flicker unexpectedly | Verify ground connections and check for EMI sources |
| Cannot program the controller | Ensure USB connection is secure and drivers are installed |

## Resources
- [RP2040 Hardware Design Guide](https://datasheets.raspberrypi.com/rp2040/hardware-design-with-rp2040.pdf)
- [L6226QTR Datasheet](https://www.st.com/resource/en/datasheet/l6226q.pdf)
- [LMR16006 Datasheet](https://www.ti.com/lit/ds/symlink/lmr16006.pdf)

## Contributing
Contributions to the SmartishFairy project are welcome! Please feel free to submit pull requests or open issues for bugs, questions, or feature requests.

## License
This project is licensed under the MIT License - see the LICENSE file for details.
