
https://github.com/user-attachments/assets/870a788a-4940-426f-b520-d26b13783ed8
# Laboratory Activity 5: Structured Workstation Light

**Name:Shyane Novelle R. Canayan
**Course: Programming For Internet of Things
**Board:** ESP32 (ESP-WROOM) | **Language:** Arduino C++ 

---

## 1. Objective

Build a small "workstation light" with an ESP32. The light only works while a push button is **held down**. While the button is held, a potentiometer (knob) controls how bright an LED is. When the button is **released**, everything turns off, even if the knob is turned.

The code must follow three clear parts: **input**, **processing**, and **output**, each in its own function.

---

## 2. Materials

| Item | Quantity |
|---|---|
| ESP32 development board (ESP-WROOM, USB-C) | 1 |
| Breadboard and jumper wires | 1 set |
| Push button | 1 |
| Potentiometer (knob) | 1 |
| LED (status, blue) | 1 |
| LED (brightness, yellow) | 1 |
| 220 Ω resistor | 2 (one per LED) |

---

## 3. Circuit Diagram and Wiring
<img width="3060" height="2330" alt="0a8d3139-31ef-42db-83b7-83b53fe41056" src="https://github.com/user-attachments/assets/edb71537-cc29-4008-98f3-cebe871d6bff" />


### Wiring table

| Part | ESP32 Pin | Other side | Type |
|---|---|---|---|
| Push button | GPIO 23 | GND | **Input** |
| Potentiometer, middle pin (pin 2) | GPIO 34 | n/a | **Input** |
| Potentiometer, pin 3 | 3V3 | n/a | Power |
| Potentiometer, pin 1 | GND | n/a | Ground |
| Status LED + 220 Ω resistor | GPIO 18 | GND | **Output** |
| Brightness (PWM) LED + 220 Ω resistor | GPIO 19 | GND | **Output** (PWM) |

### Inputs and outputs

**Inputs**
- **Push button (GPIO 23):** tells the program if the light is allowed to work. The button connects to GND when pressed, so the pin reads LOW when pressed.
- **Potentiometer (GPIO 34):** gives a voltage from 0 V to 3.3 V that sets the brightness.

**Outputs**
- **Status LED (GPIO 18):** ON means "the light is enabled" (button is held).
- **Brightness LED (GPIO 19):** its brightness follows the knob, using PWM.

> **What is PWM?** PWM (Pulse Width Modulation) turns the LED on and off very fast (5000 times per second). A longer "on" time looks brighter, a shorter "on" time looks dimmer.

---

## 4. Program Design

The program is split into three functions plus the helper `scaleToDuty()`, as required.

| Function | Job | What it does |
|---|---|---|
| `readInputs()` | **Input** | Reads the button and the knob |
| `processInputs()` | **Processing** | Decides what brightness to use |
| `updateOutputs()` | **Output** | Sets the two LEDs |
| `scaleToDuty(int raw)` | Helper | Converts the knob reading (0–4095) to brightness (0–255) |

The `loop()` just calls them in order, then waits 20 ms:

```cpp
void loop() {
  readInputs();
  processInputs();
  updateOutputs();
  delay(20);
}
```

### Scaling function (Requirement 3)

The knob is read with a 12-bit ADC (0 to 4095). The PWM uses 8 bits (0 to 255). The function below changes one range into the other:

```cpp
int scaleToDuty(int raw) {
  return constrain(map(raw, 0, 4095, 0, 255), 0, 255);
}
```

`map()` does the conversion and `constrain()` makes sure the result never goes outside 0 to 255.

---

## 5. How Constants, Boolean State, and Numeric Readings Are Used (Requirement 2)

### Constants
Constants are values that never change. They are declared with `const`, so the pin numbers are written only once and are easy to change later.

```cpp
const uint8_t BUTTON_PIN     = 23;
const uint8_t POT_PIN        = 34;
const uint8_t STATUS_LED_PIN = 18;
const uint8_t PWM_LED_PIN    = 19;
```

### Boolean state (true / false)
Booleans store a yes/no answer.

| Variable | Meaning |
|---|---|
| `buttonPressed` | `true` while the button is held (pin reads LOW) |
| `pwmReady` | `true` if the PWM setup worked, so the program does not use a broken PWM |

The light is allowed to turn on only when **both** are true:
`buttonPressed && pwmReady`

### Numeric readings
Numbers store values that change while the program runs.

| Variable | Meaning | Range |
|---|---|---|
| `rawInput` | Raw knob reading from the ADC | 0 – 4095 |
| `requestedDuty` | Brightness the knob is asking for | 0 – 255 |
| `appliedDuty` | Brightness actually sent to the LED | 0 – 255 |

`requestedDuty` is what the knob wants. `appliedDuty` is what the LED gets. If the button is released, `appliedDuty` is forced to 0, so turning the knob does nothing. This is the key idea of the lab.

```cpp
if (buttonPressed && pwmReady) {
  appliedDuty = requestedDuty;
} else {
  appliedDuty = 0;
}
```

---

## 6. Full Source Code

```cpp
#include <Arduino.h>

const uint8_t BUTTON_PIN = 23;
const uint8_t POT_PIN = 34;
const uint8_t STATUS_LED_PIN = 18;
const uint8_t PWM_LED_PIN = 19;

bool buttonPressed = false;
bool pwmReady = false;
int rawInput = 0;
int requestedDuty = 0;
int appliedDuty = 0;

void readInputs();
void processInputs();
void updateOutputs();
int scaleToDuty(int raw);

void setup() {
  Serial.begin(115200);

  pinMode(BUTTON_PIN, INPUT_PULLUP);

  pinMode(STATUS_LED_PIN, OUTPUT);
  digitalWrite(STATUS_LED_PIN, LOW);

  pinMode(PWM_LED_PIN, OUTPUT);
  digitalWrite(PWM_LED_PIN, LOW);

  analogReadResolution(12);
  analogSetPinAttenuation(POT_PIN, ADC_11db);

  pwmReady = ledcAttach(PWM_LED_PIN, 5000, 8);
  if (pwmReady) {
    ledcWrite(PWM_LED_PIN, 0);
  } else {
    Serial.println("PWM setup failed.");
  }
}

void loop() {
  readInputs();
  processInputs();
  updateOutputs();
  delay(20);
}

void readInputs() {
  buttonPressed = (digitalRead(BUTTON_PIN) == LOW);
  rawInput = analogRead(POT_PIN);
}

int scaleToDuty(int raw) {
  return constrain(map(raw, 0, 4095, 0, 255), 0, 255);
}

void processInputs() {
  requestedDuty = scaleToDuty(rawInput);

  if (buttonPressed && pwmReady) {
    appliedDuty = requestedDuty;
  } else {
    appliedDuty = 0;
  }
}

void updateOutputs() {
  digitalWrite(STATUS_LED_PIN, (buttonPressed && pwmReady) ? HIGH : LOW);

  if (pwmReady) {
    ledcWrite(PWM_LED_PIN, appliedDuty);
  }
}
```

---

## 7. Setup Notes

- `INPUT_PULLUP` turns on the ESP32's built-in resistor, so no extra resistor is needed for the button. The pin reads HIGH when released and LOW when pressed.
- `analogSetPinAttenuation(POT_PIN, ADC_11db)` lets the ADC read the full 0 – 3.3 V range.
- `ledcAttach(pin, 5000, 8)` sets up PWM at 5 kHz with 8-bit resolution (0 – 255).
- `delay(20)` makes the loop run about 50 times per second, which is fast enough to feel instant.

---

## 8. Demonstration (Photos)

| Test | Photo |
|---|---|
| Button released, outputs off | <img width="212" height="315" alt="Screenshot 2026-10-03 124843" src="https://github.com/user-attachments/assets/dd17f22c-3614-48b7-bebc-ab61a1accf56" />
 |
| Button held, low knob | <img width="212" height="304" alt="Screenshot 2026-10-03 203716" src="https://github.com/user-attachments/assets/5d98adb0-2b0e-4efb-8a2f-98e13414ab5c" />
 |
| Button held, middle knob | <img width="211" height="301" alt="Screenshot 2026-10-03 203802" src="https://github.com/user-attachments/assets/c6d5765f-c32c-49df-b6c6-717aaf5da2aa" />
 |
| Button held, high knob | <img width="214" height="301" alt="Screenshot 2026-10-03 203836" src="https://github.com/user-attachments/assets/99e26048-5860-404b-84d1-e4f8dec227e1" />
|

> https://github.com/user-attachments/assets/278fc066-ddc0-4cf0-85cc-dcb3c882dab3

---

## 9. Results

### 9.1 Required tests: expected vs. observed

| # | Test | Expected behavior | Observed behavior | Result |
|---|---|---|---|---|
| 1 | Reset with button released | Both LEDs are off | Both LEDs stayed off after reset | Pass |
| 2 | Hold the button | Status LED turns on (light is enabled) | Status LED turned on while held | Pass |
| 3 | Rotate the knob while held | Brightness LED changes with the knob | Brightness changed smoothly as the knob turned | Pass |
| 4 | Release the button | Both LEDs turn off | Both LEDs turned off right away | Pass |
| 5 | Rotate the knob while released | No LED lights up | LEDs stayed off no matter how the knob was turned | Pass |

### 9.2 Results for low, middle, and high knob positions

The expected values come from the formula `duty = raw × 255 / 4095`. The button is held for all three rows.

| Knob position | Pot voltage (approx.) | Expected `rawInput` | Expected `appliedDuty` | Expected brightness | Observed brightness |
|---|---|---|---|---|---|
| Low (fully one side) | 0 V | 0 | 0 | Off | Brightness LED off or very dim; status LED on |
| Middle | about 1.65 V | about 2048 | about 127 | About half | Medium brightness |
| High (fully other side) | 3.3 V | 4095 | 255 | Full | Brightest, the LED was at full brightness |

> The observed column is from the photos and the live demonstration. Replace the wording with the exact numbers from the Serial Monitor if you recorded them.

---

## 10. Discussion

- **Why use separate functions?** Each function has one job. If the LED did not work, I would check `updateOutputs()`. If the knob reading was wrong, I would check `readInputs()`. This makes the code easier to read and fix.
- **Why have both `requestedDuty` and `appliedDuty`?** The knob always asks for a brightness, but the button decides if the LED may use it. Keeping the two values separate makes this rule easy to see in the code.
- **Why check `pwmReady`?** If PWM setup fails, the program does not try to use it, and it prints an error on the Serial Monitor.
- **Safety of the outputs:** the 220 Ω resistors limit the current so the LEDs and the ESP32 pins are not damaged.

---

## 11. Conclusion

The workstation light works as required. The button acts as an enable switch, the knob sets the brightness only while the button is held, and releasing the button turns everything off. The code is organized into input, processing, and output functions, and the scaling is done in `scaleToDuty(int raw)`. All required tests passed.

---
