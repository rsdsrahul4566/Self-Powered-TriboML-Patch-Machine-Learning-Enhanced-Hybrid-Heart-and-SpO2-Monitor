# 💡 Self-Powered TriboML Patch   
     (DEMO VIDEO URL MENTIONED BELOW)
> 🚑 *Machine Learning–Enhanced Batteryless Wearable for Real-Time Heart Rate, SpO₂ & Skin Temperature Monitoring*

---

## 🚀 Project Summary

🔬 The **Self-Powered TriboML Patch** is a next-gen **batteryless wearable health monitor** combining energy-harvesting sensors with embedded AI. Designed for **remote patient monitoring**, it offers continuous, accurate, and real-time tracking of **heart rate**, **blood oxygen (SpO₂)**, and **skin temperature** — ideal for elderly care, post-COVID recovery, and chronic disease management.

Built for the **ELCIA Sense2Scale Hackathon**, this device integrates a **triboelectric nanogenerator (TENG)**, **PPG sensor (MAX30102)**, **IR temperature sensor (MLX90614)**, and a **lightweight ML pipeline** on an ESP32 board.

---

## 🎯 Target Application Areas

- 👵 Elderly and home-based healthcare
- 🫁 Post-COVID recovery support
- 🏃 Ambulatory chronic care monitoring
- 💓 Continuous real-time vitals tracking
- 🔋 Batteryless and maintenance-free health patch

---

## ⚙️ Core Features

| ⚡ Feature | 🔍 Description |
|-----------|----------------|
| 🔋 **Batteryless Operation** | TENG-based self-powered sensing for heartbeats |
| 🧠 **Dual ML Architecture** | CNN + LSTM + Random Forest for robust signal processing |
| 📡 **Wireless Communication** | BLE / Wi-Fi streaming to mobile dashboards |
| 📊 **Live Visualization** | Real-time vitals, signal quality indicators, and trend graphs |
| 🚨 **Smart Alerts** | Configurable triggers for bradycardia, hypoxia, fever, etc. |
| 🧾 **Clinician-Ready Logs** | Time-stamped data exports and summary reports |

---

## 🧠 Machine Learning Pipeline

| 🧠 ML Layer | 📋 Function |
|------------|-------------|
| 🔹 1D-CNN | Filters noise and assesses signal quality of raw TENG/PPG streams |
| 🔸 LSTM | Extracts accurate heart-rate peaks from filtered signals |
| 🌲 Random Forest | Enhances SpO₂ accuracy under movement-induced noise |

> ML runs on-device on the ESP32 for fast, offline inference — no cloud needed! 🚫☁️

---

## 🧩 Hardware Components

### 🧪 Sensors & Modules

| 🔧 Component | 💡 Description | 💰 Cost (INR) |
|-------------|----------------|--------------|
| TENG Film | Self-powered triboelectric heartbeat sensor | ₹200 |
| MAX30102 | PPG + SpO₂ sensor module | ₹350 |
| MLX90614 | IR temperature sensor | ₹1350 |
| Li-Battery + Rectifier | Power management unit | ₹400 |

> 💸 **Total Estimated Cost**: ₹2,749 — well within ₹4,000 constraint ✅

---

## 🔌 ESP32 Pin Connections

| 📦 Module | 📍 ESP32 Pin | 🔧 Function |
|----------|-------------|-------------|
| TENG Film | GPIO36 (ADC1_0) | Pulse voltage detection |
| MAX30102 (SDA) | GPIO21 | I²C Data Line |
| MAX30102 (SCL) | GPIO22 | I²C Clock Line |
| MLX90614 (SDA) | GPIO21 | Shared I²C Data |
| MLX90614 (SCL) | GPIO22 | Shared I²C Clock |
| Supercap Monitor | GPIO39 (ADC1_3) | Voltage sensing |

---

## 🧪 Testing & Validation Plan (TRL-8 Ready)

📐 **Accuracy Targets:**

- ✅ **Heart Rate**: <1% error vs. ECG (50–120 BPM)
- ✅ **SpO₂**: ±2% vs. clinical oximeter (70–100%) under motion & rest
- ✅ **Temperature**: ±0.5 °C vs. calibrated NIST thermometer (30–40 °C)

🧪 **Robustness Tests:**

- 🔋 48h continuous wear trial (≥98% uptime)
- 🌡️ Operates in −10 °C to +40 °C, 20–90% RH
- 📶 Over-the-air (OTA) firmware updates
- 📊 ML Accuracy:  
  - CNN signal classifier ≥97%  
  - LSTM HR MAE < 0.5 BPM

---

## 🇮🇳 Indian Sensor Substitution Plan

| 🧩 Imported | 🇮🇳 Substitute | 🔄 Strategy |
|------------|----------------|------------|
| TENG Film | Local PTFE-based tribo films | Retrain ML model on local data |
| MAX30102 | Indian PPG sensor with I²C | Retrain CNN + Random Forest |
| MLX90614 | Local IR temperature module | Plug-and-play via I²C firmware APIs |

---

## 📲 Expected Outputs

- 📈 **Real-Time Dashboard**  
  - Heart Rate (50–120 BPM)  
  - SpO₂ (70–100%)  
  - Skin Temp (30–40 °C)  
  - Signal Quality Indicators

- 🚨 **Alerts & Notifications**  
  - Bradycardia  
  - Hypoxia  
  - Fever  
  - Patch detachment  

- 🧾 **Data Logging & Analytics**  
  - CSV logs with timestamps  
  - Annotated ML outputs  
  - Trend reports for clinicians  

---

## 🧑‍💻 Team Ignis Vision

| 👤 Name | 🧑‍💼 Role | 📧 Email |
|--------|-----------|-----------|
| Rahul Kumar | Team Lead | rahul.kumar791@ptuniv.edu.in |
| Suwathi J | Team Member | suwathi.j881@ptuniv.edu.in |

📞 Contact: +91 7878260266  
🏛️ Institute: Puducherry Technological University

---

## ✅ Declaration

We understand and agree to comply with the **cost**, **TRL‑8 standards**, and **data-sharing policies** set by the ELCIA Sense2Scale Hackathon.

> 🧠 This patch is India’s first hybrid wearable that combines **self-powered sensing** with **edge AI** for reliable vitals monitoring — optimized for chronic care, and built for scale.

---

## 📸 Screenshots / Demo Video URL

```bash






https://drive.google.com/file/d/1Mj_FknmQuOdrHJPVEiVNY1OkJiJjHbp6/view?usp=drive_link







```



![elcia1](https://github.com/user-attachments/assets/cad33acb-d950-4534-9c69-a4a960ebb460)

![elcia2](https://github.com/user-attachments/assets/87e512ff-2092-416d-93a2-906268e6b194)

![elcia3](https://github.com/user-attachments/assets/17a8adfe-fb8f-408c-8064-692b86ba5483)

![elcia5](https://github.com/user-attachments/assets/d04e0ae7-74a0-48ca-b013-42437ad2ec25)


---

## 📜 License

This project is for educational and demonstration purposes under the ELCIA Hackathon. Licensing terms may be updated post evaluation.

---

## ⭐ Support This Project

If you like this project, consider ⭐ staring and selecting for the finale! Your feedback and contributions are welcome.

---

🧠 *"Inspiring intelligent healthcare through self-powered innovation."*  
— Team Ignis Vision 💡
