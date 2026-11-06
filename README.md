# PAW3204DB optical sensor Arduino driver

Embedded C++ driver and PlatformIO firmware for interfacing the PixArt PAW3204DB optical mouse sensor with an Arduino Nano.

## Purpose

This project provides an embedded C++ firmware implementation to communicate with the PixArt PAW3204DB optical navigation sensor. The code runs on an ATmega328P microcontroller using PlatformIO and the Arduino framework. It reads relative X and Y coordinate displacements, configures internal sensor registers, and outputs real-time motion data over a 9600 baud serial connection.

The driver is useful for robotics odometry, custom optical mouse designs, trackballs, and surface motion tracking systems that require direct register access to optical navigation sensors.

## Architecture

The firmware communicates with the sensor over a synchronous two-wire serial interface consisting of a clock line and a bidirectional data line.

### Serial communication

The driver controls transmission timing directly through bit-banged GPIO operations.
- The clock line (CLK) synchronizes data transfers.
- The serial data input and output line (SDIO) handles half-duplex command writes and register reads.
- Writing a register sets the most significant bit to 1, followed by the register address and data byte.
- Reading a register sends the 8-bit address, switches the data line to input mode, and reads the returned byte on clock cycles.

### Analog threshold reading

The PAW3204DB operates at 1.8V to 3.3V logic levels. When connected to a 5V Arduino Nano through a passive voltage divider, the SDIO read routine uses the ATmega328P analog-to-digital converter on pin A4.
- The microcontroller ADC runs with a prescaler of 16 to reduce conversion latency.
- An analog threshold of 368 ADC counts marks the logic high boundary for 1.8V signals.
- An active bidirectional level shifter also enables direct digital reading on standard GPIO pins.

### Motion processing

The sensor tracks surface textures and stores relative movements in internal registers.
- Register `0x02` reports motion status, including surface detection and overflow flags.
- Registers `0x03` and `0x04` store Delta X and Delta Y movements in two's complement format.
- A conversion helper transforms two's complement bytes into signed integers.
- A noise threshold filter suppresses readings below 2 counts to eliminate sensor jitter.

### Algorithms and data representation

The driver uses specific algorithms and bit-level data structures to process sensor signals:
- Two's complement integer conversion. An 8-bit two's complement decoding algorithm converts raw register readings into signed directional displacements for X and Y coordinates.
- Bit-banged serial shift algorithm. Microsecond pulse sequences clock 8-bit addresses and data words in MSB-first order across the bidirectional bus.
- Fast ADC threshold evaluation. Bitwise register configuration on the ATmega328P sets the ADC prescaler to 16, allowing fast voltage sampling of the sensor line.
- Noise filtering threshold. A threshold algorithm discards values within the deadband between -2 and +2 counts to isolate intentional movement from baseline jitter.

## Setup

### Hardware setup

The table below describes the pin connections between the Arduino Nano and the PAW3204DB sensor module.

| Arduino Nano pin | Sensor pin | Function | Notes |
| --- | --- | --- | --- |
| D3 | CLK | Clock | Master clock output |
| A4 | SDIO | Data | Bidirectional serial data line |
| 3V3 | VDD | Power | Supply voltage between 1.8V and 3.3V |
| GND | GND | Ground | Common system ground reference |

The PAW3204DB sensor does not tolerate 5V signals. Supplying 5V directly to sensor power or communication pins will damage the sensor. Use a 3.3V power supply and a bidirectional logic level shifter or calibrated voltage divider.

### Software setup

This repository uses PlatformIO for compilation and deployment.

#### Prerequisites

Install the PlatformIO Core CLI or the PlatformIO IDE extension in Visual Studio Code.
- Python 3.8 or newer
- PlatformIO Core CLI

#### Build and upload

1. Clone the repository:
   ```bash
   git clone https://github.com/devsanthoshmk/main.git
   cd main
   ```

2. Compile the firmware:
   ```bash
   pio run
   ```

3. Flash the binary to an Arduino Nano:
   ```bash
   pio run --target upload
   ```

4. Open the serial terminal:
   ```bash
   pio device monitor --baud 9600
   ```

## Usage

When the Arduino boots, the program executes the following sequence:
1. Configures clock and data pins as outputs and sets them high.
2. Configures the ATmega328P ADC registers for rapid analog conversions on pin A4.
3. Initializes serial communication at 9600 baud.
4. Reads `Product_ID_1` (register `0x00`, expected `0x30`) and `Product_ID_2` (register `0x01`).
5. Writes the target sleep mode and resolution settings to internal registers.
6. Enters the continuous polling loop.

In the polling loop, the microcontroller checks the motion status register every 2 milliseconds. When the sensor detects valid motion, the driver prints the signed relative displacement to the serial console:
```text
  ---  X : 4  --- Y : -2
  ---  X : 12  --- Y : 8
```

## Configuration

You can adjust sensor parameters at the top of `src/main.cpp`.

| Parameter | Default value | Description |
| --- | --- | --- |
| `CLK_Pin` | `3` | Arduino pin number assigned to the serial clock line |
| `SDIO_Pin` | `A4` | Arduino pin number assigned to the bidirectional data line |
| `RESOLUTION` | `0x06` | Resolution index from `0x00` (lowest CPI) to `0x07` (highest CPI) |
| `SLEEP_MODE` | `0xB8` | Power management mode. Set to `0xA0` to disable sleep mode |
| `NOISE_FILTER` | `0x02` | Minimum displacement count required to report movement |
| `DELAY` | `500` | Milliseconds to wait between register initialization steps |

## Troubleshooting

- If the system does not detect the sensor, verify ground continuity and confirm that the sensor receives 1.8V to 3.3V. Check that Product ID registers return non-zero numbers.
- If coordinate movements are inverted, invert the sign of Delta X or Delta Y in `loop()` to match the physical orientation of the sensor.
- If analog read errors occur, check the voltage divider resistors on pin A4. When using a bidirectional level shifter, change `SDIO_Pin` to a digital GPIO and replace `analogRead` with `digitalRead`.
