# Smart 24-Valve ESPHome LoRa Irrigation Controller

A long-range, smart irrigation system built with ESPHome. It uses a Gateway Node to bridge a standard Wi-Fi network to a Remote Node via LoRa, allowing control of up to 24 separate valves without needing Wi-Fi coverage in the field.

## Use Case & Architecture

This system is designed to run as a standalone, but also run in parallel with a traditional commercial irrigation controller (such as a Hunter ICC) operating standard 24V AC solenoid valves. By tapping into the existing 24V AC power, this LoRa setup adds long-range smart home control without disrupting the existing infrastructure.

The system consists of two microcontrollers communicating over an 868 MHz LoRa link using a highly efficient custom 5-byte bitmask protocol:

1. **Gateway Node (gateway-node.yaml):** Connects to Wi-Fi and Home Assistant. It maintains 24 virtual template switches. When toggled, it calculates a 32-bit integer mask and transmits it over LoRa.
2. **Remote Node (remote-node.yaml):** It receives the LoRa bitmask, applies the state to a CH423 I2C I/O expander driving 24 relays and transmits the exact mask back as an acknowledgment.

*Note: The Gateway only updates its internal state to "ON" after receiving the ACK packet back from the remote node, ensuring 100% sync reliability over the radio link.*

## Home Assistant Integration

Because the Gateway Node is built with ESPHome, it will automatically be discovered by Home Assistant on your Wi-Fi network. The gateway exposes the following entities to your dashboard:
* **24x Valve Controls:** Exposed as standard switches. Toggling these in Home Assistant instantly sends the updated bitmask to the remote node.
* **Signal Diagnostics:** Real-time (LoRa RSSI) and (LoRa SNR) sensors for monitoring the radio link quality.
* **Connectivity Monitor:** A (Remote Node Online) binary sensor that warns you if the field node misses its 30-second keepalive ping.

## Hardware Bill of Materials (HBOM)

**Microcontrollers & Radio**
* 2x ESP32-C6 Development Boards
* 2x RFM95 Ultra-long Range Transceiver LoRa Modules (868MHz) with breakout boards
* 2x LoRa Outdoor Waterproof Antennas (868MHz - LPWA 5dBi)
* 2x U.FL to SMA adapter cables

**Relay & Power Control (Remote Node)**
* 1x CH423 I/O Expander
* 3x 8-Channel 5V Relay Boards (Active-Low)
* 1x AC/DC-DC LM2596HV Step-Down Converter (Converts 24V AC down to 5V DC for the relay boards)

## Recommended Alternative Hardware (Beginner Friendly)

If you are replicating this project and want to avoid delicate RF soldering, a safer and more user-friendly approach is to use 2x TTGO LoRa32 v2.1 (868MHz) development boards instead of the discrete ESP32-C6 and RFM95 modules. 

The TTGO boards integrate the microcontroller and the LoRa radio into a single package and feature a factory-installed U.FL connector for the antenna. This greatly reduces the complexity of the hardware build and protects the radio from accidental damage.

*Disclaimer: The YAML configurations in this repository (gateway-node.yaml and remote-node.yaml) were explicitly built and tested using the ESP32-C6 and separate SPI RFM95 modules. If you opt for the TTGO LoRa32 boards, you will need to modify the YAML files to update the ESP32 variant and change the SPI/LoRa pin definitions to match the TTGO's internal pinout.*

## Hardware Warnings & Safety

**CRITICAL: LoRa Antenna Required**
Never power on the ESP32 or the RFM95 LoRa module without the 5dBi antenna physically connected to the radio board. If the module attempts to transmit a packet without an antenna, the RF energy will reflect back into the chip and permanently destroy the radio's amplifier. Always connect the antenna BEFORE applying power.

**RFM95 Antenna Soldering Modification**
The specific RFM95 LoRa modules used in this build do not come with a built-in U.FL connector. To attach the external 5dBi antennas, a U.FL to SMA adapter cable must be manually soldered to the LoRa module:
* Core Wire: Solder the inner core of the adapter cable directly to the Antenna (ANT/ANA) pad on the LoRa module.
* RF Shielding: Solder the outer braided RF shielding of the adapter cable to the Ground (GND) pad. 
* Make sure this is a very strong, stable solder connection. Poor connections here will cause severe signal loss or damage the transmitter. Once the adapter is securely soldered, screw the SMA antenna onto the adapter cable before powering up the board.

**External Power for Relays**
Do not attempt to power the three 8-channel relay boards directly from the ESP32's 5V or 3.3V pins. The LM2596HV is required to step down the Hunter controller's 24V AC to 5V DC specifically to handle the current draw of the relay coils. Ensure the ground (GND) of your 5V DC output is shared with the GND of the ESP32-C6 and CH423.

## Pinouts

Both nodes utilize an ESP32-C6 and an SX127x LoRa Radio. The Remote Node adds a CH423 I2C expander.

**LoRa Radio (SPI) - Both Nodes**
* CLK: GPIO18
* MISO: GPIO19
* MOSI: GPIO20
* CS: GPIO21
* DIO0: GPIO22
* RST: GPIO15

**Relay Expander (I2C) - Remote Node Only**
* SDA: GPIO7
* SCL: GPIO0
* IC: CH423 (Pins 0-7 Bidirectional, Pins 8-23 Output-Only)

## Installation & Setup

1. Clone this repository: git clone https://github.com/LefterisPapaioannou/esp32-lora-irrigation-controller.git
2. Fill in your Wi-Fi credentials and ESPHome API key.
3. Compile and flash gateway-node.yaml to the indoor Gateway ESP32-C6.
4. Compile and flash remote-node.yaml to the field Remote ESP32-C6.
