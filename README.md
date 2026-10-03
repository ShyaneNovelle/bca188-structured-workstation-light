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

## ⚡ System Schematic & Architecture

```
                 +-------------------+
                 |    ESP-WROOM-32   |
                 |                   |
[Pushbutton] ----| D23 (Active-Low)  |
(to GND)         |                   |
                 |                   |
[Potentiometer] -| D34 (ADC1_CH6)    |---- LEDC PWM (D19) ---> [220Ω] ---> (Blue LED)   ---> GND
(0V - 3.3V)      |                   |
                 |                   |---- LEDC PWM (D18) ---> [220Ω] ---> (Yellow LED) ---> GND
                 +-------------------+
```

### Circuit Operation Notes
- **ADC Input (`GPIO 34`):** An input-only pin without internal pull-ups, ideal for pure analog readings. Reads raw voltages from $0\text{ V}$ to $3.3\text{ V}$ mapped to $[0, 4095]$.
- **Active-Low Pushbutton (`GPIO 23`):** Configured with `INPUT_PULLUP`. Resting state is `HIGH` (`true`); pressed state is `LOW` (`false`).
- **Dual Outputs (`GPIO 18`, `GPIO 19`):** Controlled synchronously via ESP32's LEDC hardware timer.

---

## 🧠 System Logic & Code Implementation

The project avoids unstructured monolithic loops in favor of a clean **Input-Processing-Output (IPO)** design pattern.

### Architectural Breakdown

```
       +-----------------------+
       |   1. readInputs()     | -> Gathers Button state & ADC numeric values
       +-----------------------+
                   |
                   v
       +-----------------------+
       | 2. processLogic()     | -> Checks gating logic; invokes scaleToDuty()
       +-----------------------+
                   |
                   v
       +-----------------------+
       |  3. writeOutputs()    | -> Drives PWM duty cycle to LEDC channels
       +-----------------------+
```

### Core Mathematical Scaling Function
ESP32's ADC operates at **12-bit** resolution ($0 \to 4095$), while the LEDC PWM timer operates typically at **8-bit** resolution ($0 \to 255$):

$$\text{Duty Cycle} = \left\lfloor \frac{\text{rawADC}}{4095} \times 255 \right\rfloor$$

```cpp
int scaleToDuty(int raw) {
  // Constrain raw input to valid 12-bit bounds
  int clamped = constrain(raw, 0, 4095);
  // Linear scale mapping from 12-bit ADC to 8-bit PWM duty
  return map(clamped, 0, 4095, 0, 255);
}
```

### Sketch Source Code (`workstation_light.ino`)

```cpp
/**
 * Project: Structured Workstation Light Controller
 * Description: Modular dual-LED brightness control gated by a momentary enable button.
 */

// ================= CONSTANTS & PIN DEFINITIONS =================
const int PIN_BUTTON = 23;  // Active-low pushbutton
const int PIN_POT    = 34;  // Analog input (Potentiometer wiper)
const int PIN_LED_BLUE   = 19;  // Primary task light
const int PIN_LED_YELLOW = 18;  // Secondary ambient light

// PWM Configuration (ESP32 LEDC)
const int PWM_FREQ       = 5000; // 5 kHz PWM frequency
const int PWM_RESOLUTION = 8;    // 8-bit resolution (0 - 255)
const int PWM_CHANNEL_1  = 0;
const int PWM_CHANNEL_2  = 1;

// ================= GLOBAL STATE VARIABLES =================
int rawPotValue       = 0;       // Numeric reading from ADC (0 - 4095)
bool isEnabled        = false;   // System enable state
int computedDutyCycle = 0;       // Scaled PWM duty cycle (0 - 255)

// ================= FUNCTION PROTOTYPES =================
void readInputs();
void processLogic();
void writeOutputs();
int scaleToDuty(int raw);

// ================= SETUP =================
void setup() {
  Serial.begin(115200);

  // Input Configuration
  pinMode(PIN_BUTTON, INPUT_PULLUP);
  pinMode(PIN_POT, INPUT);

  // Configure ESP32 LEDC PWM channels
  ledcAttach(PIN_LED_BLUE, PWM_FREQ, PWM_RESOLUTION);
  ledcAttach(PIN_LED_YELLOW, PWM_FREQ, PWM_RESOLUTION);

  Serial.println("[SYSTEM READY] Structured Workstation Light initialized.");
}

// ================= MAIN LOOP (IPO PATTERN) =================
void loop() {
  readInputs();     // Phase 1: Input Sensing
  processLogic();   // Phase 2: Processing & Arithmetic
  writeOutputs();   // Phase 3: Actuation
  delay(10);        // Modest loop pacing for stable sampling
}

// ================= MODULE IMPLEMENTATIONS =================

/**
 * Reads sensor and button hardware states
 */
void readInputs() {
  rawPotValue = analogRead(PIN_POT);
  // Button active-low: reading LOW means pressed/enabled
  isEnabled = (digitalRead(PIN_BUTTON) == LOW);
}

/**
 * Handles logic gating and duty cycle arithmetic
 */
void processLogic() {
  if (isEnabled) {
    computedDutyCycle = scaleToDuty(rawPotValue);
  } else {
    computedDutyCycle = 0; // Forced safety cutoff when released
  }
}

/**
 * Writes computed PWM states to physical pins
 */
void writeOutputs() {
  ledcWrite(PIN_LED_BLUE, computedDutyCycle);
  ledcWrite(PIN_LED_YELLOW, computedDutyCycle);
}

/**
 * Calculates 8-bit PWM duty cycle from raw 12-bit ADC reading
 */
int scaleToDuty(int raw) {
  int safeRaw = constrain(raw, 0, 4095);
  return map(safeRaw, 0, 4095, 0, 255);
}
```

---

## 📊 Verification & Test Results

The system was evaluated against baseline functional expectations under the specified operational states:

| Test Case | Button State | Potentiometer Position | ADC Raw Reading | Expected Duty Cycle | Observed Behavior | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Boot Idle** | Released (`HIGH`) | Arbitrary | `2048` (Mid) | `0` | Both LEDs completely OFF | **PASS** |
| **Knob Twist (Idle)**| Released (`HIGH`) | $0 \to 100\%$ sweep | Any (`0 - 4095`) | `0` | Both LEDs remain OFF throughout | **PASS** |
| **Min Range (Active)**| Held (`LOW`) | Full Counter-Clockwise | $\approx 0 - 50$ | `0` | LEDs off / min threshold | **PASS** |
| **Mid Range (Active)**| Held (`LOW`) | 12 o'clock ($50\%$) | $\approx 2048$ | `127 - 128` | LEDs illuminate at half brightness | **PASS** |
| **Max Range (Active)**| Held (`LOW`) | Full Clockwise ($100\%$)| $\approx 4095$ | `255` | LEDs illuminate at max brightness | **PASS** |
| **Immediate Release** | Released mid-dial | Any ($>0$) | Current value | Drops to `0` | Immediate shutdown on switch release | **PASS** |

---

## 🚀 Setup & Flashing Instructions

### Prerequisites
1. **Arduino IDE 2.x** or **VS Code with PlatformIO** installed.
2. ESP32 board package installed (`esp32` by Espressif Systems).
3. Silicon Labs CP210x or CH340 USB-to-UART driver (depending on DevKit board revision).

### Installation Steps
1. **Clone the Repository:**
   ```bash
   git clone https://github.com/<your-username>/structured-workstation-light.git
   cd structured-workstation-light
   ```
2. **Open Project:**
   - In Arduino IDE, open `workstation_light.ino`.
3. **Board Selection:**
   - Select **Tools > Board > ESP32 Arduino > ESP32 Dev Module** (or `DOIT ESP32 DEVKIT V1`).
   - Select the appropriate USB Serial Port (**Tools > Port**).
4. **Flash Firmware:**
   - Click the **Upload** button ($\rightarrow$).
   - (If needed, hold down the `BOOT` button on your ESP32 board during flashing until write progression begins).
5. **Monitor Serial Output:**
   - Open Serial Monitor at **115200 baud** to view real-time state logs.

---

## 👨‍💻 Author & Acknowledgments

- **Developed as part of:** Laboratory Activity 5: *Structured Workstation Light*
- **Target Microcontroller:** Espressif ESP32-WROOM-32
- **Design Pattern:** IPO (Input-Processing-Output) Architecture
