# SmartishFairy

## Overview
SmartishFairy is a smart controller for inexpensive consumer fairy lights that provides advanced control features at an affordable price. This project allows you to transform ordinary fairy light strings into programmable, dynamic lighting installations with various effects and control options.

### Key Features
- Control up to 12 output channels (6 pairs of lights)
- Support for high voltage input (up to 50V)
- Multiple operating modes for different control schemes
- Compatible with common lighting protocols (WS2812, SPI)
- Based on the powerful RP2040 microcontroller
- Cost-effective solution for home or commercial lighting displays

## Hardware Specifications

### Power and Output
- Input voltage: up to 50V DC
- Output channels: 12 channels total (6 sets, assuming 2 channels per set)
- Driver: 3 × L6226QTR dual channel motor drivers

### Driver Configuration
- Each driver has separate input - jumper available to link input rails
- Output side presents source voltage and ground for independent channels

### Pin Requirements
- 12 output pins (for driving the L6226QTR controllers)
- 8 configuration pins
- 3 SPI pins (possibly 4 if chip select is needed)
- Based on the RP2040 microcontroller

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
- RP2040-based board (such as Raspberry Pi Pico)
- 3 × L6226QTR dual channel motor drivers
- Appropriate power supply (rated for your fairy lights, up to 50V)
- Connecting wires/jumpers
- Fairy light strings (compatible with the voltage of your setup)

### Wiring
1. Connect the RP2040 to the L6226QTR drivers according to the pin mapping in the firmware
2. Set configuration jumpers according to your desired operating mode
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

## Contributing
Contributions to the SmartishFairy project are welcome! Please feel free to submit pull requests or open issues for bugs, questions, or feature requests.

## License
This project is licensed under the MIT License - see the LICENSE file for details.
