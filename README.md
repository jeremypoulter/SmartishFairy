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
- Input voltage: 8-52V DC per driver (L6226 limit). Match each driver's supply to the rated voltage of the light strings on it (typically 24-31V for mains-adapter strings). Three independent supply inputs (IN1-IN3) can be linked with the J9/J10 jumpers.
- Each supply input has a series Schottky diode (reverse-polarity protection) and a **DNP** TVS footprint (SMBJ33A). Fit the TVS only on inputs of 31V or less - no TVS can protect the L6226 (60V absolute max) above about 33V. For higher-voltage strings, connect the adapter to the board *before* plugging it into the wall (long adapter leads plus ceramic caps can ring to almost twice the supply voltage on hot-plug).
- Output channels: 12 channels total (6 strings, each wired as one H-bridge output pair)
- Driver: 3 x L6226QTR dual full-bridge drivers. The L6226 can drive both outputs of a bridge high at the same time, which the independent output modes rely on.
- Overcurrent trip: R_CL = 22K gives about 0.5A (I = 11050 / R_CL). Each driver's EN/OCD pins form one fault node (pull-up + 5.6nF); a fault disables both bridges of that driver and is readable on its EN_FAULT GPIO.
- Logic power: a 5V buck (LMR16006X, 700kHz) runs from whichever of VS1-VS3 is powered, OR-ed (diodes) with USB 5V and the Ctl_Pwr terminal, so the board runs from a single supply. 3.3V comes from an AP2112K LDO.
- Supply voltage monitoring: each VS rail is divided by about 17 (100K/6.2K) into an RP2040 ADC pin (52V reads 3.0V).

### Data Interface
- Data_In / Data_Out terminals carry GND, CLK, DIN/DOUT and 5V. DIN and CLK are buffered into the 3.3V domain (74LVC2G17, 5V-tolerant inputs); DOUT is buffered up to 5V (74AHCT1G125). CLK is input-only and is passed through to Data_Out.
- USB D+/D- are protected by a USBLC6-2SC6 ESD array.

### Pin Allocation (RP2040)
| GPIO | Function |
|------|----------|
| 0, 1 | UART TX/RX (debug header) |
| 2-13 | CH0-CH11 driver inputs. Each bridge's IN1/IN2 are an even/odd pair so they share a PWM slice (CH0/CH1 are on GPIO12/13) |
| 14, 15, 17 | EN_FAULT1-3 (one per L6226; open-drain style - the board pulls up, firmware drives low to disable, reads low as a fault) |
| 16, 18, 19 | DIN, CLK, DOUT |
| 20-23 | Status LEDs: PWR, DATA, FAULT, SETUP (left to right on the board) |
| 26-28 | VS1-VS3 voltage sense (ADC0-2) |
| 24, 25, 29 | Spare |

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
