# The Museum Break-In
## Physical Laser Obstacle Course Game (DE1-SoC / Nios V)

A physical museum security game built around the **DE1-SoC FPGA board** and its **Nios V / RISC-V processor**. A LEGO minifigure is moved through a 2D laser obstacle course using two independently controlled conveyor axes. The player has to avoid breaking the laser beams, reach a checkpoint that disables the final barrier, and retrieve an "expensive GPU" from the pedestal before the countdown timer expires.

The DE1-SoC is the main controller for the project: it handles game state, timing, sensor events, motor control, and VGA feedback. Arduino Uno boards are used only as supporting hardware interfaces, including analog-to-digital conversion for the photoresistor sensors.

By: **Abby Lui & Christopher Lee**

<p align="center">
  <img src="https://drive.google.com/uc?export=view&id=1f-Li3235e5jmVRYHCrA9G5RY_XgpWKHg" alt="Museum Break-In final project setup with DE1-SoC, wiring, and laser enclosure" width="820">
</p>

---

## Project Demo

**Video:** [Chris & Abby - Final Project Demo](https://drive.google.com/file/d/1x_qFAAnGF3lfjsZumd14p-Xw97Y5nBXw/view?usp=drive_link)

---

## Repository Structure

This public repository is intentionally focused on the **design, architecture, development process, and final prototype**:

- `README.md` - project walkthrough and system explanation.
- [Project documents / report material](https://drive.google.com/drive/folders/1bBI3RPtuUEf3cIKu27lS2UjoVutHspdm?usp=drive_link) - original planning, report, slides, and supporting material.
- [Presentation slideshow carousel](https://docs.google.com/presentation/d/11W4LYpG8vsNoAr06nVIB9mBPIQl4FCc0S6uTiqc41nk/edit?usp=drive_link) - visual project progression.
- [Original project photos and videos](https://drive.google.com/drive/folders/1d_iJXyZSv2pmsGAxg1oJHmERRFaabQWI) - build and final-demo media.

> **Academic Integrity & Licensing**  
> To comply with academic integrity and plagiarism policies at the University of Toronto, the C source code for this course project will **not** be published in this repository.

---

## Game Overview

The player controls a LEGO minifigure suspended from a two-axis conveyor system inside a laser-filled museum enclosure.

The objective is to:

1. Move through the obstacle course using the DE1-SoC pushbuttons.
2. Avoid interrupting any of the laser/photoresistor security beams.
3. Reach the checkpoint, which disables the blocking checkpoint laser.
4. Continue to the final pedestal.
5. Trigger the artifact sensor and retrieve the GPU before the timer expires.

A broken security beam or a timeout immediately transitions the project into a **game-over state**, stops player movement, and changes the VGA output. Reaching the final pedestal after clearing the checkpoint produces the successful completion state.

---

## High-Level Architecture

The project was split into three major layers:

1. **Physical game hardware** - enclosure, lasers, sensors, conveyor motors, fans, wiring, and pedestal.
2. **DE1-SoC control system** - central game state, interrupts, countdown timing, motor control, and PWM.
3. **VGA feedback** - visual state and status output for the player.

```mermaid
flowchart LR
    Keys[DE1-SoC Pushbuttons] --> Core[DE1-SoC / Nios V<br/>Game Logic]

    Lasers[Laser + Photoresistor Pairs] --> ADC[Arduino Uno<br/>Analog-to-Digital Interface]
    ADC --> Core

    Checkpoint[Checkpoint IR Sensor] --> Core
    Pedestal[Pedestal IR Sensor] --> Core

    Core --> Driver[L298N Motor Driver]
    Driver --> MotorX[X-Axis Conveyor Motor]
    Driver --> MotorY[Y-Axis Conveyor Motor]

    Core --> Barrier[Checkpoint Laser Control]
    Core --> VGA[VGA State / Status Display]
```

The important design choice was keeping the **actual game logic on the DE1-SoC**. The Arduino interface converts analog photoresistor readings into signals the FPGA system can use, but movement, timing, game-state decisions, interrupt handling, and output behaviour remain on the Nios V side.

---

## 1. Physical Movement System

The original idea was much simpler: attach the player to a stick and move it manually through the course. During development, that evolved into a much more ambitious **two-axis motorized conveyor system**.

Two TT DC motors independently control the X and Y movement axes. Custom mounts and conveyor parts were designed and 3D printed so the player could be positioned anywhere across the 2D playfield rather than following a fixed track.

| Conveyor development | Final laser course |
| --- | --- |
| <img src="https://drive.google.com/uc?export=view&id=1nMDbdTKCu86vpwDayj4A9Xxa_wHRp61k" alt="3D printed conveyor prototype and TT motor assembly during development" width="390"> | <img src="https://drive.google.com/uc?export=view&id=1bH7-oHzwUtEAOyTYL6LepTP0CKa5BkyF" alt="Interior of the final laser obstacle course with the player and multiple laser beams" width="390"> |

This mechanical system became one of the defining parts of the project because it connected the player's physical movement directly to the board's real-time control system.

---

## 2. Laser Security System

The obstacle system uses **four laser emitters paired with four photoresistors**. Each photoresistor watches a laser beam, creating a physical security boundary across the course.

Because the photoresistors are analog sensors, an Arduino Uno is used as an **analog-to-digital conversion interface**. The converted sensor states are then passed to the DE1-SoC, where the actual game logic decides whether a beam has been interrupted.

When a beam is broken:

- the sensor event is handled by the DE1-SoC,
- the game transitions to the failure state,
- player movement is stopped,
- the VGA output updates to reflect the failure.

This let the laser hardware behave like a real obstacle rather than just a visual effect.

---

## 3. Checkpoint and Artifact Progression

The game is intentionally multi-stage instead of being a single "reach the end" path.

The final design uses **two IR sensors**:

- **Checkpoint IR sensor** - detects when the player has reached the checkpoint and allows the blocking laser to be disabled.
- **Pedestal IR sensor** - detects the final artifact pickup / completion condition.

```mermaid
stateDiagram-v2
    [*] --> Playing
    Playing --> GameOver : security beam interrupted
    Playing --> GameOver : timer reaches zero
    Playing --> CheckpointCleared : checkpoint IR triggered
    CheckpointCleared --> GameOver : security beam interrupted
    CheckpointCleared --> GameOver : timer reaches zero
    CheckpointCleared --> Win : pedestal IR triggered
    GameOver --> Playing : reset
    Win --> Playing : reset
```

This progression gave the physical course a reason to have multiple regions: the player first has to survive the laser maze, then unlock access to the final target, and finally reach the pedestal.

---

## 4. Interrupt-Driven Game Logic

Timing mattered throughout the project. The program needed to watch sensors, update the countdown, control the motors, and update the VGA without one task blocking another.

Rather than putting everything into one large polling loop, the final system uses **interrupt-driven event handling** for the time-sensitive parts of the game.

| Event | Role in the game |
| --- | --- |
| Photoresistor / laser event | Detects a broken security beam and triggers failure |
| Checkpoint IR event | Advances the game and disables the checkpoint barrier |
| Pedestal IR event | Detects successful artifact pickup |
| Countdown timer | Maintains real-time game timing and triggers timeout |
| Motor PWM timer | Schedules motor enable/disable timing independently from the main game flow |

This structure kept input handling and timing responsive even while the display and other logic were active.

---

## 5. Motor Control and Interrupt-Driven PWM

The DE1-SoC drives an **L298N motor driver**, which controls the two conveyor motors.

Directly switching a motor fully on or off was too abrupt for precise movement, so the motor speed was regulated using **pulse-width modulation (PWM)**. The PWM logic had to run on predictable timing; otherwise VGA work or other game logic could introduce delays and make movement inconsistent.

To solve that, motor PWM was implemented using a separate interval-timer interrupt. At the end of each PWM cycle, the motor-control logic checks the active movement keys and updates the direction and enable signals for the two motors.

The result is that:

- the player movement remains responsive,
- motor speed can be reduced to a usable level,
- movement timing is separated from VGA rendering and higher-level game-state work.

---

## 6. VGA Feedback

The VGA display acts as the player's software-side view of the game.

The display was designed to show the current game state and provide clear feedback for states such as:

- active gameplay,
- timer / status information,
- laser-triggered failure,
- timeout,
- successful artifact retrieval.

The project reused the low-level VGA techniques developed earlier in ECE243, but extended them into a complete physical-game interface where the screen reacts to events occurring inside the enclosure.

The original brainstorming also included multiple background concepts and state screens before the final visual direction was settled. Those iterations are preserved in the [project documents and presentation material](https://drive.google.com/drive/folders/1bBI3RPtuUEf3cIKu27lS2UjoVutHspdm?usp=drive_link).

---

## 7. Hardware

| Component | Purpose |
| --- | --- |
| DE1-SoC FPGA board | Main game controller running Nios V |
| 4 x laser emitters | Physical security beams |
| 4 x photoresistors | Laser interruption detection |
| 2 x IR sensors | Checkpoint and final pedestal detection |
| 2 x TT DC motors | X/Y conveyor movement |
| L298N motor driver | Motor direction and enable control |
| 2 x Arduino Uno | Supporting hardware interfaces, including sensor ADC |
| 3 x fans | Improve laser visibility inside the enclosure |
| 12 V / 12 A power supply | Power for the physical hardware |
| 10 kOhm resistors / voltage division | Signal conditioning between hardware levels |
| VGA display | Game-state and status feedback |

### DE1-SoC Inputs

- Four pushbuttons for X/Y movement.
- Converted photoresistor security signals.
- Checkpoint IR sensor.
- Pedestal IR sensor.
- Reset / control input.

### DE1-SoC Outputs

- Six motor-driver control signals.
- Checkpoint laser control.
- VGA output.

---

## 8. Building the Enclosure

The physical enclosure changed significantly as the project moved from planning to integration.

The laser paths were first modelled in 3D so there would be a valid path through the course. The conveyor mounts were then designed around the actual TT motors and printed to fit the required movement geometry.

As more hardware was installed, the enclosure gained:

- a back-access panel for maintenance,
- clear front/top covering for visibility,
- an internal pedestal and checkpoint,
- laser emitters and matching sensor locations,
- cable routing and power distribution,
- fans to make the beams more visible inside the enclosure.

The final prototype ended up being much more hardware-heavy than we originally expected, which also made integration and debugging a major part of the project.

---

## 9. Testing, Debugging, and Iteration

This project made debugging very different from a normal software-only lab because a failure could come from several layers at once.

During integration, issues included:

- incorrect game logic,
- GPIO or wiring mistakes,
- loose cables,
- sensor alignment,
- physical wear on the moving parts,
- restricted access inside the enclosure,
- material interfering with sensors or gearboxes,
- laser/photoresistor misalignment causing immediate false triggers.

The first stage of development focused on testing sensors, motors, and the enclosure separately. The later stage was mostly about integrating those systems and finding the boundary between a software bug, an electrical problem, and a mechanical problem.

That integration process was one of the biggest lessons from the project: once software is controlling a real physical system, debugging becomes a full-system engineering problem.

---

## Design Evolution

Not every feature stayed exactly as it was first proposed.

### What changed

- The early checkpoint concept used an ultrasonic sensor and manual trigger; the final design moved to an **IR-based checkpoint system**.
- The movement concept evolved from a simpler manual mechanism into a **fully motorized two-axis conveyor**.
- VGA feedback and final artifact pickup were completed for the demo.
- Background music and the planned audio alarm were part of the optional scope but were **not completed** in the final build.

Keeping these changes visible is important because the final project was the result of repeated scope decisions rather than a one-shot implementation of the proposal.

---

## Attribution

### Abby Lui

- Used the Arduino as an analog-to-digital interface for the photoresistors.
- Sent and received GPIO signals for the IR sensor, photoresistor interrupts, and laser activation.
- Implemented the IR, photoresistor, and timer interrupt handling.
- Implemented the main game-state logic for progress, laser failure, and timeout.
- Implemented the VGA display and LED indicators.

### Christopher Lee

- Retrieved movement input from the DE1-SoC keys.
- Configured GPIO control signals for the motor driver.
- Designed and integrated the custom interrupt-driven PWM motor-control system.
- Designed and 3D printed the physical conveyor parts.
- Assembled the physical components and managed the project wiring.

**Supervising TA:** Angela Yu

---

## Supporting Material

- [Project documents, report, brainstorming, and source visuals](https://drive.google.com/drive/folders/1bBI3RPtuUEf3cIKu27lS2UjoVutHspdm?usp=drive_link)
- [Presentation slideshow carousel](https://docs.google.com/presentation/d/11W4LYpG8vsNoAr06nVIB9mBPIQl4FCc0S6uTiqc41nk/edit?usp=drive_link)
- [Original photos and videos](https://drive.google.com/drive/folders/1d_iJXyZSv2pmsGAxg1oJHmERRFaabQWI)
- [Final demo video](https://drive.google.com/file/d/1x_qFAAnGF3lfjsZumd14p-Xw97Y5nBXw/view?usp=drive_link)

---

## Final Result

The Museum Break-In ended up combining **embedded software, FPGA I/O, real-time interrupts, motor control, sensors, 3D-printed mechanisms, physical fabrication, and VGA graphics** into one interactive system.

What began as a laser-security game became a project where the difficult part was getting every layer to work together at the same time. That systems-integration challenge is also what made the project one of the most rewarding parts of ECE243.
