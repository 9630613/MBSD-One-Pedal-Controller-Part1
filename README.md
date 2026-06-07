# Model-Based Software Design — One Pedal Controller


A project implementing a **one-pedal drive controller** for an electric vehicle using **MATLAB/Simulink**, following the ISO 26262 automotive safety standard lifecycle. The project covers the full development pipeline: safety analysis → model-based design → code generation → hardware deployment.


## Project Overview

The item under development is a **one-pedal acceleration/braking system** for an electric compact car. It allows the driver to use a single throttle pedal for both acceleration and regenerative braking, depending on the selected transmission mode (D or B).

The system features an automatic transmission selector with five positions: **P** (Park), **R** (Reverse), **N** (Neutral), **D** (Drive), **B** (Brake/One-Pedal).


## 

### Item Definition & HARA
**Files:** `1-2_Item_definition_-_HARA_One_pedal_-_students.docx`

- Defined the item purpose and functional behavior per ISO 26262
- Described the one-pedal torque control law:
  - *Braking region* (pedal 0–1/3): regenerative braking, max torque when fully released
  - *Acceleration region* (pedal 1/3–1): traction torque proportional to pedal position
- Conducted the **Hazard Analysis and Risk Assessment (HARA)**

### Simulink Controller & Plant Model
**Files:** `MBSD_Lab_2.pdf`

Implemented the system as a set of three Simulink models:
- `Harness.slx` — test stimuli and reference model wiring
- `Controller.slx` — finite state machine-based controller
- `Plant.slx` — vehicle longitudinal dynamics model

**Controller FSM states:**
| State | Description |
|-------|-------------|
| Park (P) | Initial state, zero torque, vehicle stationary |
| Neutral (N) | Freewheel, zero torque |
| Reverse (R) | Negative torque; entered from N at speed < 5 km/h with brake pressed |
| Drive (D) | Traditional throttle-proportional torque; braking via hydraulic pedal |
| Brake (B) | One-pedal mode with sub-states: Accelerate, Decelerate, Stopped |

**Plant parameters (electric compact car):**
- Mass: 1600 kg | Wheel radius: 0.3 m | Frontal area: 3.5 m²
- Torque range: [−60, 960] Nm | Top speed: ~230 km/h

**Vehicle longitudinal dynamics model:**

$$v(t) = \frac{1}{m} \int_0^t \left[\frac{T(t)}{r} - X_{air} \cdot v(t)^2 - X_{tyres} \cdot |v(t)| \right] dt$$

### Safety Mechanisms
Integration of functional safety mechanisms into the Simulink model to meet ISO 26262 requirements.

### Code Generation & Arduino Deployment
**Files:** `MBSD_Lab_4.docx`

Generated embedded C code from the Simulink model and deployed it to **Arduino hardware** using the Simulink Support Package for Arduino.

**Hardware I/O mapping:**

| Signal | Type | Conversion |
|--------|------|-----------|
| Brake Pedal Pressed | Digital Input | Direct |
| Throttle Pedal Position | Analog Input | `5/(2^10−1) × 1/5` |
| Vehicle Speed | Analog Input | `5/(2^10−1) × (275/5)` |
| Transmission Selector (R,B,P,N,D) | Digital Input | 5 pins via Stateflow |
| Requested Torque | PWM Output | `255/160` scaling |
| Transmission State Output | Digital Output | 5 pins via Stateflow |

Hardware-in-the-loop testing performed in **SimulIDE**, using switches, potentiometers, LEDs, and an oscilloscope to validate the controller behavior.


## Tech Stack

- MATLAB / Simulink / Stateflow
- Simulink Support Package for Arduino Hardware
- SimulIDE (hardware-in-the-loop simulation)
- ISO 26262 (automotive functional safety standard)


## Key Skills Demonstrated

- Model-Based Software Development (MBSD)
- Finite State Machine design in Stateflow
- Vehicle dynamics modeling
- Functional safety analysis (HARA, ISO 26262)
- Automatic C code generation from Simulink models
- Embedded systems deployment on Arduino
