# 💡 Structured Workstation Light Controller

[![Platform: ESP32](https://img.shields.io/badge/Platform-ESP32%20WROOM-3297CD.svg)](https://www.espressif.com/)
[![Framework: Arduino C++](https://img.shields.io/badge/Framework-Arduino%20C%2B%2B-00979D.svg)](https://www.arduino.cc/)
[![Status: Completed](https://img.shields.io/badge/Status-Completed-success.svg)]()

> A modular, safety-gated dual-workstation luminaire controller built on the ESP-WROOM-32 platform. Implements structured input-processing-output architecture, analog scaling via PWM duty cycle mapping, and momentary hold-to-enable actuation.

---

## 📌 Project Overview & Background

Modern embedded user interfaces frequently demand **contingent control**—ensuring that dynamic adjustments (such as lighting levels, motor speeds, or heating elements) are only enacted when a primary supervisor/enable condition is satisfied.

The **Structured Workstation Light** is an embedded systems engineering prototype developed to address this design requirement. Using an **ESP32 DevKit (ESP-WROOM-32)**, the system manages a dual-LED task luminaire with decoupled software modules:
- **Active Hold-to-Enable:** Brightness control is gated by an active-low momentary pushbutton. While released, lighting outputs remain completely off regardless of analog dial adjustments.
- **Dynamic PWM Regulation:** When held, the user can adjust luminance across a continuous spectrum via an analog potentiometer wiper.
- **Modular Software Architecture:** Strictly separates **Input Sensing**, **Numeric Processing / Math Scaling**, and **Actuation / Output** into distinct functions, adhering to clean embedded C/C++ design standards.

---

## 🛠️ Hardware Components & Pin Mapping

### Bill of Materials (BOM)

| Component | Specification / Value | Quantity | Role / Description |
| :--- | :--- | :--- | :--- |
| **Microcontroller** | ESP-WROOM-32 (38-pin DevKit, USB-C) | 1 | Main system logic & PWM driver |
| **Potentiometer** | $10\text{ k}\Omega$ Linear Rotary Potentiometer | 1 | Manual analog voltage divider / control knob |
| **Pushbutton** | Momentary Tactile Switch (SPST) | 1 | Active-low enable switch (Internal / External Pull-up) |
| **LED 1 (Workstation)** | High-brightness Blue LED | 1 | Primary workstation indicator |
| **LED 2 (Ambient)** | Diffused Warm Yellow LED | 1 | Secondary ambient indicator |
| **Current-Limiting Resistors** | $220\,\Omega,\ 1/4\text{W}$ ($\pm5\%$) | 2 | Anode current limitation for LEDs |
| **Prototyping** | Breadboard & M-M Jumper Wires | — | Modular circuit interconnection |

---

### Detailed Pin Configuration

Based on the validated circuit schematic:

| Peripheral Pin | ESP32 GPIO | Direction | Mode / Characteristics |
| :--- | :--- | :--- | :--- |
| **Pushbutton Input** | `GPIO 23` | Digital Input | Pulled to `GND` on press (`INPUT_PULLUP`) |
| **Potentiometer Wiper (Pin 2)** | `GPIO 34` | Analog Input | ADC1 channel (`0`–$4095$ count, $12$-bit resolution) |
| **Potentiometer VCC (Pin 3)** | `3V3` | Power Out | High rail ($+3.3\text{V}$) |
| **Potentiometer GND (Pin 1)** | `GND` | Ground | System common rail |
| **Blue LED (Anode via $220\Omega$)** | `GPIO 19` | Output | Hardware LEDC PWM Channel |
| **Yellow LED (Anode via $220\Omega$)** | `GPIO 18` | Output | Hardware LEDC PWM Channel |
| **LED Cathodes** | `GND` | Return | Connected to ESP32 ground bus |

---

### Circuit Operation Notes
- **ADC Input (`GPIO 34`):** An input-only pin without internal pull-ups, ideal for pure analog readings. Reads raw voltages from $0\text{ V}$ to $3.3\text{ V}$ mapped to $[0, 4095]$.
- **Active-Low Pushbutton (`GPIO 23`):** Configured with `INPUT_PULLUP`. Resting state is `HIGH` (`true`); pressed state is `LOW` (`false`).
- **Dual Outputs (`GPIO 18`, `GPIO 19`):** Controlled synchronously via ESP32's LEDC hardware timer.

---

## 🚀 Setup & Flashing Instructions

### Prerequisites
1. **Arduino IDE 2.x** or **VS Code with PlatformIO** installed.
2. ESP32 board package installed (`esp32` by Espressif Systems).
3. Silicon Labs CP210x or CH340 USB-to-UART driver (depending on DevKit board revision).

### Installation Steps

1. **Open Project:**
   - In Arduino IDE, open `workstation_light.ino`.
2. **Board Selection:**
   - Select **Tools > Board > ESP32 Arduino > ESP32 Dev Module** (or `DOIT ESP32 DEVKIT V1`).
   - Select the appropriate USB Serial Port (**Tools > Port**).
3. **Flash Firmware:**
   - Click the **Upload** button ($\rightarrow$).
   - (If needed, hold down the `BOOT` button on your ESP32 board during flashing until write progression begins).
4. **Monitor Serial Output:**
   - Open Serial Monitor at **115200 baud** to view real-time state logs.

---
