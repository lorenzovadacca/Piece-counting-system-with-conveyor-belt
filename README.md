# Piece-counting system with conveyor belt (Fluid Automation)

A complete simulation, electro-pneumatic design, PLC ladder logic, and Grafcet implementation for an automated piece-counting conveyor system developed for the **Fluid Automation** course (Master's Degree in Mechatronics Engineering) at Politecnico di Torino.

---

## Table of contents
1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Architecture](#architecture)
   - [Pneumatic & Electro-pneumatic circuit](#1-pneumatic--electro-pneumatic-circuit)
   - [Grafcet control logic](#2-grafcet-control-logic)
   - [PLC Ladder logic implementation](#3-plc-ladder-logic-implementation)
   - [Human-Machine Interface (HMI)](#4-human-machine-interface-hmi)
4. [Specifications](#specifications)
   - [Digital inputs](#digital-inputs)
   - [Digital outputs](#digital-outputs)
5. [Results](#results)

---

## Overview

The system automates and monitors a conveyor belt driven by a pneumatic motor, equipped with an optical detection line to count transported workpieces:
* **Conveyor Belt Drive:** Actuated by a pneumatic motor controlled via a monostable $5/2$-way directional valve ($M+$).
* **Piece Simulation & Optical Counting:** The movement of the pieces along the belt is emulated via a double-acting cylinder with magnetic carriage passing magnetic limit switches ($a_0$, $a_1$, $a_2$) and counted by a retroreflective photosensor ($I_2$) on the outward stroke ($A+$) at intermediate position $a_1$.
* **Interruption & Resumption:** The belt can be halted at any moment by pressing a Stop pushbutton. When restarted via the Start pushbutton, the sequence and counting resume from where they were interrupted without loss of counter state.
* **Safety & Homing:** Integrated emergency stop with latching twist-release button, acoustic/visual alarms, and a dedicated 3-second long-press Reset routine for system homing ($a_0$) and counter clear.

---

## Requirements

* **Simulation Suite:** Famic Technologies **Automation Studio™** (v7.0 or higher recommended).
* **Enabled Modules:**
  * Pneumatics & Electro-Pneumatics Library
  * PLC Ladder Logic Module (IEC 61131-3 compliant)
  * Sequential Function Chart (SFC / Grafcet) Module
  * 2D HMI & Animation Dashboard

---

## Architecture

### 1. Pneumatic & Electro-pneumatic circuit
* **Conveyor Pneumatic Motor Loop:**
  * Driven by a $5/2$-way solenoid valve with spring return ($M+$).
  * Monitored upstream by an air flow switch ($r_2$) to verify active air delivery ($13.3\text{ L/min}$ nominal, $9.5\text{ bar}$) versus idle state ($0\text{ L/min}$, $10\text{ bar}$).
* **Piece Motion Cylinder Loop:**
  * Actuated by a bistable $5/2$-way directional control valve with double solenoid ($A+$, $A-$).
  * Outward/inward line fitted with two $3/2$ normally closed emergency cut-off valves controlled by solenoid `em_sol`.
  * Return chamber equipped with a variable one-way throttle valve (meter-out) for linear speed regulation.

### 2. Grafcet control logic
Organized into modular, interacting partial GRAFCETs:
* **Start/Stop & Main Setting:** Interlocks Start and Stop pushbuttons with the master ON/OFF toggle switch; latches memory flags `M_start` and `M_stop`.
* **Cycle Execution:** Controls cylinder extension ($A+$) from home position $a_0$, triggers count register increment ($\text{counter} := \text{counter} + 1$) upon crossing $a_1$, commands cylinder retraction ($A-$) once reaching fully extended sensor $a_2$, and loops while `M_start` is asserted.
* **Emergency Management:** Evaluates the emergency stop button state. If engaged, trips `em_sol`, turns off active line solenoids, sounds the acoustic buzzer, and enables the flashing red beacon.
* **Homing & Reset:** Detects a 3-second hold on the Reset button (`3s / HMI5_3."Reset"`), retracts the cylinder to $a_0$, clears the internal counter register, and resets internal sequence steps.
* **Signaling & LED GRAFCET:** Manages pilot light states ($a_0\text{LED}$, $a_1\text{LED}$, $a_2\text{LED}$, Run, Stop).

### 3. PLC Ladder logic implementation
Implemented using the **Step-Transition Method** partitioned across specialized Function Calls (FCs):
* `Main (L3_1)`: Master scheduling rung calling subroutines and driving cylinder physical position LEDs.
* `InitialconditionsFC (L69_1)`: Verifies initial readiness ($a_0$ reached, air flow switch $r_2$ clear, system steps reset, and main power switch active).
* `steptransitionFC (L281_1)`: Sequencer enforcing Step 1 (extension / belt run), Step 2 (retraction), and Step 3 transitions via Set/Reset $(S)/(R)$ memory bits.
* `ConveyormotionFC (L174_1)`: Energizes conveyor drive valve ($M+$ / `OUT0`) in Step 1, with stop interlocks requiring sensor $a_2$ reach to prevent mid-cycle workpiece entrapment.
* `BoxmotionFC (L2_1)`: Governs solenoid drivers $A+$ (`OUT3`) and $A-$ (`OUT4`), safety valve `em_sol` (`OUT2`), and evaluates on-delay timer `S_ODT` ($3\text{ s}$) for homing memory `M_home`.
* `CounterFC (L267_1)`: Increments up-counter `S_CU` exclusively on the rising edge of sensor $a_1$ while Step 1 is active (eliminating false triggers on the inward stroke).
* `LightsFC (L194_1)`: Controls `go_led` (`OUT5`) and `stop_led` (`OUT6`).
* `EmergencyFC (L7_1)` & `ResetFC (L8_1)`: Handle latching safety interrupts and homing resets.

### 4. Human-Machine Interface (HMI)
* **Control Elements:** Start, Stop, Reset, Emergency twist-release mushroom pushbutton, ON/OFF selector switch.
* **Visual & Acoustic Indicators:**
  * Running pilot lamp (Green - `go_led`), Stopped lamp (Red - `stop_led`), Reset indicator (Yellow).
  * Cylinder position status pilot lamps ($a_0, a_1, a_2$).
  * High-visibility blinking stack lights, flashing beacon light, and audible horn (`MMI Buzzer`).
  * 7-segment / numeric LCD display showing total counted parts in real-time.

---

## Specifications

### Digital inputs

| Card Address | Symbolic Tag | Hardware Component | Functionality |
| :--- | :--- | :--- | :--- |
| `E10_1.IN0` | `startPB` | Pushbutton (NO) | Start conveyor & cycle |
| `E10_1.IN1` | `stopPB` | Pushbutton (NO) | Normal Cycle Stop |
| `E10_1.IN2` | `emergencyPB` | Twist-release Pushbutton (NC) | Emergency Stop |
| `E10_1.IN3` | `a0` | Magnetic Reed Switch | Cylinder Home Position ($0\text{ mm}$) |
| `E10_1.IN4` | `a1` | Magnetic Reed / Photosensor | Piece Count Position ($a_1$) |
| `E10_1.IN5` | `a2` | Magnetic Reed Switch | Cylinder Fully Extended Position |
| `E10_1.IN6` | `r2` | Flow Switch | Pneumatic Air Flow Detection |
| `E10_1.IN7` | `resetPB` | Pushbutton (NO) | Reset & Homing ($3\text{ s}$ hold) |
| `E73_1.IN0` | `onoffPB` | Toggle / Slider Switch | System Main Power (ON/OFF) |

### Digital outputs

| Card Address | Symbolic Tag | Actuator / Indicator | Description |
| :--- | :--- | :--- | :--- |
| `E13_1.OUT0` | `a0_led` | Pilot Light (Green) | Carriage at Home Position $a_0$ |
| `E13_1.OUT1` | `a1_led` | Pilot Light (Green) | Carriage at Counting Station $a_1$ |
| `E13_1.OUT2` | `a2_led` | Pilot Light (Green) | Carriage at Stroke End $a_2$ |
| `E13_1.OUT3` | `A+` | Solenoid Valve Coil | Double-acting cylinder extension |
| `E13_1.OUT4` | `A-` | Solenoid Valve Coil | Double-acting cylinder retraction |
| `E13_1.OUT5` | `go_led` | Pilot Light (Green) | Machine in Running state |
| `E13_1.OUT6` | `stop_led` | Pilot Light (Red) | Machine in Stopped state |
| `E13_1.OUT7` | `em_led` / `alarm_buzzer` | Beacon Lamp & Buzzer | Emergency acoustic/visual alarm |
| `E33_1.OUT0` | `M+` | Solenoid Valve Coil | Conveyor pneumatic motor drive |
| `E33_1.OUT1` | `reset_led` | Pilot Light (Yellow) | Reset / Homing execution active |
| `E33_1.OUT2` | `em_sol` | Solenoid Valve Coil | Emergency safety pneumatic valves |

---

## Results

Dynamic behavior and timing diagrams obtained in Automation Studio confirm expected system performance across all operational modes:

1. **Continuous counting cycle:**
   * Upon pressing Start (`E10_1.IN0`), `go_led` and motor drive $M+$ activate simultaneously with steady flow ($13.3\text{ L/min}$).
   * The piece stroke sequence initiates ($A+$). Crossing $a_1$ produces an instantaneous count signal, incrementing the display counter register by $1$.
   * At stroke completion ($a_2$), retraction ($A-$) engages smoothly, returning to $a_0$ before restarting the subsequent cycle.
2. **Cycle stop execution:**
   * Activating Stop (`E10_1.IN1`) allows the piece currently in transit to reach position $a_2$ before motor shutdown, preventing workpieces from stalling midway along the belt.
   * `stop_led` illuminates, and the counter retains its accumulated value.
3. **Emergency interruption:**
   * Activating the twist-release emergency stop triggers `em_sol` and `OUT7` in $< 20\text{ ms}$.
   * Air supply to the cylinder and pneumatic motor exhausts immediately ($0\text{ bar}$ line pressure), locking movement, turning on the flashing beacon, and sounding the alarm buzzer.
4. **Homing & Reset routine:**
   * Holding the Reset button for $t \ge 3\text{ s}$ energizes `M_home`, returning the carriage to $a_0$, restoring line pressure, and clearing the piece counter to $0$.
