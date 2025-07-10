# 🛠️Circuit-Diagram

```bash


                    ┌─────────────────────────────────────┐
                    │           ESP32-WROOM-32            │
                    │                                     │
    ┌──────────────►│ 3V3                    GPIO36 (A0)  │◄───── ┐
    │               │ GND                    GPIO39 (A3)  │       │
    │               │ GPIO21 (SDA)           GPIO34 (A6)  │       │
    │   ┌──────────►│ GPIO22 (SCL)           GPIO35 (A7)  │       │
    │   │           │ GPIO19 (MISO)          GPIO32 (A4)  │       │
    │   │           │ GPIO23 (MOSI)          GPIO33 (A5)  │       │
    │   │           │ GPIO18 (SCK)           GPIO25 (A1)  │       │
    │   │           │ GPIO5 (SS)             GPIO26 (A2)  │       │
    │   │           │ GPIO17 (TX2)           GPIO27       │       │
    │   │           │ GPIO16 (RX2)           GPIO14       │       │
    │   │           │ GPIO4                  GPIO12       │       │
    │   │           │ GPIO0                  GPIO13       │       │
    │   │           │ GPIO2 (LED)            GPIO15       │       │
    │   │           │ GND                    GND          │       │
    │   │           └─────────────────────────────────────┘       │
    │   │                                                         │
    │   │           ┌─────────────────┐    ┌─────────────────┐    │
    │   │           │   MAX30102      │    │   MLX90614      │    │
    │   │           │   PPG Sensor    │    │ IR Temp Sensor  │    │
    │   │           │                 │    │                 │    │
    │   └──────────►│ SCL             │    │ SCL             │◄───┤
    │               │ SDA             │    │ SDA             │◄───┼─┐
    └──────────────►│ VIN (3.3V)      │    │ VCC (3.3V)      │◄───┤ │
                    │ GND             │    │ GND             │    │ │
                    │ INT (Optional)  │    │                 │    │ │
                    └─────────────────┘    └─────────────────┘    │ │
                                                                  │ │
    ┌─────────────────────────────────────────────────────────────┘ │
    │                                                               │
    │   ┌─────────────────┐     ┌───────────────┐                   │
    │   │      TENG       │     │ Voltage       │                   │
    │   │  Triboelectric  │────►│ Divider       │────────────────── ┘
    │   │  Nanogenerator  │     │ R1: 10kΩ      │
    │   │                 │     │ R2: 10kΩ      │
    │   │ (+) Output      │     │               │
    │   │ (-) Ground      │────►│ GND           │
    │   └─────────────────┘     └───────────────┘
    │
    │   ┌────────────────────────────────────────┐
    │   │           Power Management             │
    │   │                                        │
    │   │  ┌──────────┐    ┌─────────────────┐   │
    │   │  │Li-Ion    │    │ LDO Regulator   │   │
    │   │  │Battery   │───►│ (3.3V Output)   │───┼─────────────────┐
    │   │  │3.7V      │    │                 │   │                 │
    │   │  │2000mAh   │    │ Enable Pin      │   │                 │
    │   │  │          │    │ GND             │   │                 │
    │   │  └────┬─────┘    └─────────────────┘   │                 │
    │   │       │                                │                 │
    │   │       │          ┌─────────────────┐   │                 │
    │   │       └─────────►│ Schottky Diode  │───┼─────────────────┤
    │   │                  │ (Reverse        │   │                 │
    │   │                  │ Protection)     │   │                 │
    │   │                  └─────────────────┘   │                 │
    │   └────────────────────────────────────────┘                 │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
                                    │
                                   GND (Common Ground)

```


# 🛠️Pin-Diagram

## MAX30102 Sensor Connections
### MAX30102 Pinout:

VIN: Power input (3.3V or 5V)

GND: Ground

SCL: I2C Clock line

SDA: I2C Data line

INT: Interrupt pin (optional)

IRD: Infrared data pin (not used)

RD: Red LED data pin (not used)

### Connection to ESP32:

```bash
MAX30102    →    ESP32
VIN         →    3.3V
GND         →    GND
SCL         →    GPIO 22 (SCL)
SDA         →    GPIO 21 (SDA)
INT         →    Not connected
IRD         →    Not connected
RD          →    Not connected
```

## MLX90614 Temperature Sensor Connections
### MLX90614 Pinout:

VCC: Power supply (3.3V)

GND: Ground

SCL: I2C Clock line

SDA: I2C Data line

### Connection to ESP32:
```bash
MLX90614    →    ESP32
VCC         →    3.3V
GND         →    GND
SCL         →    GPIO 22 (SCL)
SDA         →    GPIO 21 (SDA)
```

## TENG Sensor Connection
### TENG Sensor Setup:

Output: Analog voltage signal

Range: 0-3.3V

Connection: Direct to ADC pin

### Connection to ESP32:
```bash
TENG Sensor →    ESP32
Output      →    GPIO 36 (ADC1_CH0)
GND         →    GND
```

## LED Status Indicator
### Connection:
```bash
LED Assembly    →    ESP32
Anode (+)       →    GPIO 2 (through 220Ω resistor)
Cathode (-)     →    GND
```

## Buzzer Alert System
### Connection:
```bash
Buzzer      →    ESP32
Positive    →    GPIO 4
Negative    →    GND
```

# 🛠️Detailed Circuit Diagram
## Power Distribution
```bash
ESP32 3.3V Output
    ├── MAX30102 VIN
    ├── MLX90614 VCC
    └── Pull-up resistors (4.7kΩ each on SDA/SCL)

ESP32 GND
    ├── MAX30102 GND
    ├── MLX90614 GND
    ├── TENG GND
    ├── LED Cathode
    └── Buzzer Negative
```

## I2C Bus Configuration
```bash
GPIO 21 (SDA) ──┬── MAX30102 SDA
                └── MLX90614 SDA
                    (with 4.7kΩ pull-up to 3.3V)

GPIO 22 (SCL) ──┬── MAX30102 SCL
                └── MLX90614 SCL
                    (with 4.7kΩ pull-up to 3.3V)
```

## Analog Input Setup
```bash
GPIO 2 ──── 220Ω Resistor ──── LED Anode
                                LED Cathode ──── GND

GPIO 4 ──── Buzzer Positive
            Buzzer Negative ──── GND
```
## Digital Output Configuration
```bash
GPIO 2 ──── 220Ω Resistor ──── LED Anode
                                LED Cathode ──── GND

GPIO 4 ──── Buzzer Positive
            Buzzer Negative ──── GND
```
