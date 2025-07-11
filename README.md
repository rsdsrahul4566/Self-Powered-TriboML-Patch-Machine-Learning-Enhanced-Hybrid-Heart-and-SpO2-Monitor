# 💓 Self-Powered TriboML Patch
### 🛠️ ELCIA Sense2Scale Hackathon — Team Ignis Vision

> **A Machine-Learning-Enhanced Hybrid Heart and SpO₂ Monitor for Remote Patient Monitoring**

---

## 🔍 Overview

The **Self-Powered TriboML Patch** is an ultra-thin, skin-conformable adhesive patch designed to monitor **heart rate**, **SpO₂**, and **skin temperature** continuously — all without batteries.

Combining **triboelectric nanogenerator (TENG)** technology with **PPG (MAX30102)** and **IR temperature (MLX90614)** sensors, and powered by **machine learning (CNN + LSTM + Random Forest)**, this wearable device delivers medical-grade accuracy in even motion-heavy, real-life conditions.

---

## 🎯 Target Application Area

- Elderly care monitoring
- Post-COVID recovery
- Ambulatory & chronic illness home care
- Continuous, batteryless remote vitals tracking

---

## ⚙️ Key Features

- ⚡ **Batteryless** TENG-based heart rate monitoring
- 🔁 **Dual-layer ML pipeline**:  
   - 1D-CNN for signal quality filtering  
   - LSTM for peak detection  
   - Random Forest for SpO₂ correction in motion
- 📶 **BLE/Wi-Fi connectivity** for real-time data streaming
- 📊 **Live dashboards** with alerts, trends, and analytics
- 🔒 **Secure encrypted transmission** to companion app

---

## 🧠 ML Architecture

| Layer        | Purpose                          |
|--------------|----------------------------------|
| 🧠 1D-CNN     | Denoise and classify signal quality |
| 🧠 LSTM       | Extract reliable heart rate peaks |
| 🌲 Random Forest | Enhance SpO₂ accuracy in motion   |

---

## 🔬 Sensors & Components

| Sensor/Module    | Type                     | Function                          | Cost (INR) |
|------------------|--------------------------|-----------------------------------|------------|
| TENG Film        | Triboelectric Generator  | Self-powered heartbeat sensing    | ₹550       |
| MAX30102         | PPG + SpO₂ Module        | Backup HR and SpO₂ estimation     | ₹999       |
| MLX90614         | IR Temperature Sensor    | Skin temperature measurement      | ₹800       |
| Supercapacitor   | Energy Storage           | Conditioning & storage            | ₹400       |

> 📦 **Total sensor cost:** ₹2,749 (within ₹4,000 constraint)

---

## 🔌 Circuit Pinout (ESP32)

| Sensor         | ESP32 Pin     | Function                        |
|----------------|---------------|----------------------------------|
| TENG Output    | GPIO36        | Analog peak voltage detection   |
| MAX30102 (SDA) | GPIO21        | I²C Data                        |
| MAX30102 (SCL) | GPIO22        | I²C Clock                       |
| MLX90614 (SDA) | GPIO21        | I²C Data                        |
| MLX90614 (SCL) | GPIO22        | I²C Clock                       |
| Supercap Sense | GPIO39        | ADC monitoring                  |

---

## 🧪 Testing Plan (TRL-8 Ready)

- ✅ **<1% HR error** vs. ECG on 20 subjects (50–120 BPM)
- ✅ **±2% SpO₂ accuracy** vs. clinical oximeter (static & motion)
- ✅ **±0.5°C IR accuracy** vs. NIST thermometer (30–40°C)
- ✅ **48h wear trials**: ≥98% data uptime, <1% signal loss
- ✅ **Environmental robustness**: −10 to +40°C, 20–90% RH
- ✅ **ML Validation**:  
   - CNN signal classifier ≥97% sensitivity/specificity  
   - LSTM HR MAE < 0.5 BPM

---

## 🇮🇳 Indian Sensor Substitution Plan

| Component   | Substitution Strategy |
|-------------|------------------------|
| TENG Film   | Local PTFE composite, retrain ML models |
| MAX30102    | Indian I²C-compatible PPG chips |
| MLX90614    | Local IR sensor with I²C support |

---

## 📈 Expected Output

- 📊 Real-time vitals dashboard: HR (50–120 BPM), SpO₂ (70–100%), Temp (30–40°C)
- 🚨 Alerts for bradycardia, hypoxia, fever, detachment
- 🧾 Timestamped data logs (CSV): TENG, PPG, Temp
- 📈 ML analytics: heartbeat peaks, quality flags, variability trends
- 🩺 Exportable clinician summary reports

---

## 👨‍💻 Team Ignis Vision

| Name         | Role       | Email                           |
|--------------|------------|----------------------------------|
| Rahul Kumar  | Team Lead  | rahul.kumar791@ptuniv.edu.in     |
| Suwathi J    | Member     | suwathi.j881@ptuniv.edu.in       |

📞 Contact: +91 7878260266  
🏫 Institute: Puducherry Technological University

---

## 📜 Declaration

We comply with cost, TRL-8, and data-sharing requirements as per ELCIA Sense2Scale guidelines.

> *This patch is the first-of-its-kind fusion of energy harvesting, real-time vitals monitoring, and embedded AI — made for the future of wearable healthcare.*

---
