# One-Pedal Drive Controller — System Definition, HARA & MIL


A **Model-Based Software Development (MBSD)** project for an automotive one-pedal drive controller, following the **ISO 26262** functional safety lifecycle. This repository covers system definition, requirements, and HARA, safety goals, FSM design and Model-in-the-Loop validation.

> 📦 Safety mechanism implementation, Software-in-the-Loop (SIL), code generation, and Process-in-the-Loop (PIL) testing are covered in the [companion repository](#).


## Table of Contents

- [System Overview](#system-overview)
- [System Definition & Requirements](#system-definition--requirements)
- [Hazard Analysis and Risk Assessment (HARA)](hazard-analysis-and-risk-assessment-hara)
  - [Identified Hazards](#identified-hazards)
  - [Functional Failures](#functional-failures)
  - [Operational Scenarios](#operational-scenarios)
  - [Estimation Matrix](#estimation-matrix)
- [Safety Goals & ASIL Definition](#safety-goals--asil-definition)
- [Safety Function Definition & Model-Based Design](#safety-function-definition--model-based-design)
  - [Simulink Project Structure](#simulink-project-structure)
  - [Controller FSM](#controller-fsm)
  - [Model-in-the-Loop (MIL) Validation](#model-in-the-loop-mil-validation)
- [Project Structure](#Project-Structure)
- [Technology Stack](#technology-stack)
- [Skills Demonstrated](#skills-demonstrated)


## System Overview

The controller manages torque requests to the electric motor based on pedal position and transmission selector state. It features five driving modes: **P** (Park), **R** (Reverse), **N** (Neutral), **D** (Drive — traditional throttle behaviour), and **B** (Brake / One-Pedal mode).

In **B mode**, the pedal travel is split into two regions:
- **Braking region** (0 – 1/3 pedal): regenerative braking torque, maximum at full release, tapering to zero at the neutral point
- **Acceleration region** (1/3 – 1 pedal): traction torque proportional to pedal position

Mathematically:

```
τr = −max_τa · (1 − 3p)         when 0 < p ≤ 1/3   (braking)
τr =  max_τa · 3/2 · (p − 1/3)  when 1/3 < p ≤ 1   (acceleration)
```


## System Definition & Requirements

Defined the item boundary, functional behaviour, and all interactions with external systems according to ISO 26262.

**Item boundary and responsibilities:**
- Computes the torque request (positive for acceleration, negative for regenerative braking) and transmits it to the electric motor ECU via CAN bus
- Reads throttle pedal position, transmission selector state, vehicle speed, and brake pedal status
- Prevents reverse motion during regenerative braking by monitoring vehicle speed
- Holds the vehicle stationary in B mode until the throttle pedal re-enters the acceleration region
- Mode transitions between N↔R and N↔D/B require speed < 5 km/h and brake pedal pressed; N can be selected at any time

**Functional requirements identified:**
- Gear selector position to detect active mode
- Vehicle speed to control regenerative braking behaviour
- Throttle pedal position to compute torque request
- Brake pedal status for emergency braking, creep behaviour, and mode transitions

<p align="center">
  <img width="600" alt="Functional block diagram of the one-pedal controller and its interactions with external systems" src="https://github.com/user-attachments/assets/47982422-44a5-40d6-9c73-4ae47d4004de" />
  <br><em>Functional block diagram — controller interactions with external systems</em>
</p>

**Controller interfaces:**

| Signal | Direction | Type | Range |
|--------|-----------|------|-------|
| Throttle Pedal Position | Input | Analog (normalized) | [0, 1] |
| Brake Pedal Pressed | Input | Digital (Boolean) | {0, 1} |
| Transmission Selector State | Input | Enum | {P, R, N, D, B} |
| Vehicle Speed | Input | Single (km/h) | [−45, 230] |
| Requested Torque | Output | Single (N·m) | [−40, 80] |
| Transmission State (display) | Output | Enum | {P, R, N, D, B} |

## Hazard Analysis and Risk Assessment (HARA)

Conducted a full HARA per ISO 26262, identifying hazardous events, operational scenarios, and assigning ASIL levels to safety goals.

### Identified Hazards

| ID | Hazard |
|----|--------|
| H1 | Unintended vehicle acceleration (overestimated positive torque) |
| H2 | Unintended vehicle deceleration (overestimated negative torque) |
| H3 | Insufficient vehicle acceleration (underestimated positive torque) |
| H4 | Insufficient vehicle deceleration (underestimated negative torque) |
| H5 | Unintended vehicle motion in the wrong direction (reverse during braking) |

### Functional Failures

| Failure | Description | Related Hazards |
|---------|-------------|-----------------|
| F1 | Broken analog input for pedal position | H1, H2, H3, H4 |
| F2 | Wrong vehicle speed detection | H5 |
| F3 | CAN bus failure (corrupted inputs from all connected items) | H1–H5 |

### Operational Scenarios

| Scenario | Driving Situation | Vehicle Status |
|----------|-------------------|----------------|
| S1 | Highway driving | Stopped |
| S2 | Highway driving | Low speed |
| S3 | Highway driving | Fast / Medium speed |
| S4 | City driving | Stopped |
| S5 | City driving | Low speed |
| S6 | City driving | Fast / Medium speed |
| S7 | Off-road driving | Stopped |
| S8 | Off-road driving | Low speed |
| S9 | Off-road driving | Fast / Medium speed |

### Estimation Matrix

S = Severity · E = Exposure · C = Controllability

| Hazard | S1 | S2 | S3 | S4 | S5 | S6 | S7 | S8 | S9 | Top Event (worst case) | ASIL |
|--------|----|----|----|----|----|----|----|----|-----|------------------------|------|
| H1 | — | S:1 E:4 C:1 | S:3 E:4 C:2 | S:2 E:4 C:1 | S:2 E:4 C:1 | S:3 E:3 C:2 | S:1 E:3 C:2 | S:1 E:3 C:2 | S:2 E:3 C:2 | S:3 E:4 C:2 | **C** |
| H2 | — | S:1 E:4 C:0 | S:2 E:4 C:1 | — | S:0 E:4 C:0 | S:1 E:3 C:1 | — | S:0 E:3 C:0 | S:0 E:3 C:1 | S:2 E:4 C:1 | **A** |
| H3 | — | S:1 E:4 C:0 | S:2 E:4 C:2 | S:0 E:4 C:0 | S:0 E:4 C:0 | S:1 E:3 C:2 | S:0 E:3 C:0 | S:0 E:3 C:0 | S:0 E:3 C:0 | S:2 E:4 C:2 | **B** |
| H4 | — | S:2 E:4 C:1 | S:3 E:4 C:2 | — | S:2 E:4 C:1 | S:3 E:3 C:2 | — | S:2 E:3 C:2 | S:2 E:3 C:2 | S:3 E:4 C:2 | **C** |
| H5 | — | S:1 E:4 C:1 | S:3 E:4 C:2 | — | S:1 E:4 C:1 | S:3 E:3 C:2 | — | S:1 E:3 C:1 | S:2 E:3 C:2 | S:3 E:4 C:2 | **C** |


## Safety Goals & ASIL Definition

Defined safety goals from the HARA results, specifying the safe state and fault tolerance time intervals (FTTI).

| Safety Goal | Description | ASIL | Safe State | FTTI |
|-------------|-------------|------|------------|------|
| SG1 | Detect CAN disconnection, warn driver, disable B mode | ASIL C | B mode disabled + warning lamp | Before driving begins; immediately if fault occurs during driving |
| SG2 | Detect ECU failure, warn driver, disable B mode | ASIL C | B mode disabled + warning lamp | Before driving begins; immediately if fault occurs during driving |
| SG3 | Periodic health check of vehicle speed sensor | ASIL C | B mode disabled + warning lamp | Speed fault must be detected before braking or applying extra negative torque |
| SG4 | Periodic mechanical check of the throttle pedal | ASIL B/C | B mode disabled + warning lamp | Before driving begins |


## Safety Function Definition & Model-Based Design

Designed the controller as a **Finite State Machine** in MATLAB/Simulink Stateflow and validated it at **Model-in-the-Loop (MIL)** level alongside a vehicle plant model.

### Simulink Project Structure

| File | Contents |
|------|----------|
| `Harness.slx` | Reference model wiring and test stimulus generation |
| `Controller.slx` | FSM-based one-pedal controller |
| `Plant.slx` | Vehicle longitudinal dynamics model |

### Controller FSM

| State | Behaviour | Entry Condition |
|-------|-----------|-----------------|
| Park (P) | Zero torque, vehicle held stationary | Initial state |
| Neutral (N) | Freewheel, zero torque | Selector → N at any speed |
| Reverse (R) | Negative torque proportional to pedal | From N: speed < 5 km/h AND brake pressed |
| Drive (D) | Traction torque ∝ pedal position; hydraulic brake for deceleration | From N: speed < 5 km/h AND brake pressed |
| Brake (B) | One-pedal mode with three sub-states | From D: pedal > 1/3 AND selector = B |

**Brake (B) sub-states:**
- **Accelerate**: pedal > 1/3 — traction torque proportional to pedal position
- **Decelerate**: pedal ≤ 1/3 — regenerative braking torque
- **Stopped**: vehicle speed = 0 — zero torque held until pedal re-enters acceleration region, preventing unintended reverse motion

<p align="center">
  <img width="400" alt="Controller FSM top-level states: Park, Neutral, Reverse, Drive, Brake" src="https://github.com/user-attachments/assets/2d7f6156-7e96-47be-8a0e-09502f62eb43" />
  <br><em>Controller FSM — top-level states </em>
</p>

<p align="center">
  <img width="900" alt="Full Simulink Stateflow chart showing all states and transitions of the one-pedal controller" src="https://github.com/user-attachments/assets/9e0f668c-eb48-4b03-a9ef-571e3ce978f0" />
  <br><em>Full Stateflow chart — all states and transition conditions</em>
</p>

### Model-in-the-Loop (MIL) Validation

The controller and plant models were connected in the Harness and simulated together to validate the full one-pedal control logic against the expected signal behaviour. Test stimuli covered all transmission mode transitions, one-pedal torque law correctness, and the stopped-state hold condition.

<p align="center">
  <img width="500" alt="MIL simulation results from Simulink Data Inspector showing AutomaticTransmissionState, ThrottlePedalPosition, BrakePedalPressed, Vehicle_Speed_km_h and TorqueRequest_Nm over 100 seconds" src="https://github.com/user-attachments/assets/ce927ab0-ce78-4682-a941-1d7743a5199a" />
  <br><em>MIL simulation results — Data Inspector view showing transmission state, pedal position, vehicle speed, and torque request over a 100-second test sequence</em>
</p>

## Project Structure

```
.
├── controller.slx        # FSM-based one-pedal controller (Simulink model)
├── TransmissionState.m   # Enumeration definition for transmission states (P, R, N, D, B)
├── harness.slx           # Test harness — model wiring and stimulus generation (Provided by Course instructor)
├── init_fn.m             # Initialization script — loads all model parameters
└── plant.slx             # Vehicle longitudinal dynamics plant model (Provided by Course instructor)

```

## Technology Stack

- MATLAB / Simulink / Stateflow
- ISO 26262 (automotive functional safety standard)

## Skills Demonstrated

- ISO 26262 safety lifecycle: item definition → HARA → ASIL classification → safety goals
- Finite State Machine design and implementation in Stateflow
- Vehicle longitudinal dynamics modelling
- Model-in-the-Loop (MIL) simulation and verification
- Functional safety analysis: hazard identification, SEC estimation, ASIL assignment
