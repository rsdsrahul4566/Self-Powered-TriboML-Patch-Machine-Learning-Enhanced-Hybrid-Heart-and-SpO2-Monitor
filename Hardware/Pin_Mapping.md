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

## Battery and Regulation
```bash
Li-Ion Battery (3.7V, 2000mAh)
│
├── Schottky Diode (BAT54C) ── Reverse Protection
│
├── LDO Regulator (AMS1117-3.3) ── 3.3V Output to ESP32 and Sensors
│
└── Voltage Divider ── Battery Level Monitor (GPIO39)
    ├── R1: 100kΩ
    └── R2: 47kΩ ── GND

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
