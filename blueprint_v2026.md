# Blueprint v2026: ESP32 DevKit V2 Device Classes

เอกสารนี้เป็น context reference หลักสำหรับพัฒนาโปรเจกต์ ESP32 DevKit V2 ด้วย PlatformIO + VS Code + Arduino Framework โดยอ้างอิงไฟล์ทั้งหมดในโฟลเดอร์นี้ และตั้งใจให้ทั้งมนุษย์และ AI ใช้เป็นพิมพ์เขียวก่อนแก้หรือสร้างโค้ดต่อ

## Project Identity

```yaml
project_name: ESP32_DEVKIT_V2_DEVICE_CLASSES
board_family: ESP32 DevKit V2 / DOIT ESP32 DEVKIT V1 compatible
platformio_env: esp32doit-devkit-v1
framework: Arduino
language: C++ header-only utility classes
serial_monitor_baud: 9600
primary_uart_policy: use Serial / UART0
rs485_direction: auto direction module, no DE/RE GPIO
external_mode_switch: true
```

## Current Repository Files

| File | Purpose |
|---|---|
| `DevRelay.h` | Relay control class, active-low by default, plus timer relay subclass |
| `DevSwitch.h` | Debounced switch input with one-loop edge events and callbacks |
| `DevIsoInput.h` | Debounced isolated input with one-loop edge events, callbacks, and activation counter |
| `DevXYMDSensor.h` | XY-MD03 temperature/humidity sensor via Modbus RTU over RS485 |
| `DevPZEM.h` | PZEM-016 AC power monitor via Modbus RTU over RS485, with simulation fallback |
| `_ESP32-Readme.md` | ESP32 DevKit V2 pin map, library reference, and AI cautions |
| `_DevXYMDSensorReadme.md` | Full XY-MD03 usage guide |
| `_PZEMReadme.md` | Full PZEM-016 usage guide and register reference |

## PlatformIO Baseline

Recommended `platformio.ini`:

```ini
[env:esp32doit-devkit-v1]
platform = espressif32
board = esp32doit-devkit-v1
framework = arduino
monitor_speed = 9600

lib_deps =
  4-20ma/ModbusMaster@^2.0.1
  adafruit/Adafruit GFX Library
  adafruit/Adafruit SSD1306
  paulstoffregen/OneWire
  milesburton/DallasTemperature
```

Useful commands:

```powershell
python -m platformio run
python -m platformio run --target upload
python -m platformio device monitor --baud 9600
```

## Hardware Policy

This project intentionally uses `Serial` / UART0 for Modbus RS485 communication because the board has an external RS232/RS485 mode switch.

Do not migrate PZEM or XY-MD03 to `Serial2` unless the hardware plan changes. The expected workflow is:

| Action | Switch Position |
|---|---|
| Upload firmware | RS232 / USB serial mode |
| Monitor/debug over USB | RS232 / USB serial mode |
| Read RS485 sensors | RS485 mode |
| Upload fails while RS485 connected | Move switch back to RS232 mode, then upload again |

UART0 pins:

| ESP32 Pin | Function | Connected Use |
|---:|---|---|
| GPIO 1 | UART0 TX | RS485 TX / USB serial TX path |
| GPIO 3 | UART0 RX | RS485 RX / USB serial RX path |

RS485 module expectation:

```text
ESP32 UART0 TX GPIO1 -> RS485 DI
ESP32 UART0 RX GPIO3 -> RS485 RO
RS485 A              -> Sensor A
RS485 B              -> Sensor B
GND                  -> Shared GND
```

The RS485 module is auto-direction, such as MAX13487-style hardware. No DE/RE control pin is used.

## ESP32 DevKit V2 Pin Reference

Preferred pins for general I/O:

| GPIO | Input | Output | Notes |
|---:|---|---|---|
| 4 | yes | yes | Good for relay/output |
| 13 | yes | yes | General I/O |
| 14 | yes | yes | Reserved for DS18B20 1-Wire temperature sensor |
| 16 | yes | yes | General I/O, often UART2 RX if needed |
| 17 | yes | yes | General I/O, often UART2 TX if needed |
| 18 | yes | yes | SPI SCK default |
| 19 | yes | yes | SPI MISO default |
| 21 | yes | yes | I2C SDA default |
| 22 | yes | yes | I2C SCL default |
| 23 | yes | yes | SPI MOSI default |
| 25 | yes | yes | DAC1 / GPIO |
| 26 | yes | yes | DAC2 / GPIO |
| 27 | yes | yes | General I/O |
| 32 | yes | yes | ADC1 / GPIO |
| 33 | yes | yes | ADC1 / GPIO |

Input-only pins:

| GPIO | Input | Output | Notes |
|---:|---|---|---|
| 34 | yes | no | Input-only, no reliable internal pull-up/down |
| 35 | yes | no | Input-only, no reliable internal pull-up/down |
| 36 | yes | no | Input-only ADC/input |
| 39 | yes | no | Input-only ADC/input |

Use with caution:

| GPIO | Caution |
|---:|---|
| 0 | Boot mode / flash mode |
| 1 | UART0 TX, shared with upload/debug and RS485 |
| 3 | UART0 RX, shared with upload/debug and RS485 |
| 5 | Strapping pin on many boards |
| 6-11 | Connected to ESP32 flash, do not use |
| 12 | Strapping pin, boot-voltage sensitive |
| 15 | Strapping pin |

## I/O Board Utility Mapping

| GPIO | Name | Direction | Active Level | Class |
|---:|---|---|---|---|
| 34 | SW1 | Input | Active Low | `DevSwitch` |
| 35 | SW2 | Input | Active Low | `DevSwitch` |
| 32 | SW3 | Input | Active Low | `DevSwitch` |
| 33 | ISOIN1 | Input | Active Low | `DevIsoInput` |
| 27 | ISOIN2 | Input | Active Low | `DevIsoInput` |
| 17 | RL1 | Output | Active Low | `DevRelay` |
| 16 | RL2 | Output | Active Low | `DevRelay` |
| 4 | RL3 | Output | Active Low | `DevRelay` |
| 12 | AUX1 | Output | Active High | Direct GPIO / relay active-high mode |
| 14 | DS18B20 | Input/1-Wire | 1-Wire data | `OneWire` + `DallasTemperature` |
| 15 | AUX3 | Output | Active High | Direct GPIO / relay active-high mode |
| 25 | AUX4 | Output | Active High | Direct GPIO / relay active-high mode |

For GPIO 34 and 35, prefer external pull-up/pull-down resistors because these pins are input-only and should not depend on internal pull resistors.

## OLED Reference

OLED target:

```yaml
device: SSD1306 OLED
resolution: 128x64
address: 0x3C
interface: I2C
sda: GPIO 21
scl: GPIO 22
libraries:
  - Wire
  - Adafruit GFX Library
  - Adafruit SSD1306
```

If the OLED does not display, scan I2C and check whether the address is `0x3C` or `0x3D`.

## DS18B20 Reference

The board has a DS18B20 temperature sensor connected to GPIO14. Treat GPIO14 as reserved for the 1-Wire temperature bus, not as AUX output, unless the hardware is physically changed.

```yaml
device: DS18B20
interface: 1-Wire
data_pin: GPIO 14
recommended_pullup: 4.7k ohm from DATA to 3V3
libraries:
  - OneWire
  - DallasTemperature
```

Basic usage:

```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

#define DS18B20_PIN 14

OneWire oneWire(DS18B20_PIN);
DallasTemperature ds18b20(&oneWire);

void setup() {
  Serial.begin(9600);
  ds18b20.begin();
}

void loop() {
  ds18b20.requestTemperatures();
  float tempC = ds18b20.getTempCByIndex(0);

  if (tempC != DEVICE_DISCONNECTED_C) {
    Serial.printf("DS18B20: %.2f C\n", tempC);
  }

  delay(1000);
}
```

Notes:

- Use a 4.7k pull-up resistor between DATA and 3V3 for stable 1-Wire communication.
- If multiple DS18B20 sensors are placed on the same bus later, use device addresses instead of only `getTempCByIndex(0)`.
- Keep DS18B20 reads outside tight Modbus timing windows when possible, especially if the loop also talks over UART0/RS485.

## Class Contracts

### DevRelay

Header: `DevRelay.h`

Purpose: control relay output, active-low by default.

Important behavior:

| Method | Meaning |
|---|---|
| `begin()` | Sets pin to output and calls `off()` |
| `on()` | Relay on; writes `LOW` when active-low |
| `off()` | Relay off; writes `HIGH` when active-low |
| `toggle()` | Switches current state |
| `setState(bool)` | `true` = on, `false` = off |
| `getState()` | Returns logical relay state |

Timer subclass:

| Method | Meaning |
|---|---|
| `onWithTimer(durationMs)` | Turns relay on and starts timer |
| `checkTimer()` | Must be called in `loop()`; turns relay off when expired |
| `cancelTimer()` | Cancels active timer |
| `getRemainingTime()` | Returns remaining milliseconds |

### DevSwitch

Header: `DevSwitch.h`

Purpose: debounced switch/push-button input with callbacks and one-loop edge detection.

Constructor:

```cpp
DevSwitch(uint8_t gpioPin, bool activeHigh = false, unsigned long debounceMs = 50);
```

Loop contract:

```cpp
sw.update();

if (sw.wasPressed()) {
  // true only in the update cycle where the debounced state changed to pressed
}

if (sw.wasReleased()) {
  // true only in the update cycle where the debounced state changed to released
}
```

Important methods:

| Method | Meaning |
|---|---|
| `begin()` | Sets `INPUT_PULLUP` for active-low, `INPUT` for active-high |
| `update()` | Reads, debounces, updates state, triggers callbacks |
| `isPressed()` | Current debounced logical state |
| `isReleased()` | Inverse of current debounced state |
| `wasPressed()` | One-loop edge event after `update()` |
| `wasReleased()` | One-loop edge event after `update()` |
| `onPress(callback)` | Called once when debounced press occurs |
| `onRelease(callback)` | Called once when debounced release occurs |
| `onClick(callback)` | Called on release after a press/release cycle |

### DevIsoInput

Header: `DevIsoInput.h`

Purpose: debounced isolated input with state tracking, callbacks, activation count, and one-loop edge detection.

Constructor:

```cpp
DevIsoInput(uint8_t gpioPin, bool activeHigh = false, unsigned long debounceMs = 50);
```

Loop contract:

```cpp
iso.update();

if (iso.wasActivated()) {
  // true only in the update cycle where the debounced state changed to active
}

if (iso.wasDeactivated()) {
  // true only in the update cycle where the debounced state changed to inactive
}
```

Important methods:

| Method | Meaning |
|---|---|
| `begin()` | Sets `INPUT_PULLUP` for active-low, `INPUT` for active-high |
| `update()` | Reads, debounces, updates state, triggers callbacks/counters |
| `isActive()` | Current debounced logical active state |
| `isInactive()` | Inverse of active state |
| `wasActivated()` | One-loop edge event after `update()` |
| `wasDeactivated()` | One-loop edge event after `update()` |
| `getActivationCount()` | Count of active transitions |
| `resetActivationCount()` | Clears activation counter |
| `getStateText()` | Returns `ACTIVE` or `INACTIVE` |

### DevXYMDSensor

Header: `DevXYMDSensor.h`

Purpose: read XY-MD03 temperature/humidity sensor via Modbus RTU.

Hardware and protocol:

```yaml
device: XY-MD03
interface: RS485
uart: Serial / UART0
baud_rate: 9600
serial_config: SERIAL_8N1
default_slave_id: 1
function_code: 0x04 Read Input Registers
temperature_register: 0x0001
humidity_register: 0x0002
scale: raw / 10.0
```

Usage:

```cpp
#include "DevXYMDSensor.h"

DevXYMDSensor sensor(&Serial, 1);

void setup() {
  Serial.begin(9600);
  sensor.begin(9600);
}

void loop() {
  if (sensor.update()) {
    float temperature = sensor.getTemperature();
    float humidity = sensor.getHumidity();
  }
  delay(2000);
}
```

Important methods:

| Method | Meaning |
|---|---|
| `begin(baudRate)` | Starts the serial port and Modbus master |
| `update()` | Reads temperature and humidity registers |
| `getTemperature()` | Last temperature in Celsius |
| `getHumidity()` | Last humidity percentage |
| `isLastReadSuccess()` | Last read status |
| `setSlaveID(id)` | Changes Modbus slave ID in the object |
| `printInfo()` | Prints sensor data to `Serial` |

Read every 2-3 seconds for stable Modbus operation.

### DevPZEM

Header: `DevPZEM.h`

Purpose: read PZEM-016 AC power monitor via Modbus RTU.

Hardware and protocol:

```yaml
device: PZEM-016
interface: RS485
uart: Serial / UART0
baud_rate: 9600
serial_config: SERIAL_8N1
default_slave_address: 0x01
rs485_transceiver: MAX13487-style auto direction
read_interval_ms: 2000
simulation_fallback: enabled in current header
```

Measured values:

| Value | Getter | Unit |
|---|---|---|
| Voltage | `getVoltage()` | V |
| Current | `getCurrent()` | A |
| Power | `getPower()` | W |
| Energy | `getEnergy()` | kWh |
| Frequency | `getFrequency()` | Hz |
| Power Factor | `getPowerFactor()` | 0.00-1.00 |
| Alarm | `getAlarmStatus()` | raw status |

Modbus register map:

| Register | Name | Count | Scale | Unit |
|---:|---|---:|---:|---|
| `0x0000` | Voltage | 1 | 0.1 | V |
| `0x0001` | Current | 2 | 0.001 | A |
| `0x0003` | Power | 2 | 0.1 | W |
| `0x0005` | Energy | 2 | 1 | Wh |
| `0x0007` | Frequency | 1 | 0.1 | Hz |
| `0x0008` | Power Factor | 1 | 0.01 | - |
| `0x0009` | Alarm Status | 1 | raw | - |
| `0x0042` | Reset Energy | 1 | write | - |

Usage:

```cpp
#include "DevPZEM.h"

DevPZEM pzem(&Serial, 0x01);

void setup() {
  Serial.begin(9600);
  pzem.begin();
}

void loop() {
  if (pzem.update() && pzem.isDataValid()) {
    Serial.println(pzem.toJSON());
  }
  delay(3000);
}
```

Important methods:

| Method | Meaning |
|---|---|
| `begin()` | Initializes Modbus, retries real sensor, falls back to simulation if no response |
| `update()` | Reads all PZEM values every 2 seconds |
| `isInitialized()` | True after begin path succeeds |
| `isDataValid()` | True when values are usable |
| `isSimulationMode()` | True when fallback simulated data is active |
| `resetEnergy()` | Writes reset command to register `0x0042` |
| `toJSON()` | Returns JSON string for MQTT/API logging |
| `setSimulationBaseLoad(watts)` | Changes simulated baseline load |

Current implementation note: `preTransmission()` and `postTransmission()` are global helper functions. They assume UART0/`Serial`, which matches this project hardware policy. If a future design adds another Modbus UART, rename or scope these helpers to avoid symbol conflicts and serial-port mismatch.

## AI Development Rules

Use these rules whenever an AI assistant modifies or generates code for this project:

1. Keep PZEM and XY-MD03 on `Serial` / UART0 unless the user explicitly changes the hardware design.
2. Do not replace the external RS232/RS485 switch workflow with a `Serial2` migration.
3. Use `DevXYMDSensor`, not older names such as `DevTempHumidity`.
4. Use active-low semantics correctly: relay `LOW = ON`, switch/input `LOW = pressed/active` when `activeHigh = false`.
5. Always call `begin()` for each device class in `setup()`.
6. Always call `update()` in `loop()` before checking edge methods such as `wasPressed()` or `wasActivated()`.
7. Treat `wasPressed()`, `wasReleased()`, `wasActivated()`, and `wasDeactivated()` as one-loop events only.
8. Do not use GPIO 34/35/36/39 as outputs.
9. Avoid GPIO 6-11 entirely.
10. Be careful with boot strapping pins GPIO 0, 5, 12, and 15.
11. Treat GPIO14 as reserved for DS18B20 1-Wire temperature sensing.
12. Include `OneWire` and `DallasTemperature` when creating code that reads DS18B20.
13. Read Modbus sensors every 2-3 seconds unless there is a strong reason to change timing.
14. For PZEM data, check `isDataValid()` before using getters in production logic.
15. For XY-MD03 data, check the boolean result of `update()` or `isLastReadSuccess()`.
16. If OLED fails, verify I2C wiring and address before rewriting display code.
17. Prefer non-blocking timers and `millis()` patterns for new device logic.
18. Keep files usable as PlatformIO `include/*.h` header files.

## Typical Main Loop Pattern

```cpp
#include <Arduino.h>
#include "DevSwitch.h"
#include "DevIsoInput.h"
#include "DevRelay.h"
#include "DevXYMDSensor.h"
#include "DevPZEM.h"
#include <OneWire.h>
#include <DallasTemperature.h>

#define DS18B20_PIN 14

DevSwitch sw1(34, false);
DevIsoInput iso1(33, false);
DevRelay relay1(17, true);
DevXYMDSensor xymd(&Serial, 1);
DevPZEM pzem(&Serial, 0x01);
OneWire oneWire(DS18B20_PIN);
DallasTemperature ds18b20(&oneWire);

unsigned long lastSensorRead = 0;
unsigned long lastDs18b20Read = 0;

void setup() {
  Serial.begin(9600);

  sw1.begin();
  iso1.begin();
  relay1.begin();

  xymd.begin(9600);
  pzem.begin();
  ds18b20.begin();
}

void loop() {
  sw1.update();
  iso1.update();

  if (sw1.wasPressed()) {
    relay1.toggle();
  }

  if (iso1.wasActivated()) {
    relay1.on();
  }

  if (millis() - lastSensorRead >= 3000) {
    lastSensorRead = millis();

    if (xymd.update()) {
      xymd.printInfo();
    }

    if (pzem.update() && pzem.isDataValid()) {
      Serial.println(pzem.toJSON());
    }
  }

  if (millis() - lastDs18b20Read >= 1000) {
    lastDs18b20Read = millis();
    ds18b20.requestTemperatures();
    float boardTempC = ds18b20.getTempCByIndex(0);

    if (boardTempC != DEVICE_DISCONNECTED_C) {
      Serial.printf("DS18B20: %.2f C\n", boardTempC);
    }
  }
}
```

## Troubleshooting Checklist

For upload problems:

- Switch external mode to RS232 / USB serial.
- Confirm correct COM port in PlatformIO.
- Press BOOT during upload if required by the board.
- Disconnect or isolate RS485 bus if the UART0 path is held busy.

For RS485 sensor read failures:

- Switch external mode to RS485.
- Confirm A-to-A and B-to-B wiring.
- Confirm shared GND.
- Confirm sensor power.
- Confirm baud rate 9600 8N1.
- Confirm slave ID/address.
- Add delay between Modbus devices on the same bus.

For PZEM-specific issues:

- Confirm PZEM has AC supply in its supported range.
- Confirm CT wiring and load.
- Check `isSimulationMode()`; current header falls back to simulation when real hardware does not respond.
- Use `resetEnergy()` only when intentionally clearing accumulated energy.

For input/relay issues:

- Remember active-low behavior.
- On GPIO 34/35/36/39, add external resistors.
- Call `update()` every loop for debounce logic.
- Edge methods are true only for one loop cycle.

For DS18B20 issues:

- Confirm DATA is connected to GPIO14.
- Confirm 4.7k pull-up from DATA to 3V3.
- Confirm the sensor has correct power and ground.
- Check for `DEVICE_DISCONNECTED_C` before trusting temperature values.

## Future Expansion Ideas

- Add a full PlatformIO project layout with `include/`, `src/`, and `platformio.ini`.
- Add example sketches for XY-MD03-only, PZEM-only, OLED display, and relay/switch board.
- Add a unified `main.cpp` demo that can switch between XY-MD03 and PZEM modes.
- Add a DS18B20 helper class if the board needs filtering, alarms, or multiple sensor addresses.
- Add MQTT publishing using `toJSON()` from `DevPZEM`.
- Add NVS logging for PZEM energy snapshots.
- Add a Modbus bus manager if multiple sensors share UART0.
