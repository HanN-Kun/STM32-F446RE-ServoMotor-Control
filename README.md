# ServoMotorMG90s-Control

Servo control projects for the **STM32 NUCLEO-F446RE** board, built with **STM32CubeIDE** and the **HAL** library. The projects start with simple PWM generation and gradually add an ADC input, an external interrupt and a small state machine, all driving a single **MG90S** micro servo.

## Repository Structure

Each project lives on its own branch, and every project builds on the previous one.

| Branch | Project | Description |
|---|---|---|
| `main` | **1. Servo Cycle** | The servo sweeps automatically 0° → 180° → 0° using timer PWM. |
| `servo-motor-ctrlwith-potentiometer` | **2. Servo + Potentiometer** | The servo angle follows a potentiometer read through the ADC. |
| `servo-motor-dual-mode` | **3. Servo + Button + LED** | The on-board user button (B1) toggles between automatic sweep and potentiometer control. LD2 shows the active mode. |

## Hardware

- STM32 NUCLEO-F446RE
- MG90S micro servo
- 10 kΩ potentiometer (projects 2 and 3)
- 1000 µF electrolytic capacitor (across the servo supply)
- Breadboard and jumper wires

### Wiring

| Component | Connection |
|---|---|
| Servo signal (yellow/orange) | PA6 (D12) |
| Servo power (red) | 5V |
| Servo ground (brown) | GND |
| 1000 µF capacitor | + to 5V, − to GND, placed close to the servo |
| Potentiometer outer pins | 3V3 and GND |
| Potentiometer middle pin | PA0 (A0) |

> The potentiometer must be powered from **3V3**, not 5V, because the ADC input must not exceed 3.3 V. Swapping the two outer pins only reverses the rotation direction.

## Peripheral Configuration

Default clock configuration (HSI, 16 MHz). If the clock is changed, the timer prescaler must be recalculated.

| Peripheral | Settings |
|---|---|
| TIM3 CH1 (PA6) | PWM mode 1, Prescaler = 15, Counter Period = 19999 (1 µs tick, 20 ms period, 50 Hz) |
| ADC1 IN0 (PA0) | 12-bit, single conversion (continuous mode disabled), 84 cycles sampling time |
| PC13 (B1) | EXTI, falling edge, no pull (the board has an external pull-up) |
| PA5 (LD2) | GPIO output |

ADC and EXTI are used from project 2 and project 3 onwards, respectively.

## Servo Calibration

The servo position is set by the PWM pulse width in microseconds. The values below were measured for the servo used in this project to reach the full 180° range:

```c
#define SERVO_MIN_US   500    // 0°
#define SERVO_MAX_US   2700   // 180°
```

Servos differ from unit to unit. If the servo hums or strains at either end of its travel, it is hitting its mechanical limit, so reduce the value slightly.

## Projects

### 1. Servo Cycle (`main`)

Generates a 50 Hz PWM signal on TIM3 and steps the angle from 0° to 180° and back in 1° steps every 10 ms.

**Concepts:** timer PWM, frequency and duty cycle calculation, servo pulse width control.

### 2. Servo + Potentiometer (`servo-motor-ctrlwith-potentiometer`)

Reads the potentiometer with the ADC (0–4095) and maps the value linearly to the servo pulse width. A simple exponential moving average filter reduces jitter:

```c
adc_filtered = (adc_filtered * 7 + adc_raw) / 8;
```

**Concepts:** ADC polling, converting a raw ADC value to a physical quantity, signal filtering.

### 3. Servo + Button + LED (`servo-motor-dual-mode`)

Pressing the on-board user button B1 toggles between two modes:

- **Mode A (LD2 off):** automatic sweep, as in project 1.
- **Mode B (LD2 on):** servo follows the potentiometer, as in project 2.

The button is handled with an EXTI interrupt on PC13 and a software debounce of 200 ms. The automatic sweep advances one step per loop iteration instead of using blocking loops, so a mode change takes effect immediately.

**Concepts:** EXTI interrupts, software debounce, a simple state machine, non-blocking code.
