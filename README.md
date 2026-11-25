# SmartishFairy

## Overview
SmartishFairy is a smart controller for inexpensive consumer fairy lights that provides advanced control features at an affordable price. This project allows you to transform ordinary fairy light strings into programmable, dynamic lighting installations with various effects and control options.

### Key Features
- Control up to 12 output channels (6 pairs of lights)
- Support for high voltage input (8-52V per driver; each output should be run at its light string's rated voltage)
- Multiple operating modes for different control schemes, configured over USB (no DIP switches)
- Compatible with common lighting protocols (WS2812, SPI)
- Based on the powerful RP2040 microcontroller
- Cost-effective solution for home or commercial lighting displays

## Hardware Specifications

### Power and Output
- **Single main input, 12-50 V DC** (`MAIN_IN`): Data_In pin 4 (V+ over the lighting-control cable) or the Main_In terminal, diode-OR-ed for reverse-polarity protection. Data_Out pin 4 passes the input power on through a 3A PTC fuse. A TVS footprint is provided but unfitted; fit SMBJ28A only on 12-24 V builds (no TVS can protect the 60 V-rated parts above about 33 V), and connect the supply to the board before plugging it into the wall.
- **High-voltage bus (`HVBUS`)** feeds the three L6226 drivers (VS1-VS3 through 0R links). Build options: pass-through diode (MAIN_IN is already the string voltage, 24-50 V - populated today), or a boost converter for a "12 V mode" (planned; lower power by design). Planned: a software-set buck converter per driver (24-48 V) so each driver can run a different string voltage; the 0R links are fitted when the per-driver buck is not.
- Output channels: 12 channels total (6 strings, each wired as one H-bridge output pair), 3 x L6226QTR dual full-bridge drivers. The L6226 can drive both outputs of a bridge high at once, which the independent output modes rely on. Sized for one 1000-LED string per output (about 6-12 W at 31 V).
- Overcurrent trip: R_CL = 22K gives about 0.5A (I = 11050 / R_CL). Each driver's EN/OCD pins form one fault node (pull-up + 5.6nF); a fault disables both bridges of that driver and is readable on its EN_FAULT GPIO.
- Logic power: a 5 V buck from MAIN_IN feeds the system 5 V rail (diode-OR-ed with USB 5 V, so USB-only bench configuration works) and, separately, the optional ESP module. 3.3 V comes from an AP2112K LDO off the 5 V rail.
- Supply voltage monitoring: each VS rail is divided by about 17 (100K/6.2K) into an RP2040 ADC pin (52 V reads 3.0 V).

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
| 24, 25, 29 | Spare |

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
1. Connect the RP2040 to the L6226QTR drivers according to the pin mapping in the firmware
2. Set the operating modes over USB (see Configuration)
3. Connect the power supply to the input rails
4. Connect fairy lights to the output channels

For detailed wiring diagrams, refer to the `docs/wiring.md` file in this repository.

## Firmware Setup

### Prerequisites
- C/C++ development environment for RP2040
- Raspberry Pi Pico SDK
- USB cable for programming

### Installation Steps
1. Clone this repository
   ```
   git clone https://github.com/JeremyPoulter/SmartishFairy.git
   ```
2. Configure the firmware based on your setup
3. Compile and flash to the RP2040 board

For detailed firmware instructions, see `docs/firmware.md`.

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
