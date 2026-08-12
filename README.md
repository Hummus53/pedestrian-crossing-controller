# Real-Time Pedestrian Crossing Controller

A finite-state-machine-based traffic light and pedestrian crossing controller built on an STM32
Nucleo-F446RE. Runs entirely on hardware timers and interrupts — no blocking delays in the main control
loop — with debounced button input and live UART transition logging.

## Features

- Automatic traffic light cycling (green → yellow → red) driven by a hardware timer, not `HAL_Delay`
- Pedestrian crossing request via the onboard user button, handled as a debounced external interrupt
- Safety-enforced state transitions: a pedestrian request can only be honored during the green phase,
  never interrupting yellow or an active crossing
- Flashing "don't walk" warning phase with synchronized audible buzzer alert
- Real-time state transition logging over UART (115200 baud) for observability/debugging

## Hardware

| Component | Qty | Notes |
|---|---|---|
| STM32 Nucleo-F446RE | 1 | |
| LEDs (red, yellow, green) | 3 | traffic light |
| LED (any color) | 1 | pedestrian "walk" signal |
| Active buzzer | 1 | crossing alert |
| Resistors | 4 | current-limiting, one per LED |
| Breadboard + jumper wires | — | |

Pedestrian request uses the Nucleo's **onboard B1 button (PC13)** — no external button required.

## Pin Map

| Signal | Pin |
|---|---|
| CAR_RED | PB0 |
| CAR_YELLOW | PB1 |
| CAR_GREEN | PB2 |
| WALK LED | PB3 |
| Buzzer | PB5 |
| Pedestrian button | PC13 (onboard B1) |
| UART (debug/logging) | PA2/PA3 (USART2, via ST-Link virtual COM port) |
| Phase timer | TIM2 (1-second tick) |

## State Machine

```
CAR_GREEN --(ped request, min 2s) or (12s max)--> CAR_YELLOW
CAR_YELLOW --(2s)--> CAR_RED_WALK
CAR_RED_WALK --(5s)--> CAR_RED_WALK_FLASH
CAR_RED_WALK_FLASH --(3s)--> CAR_GREEN
```

A pedestrian button press is only acted on while in `CAR_GREEN`. Presses during any other state are
ignored for that cycle — the light never skips a safety phase.

## Build & Flash

1. Open the project in STM32CubeIDE.
2. Build (hammer icon) and flash (bug/run icon) to the connected Nucleo board.
3. Open a serial terminal (PuTTY / TeraTerm / `screen`) on the ST-Link virtual COM port at **115200 baud,
   8N1** to view live state transition logs.

## Example UART Output

```
Environmental Monitor booting...
t=2001 ms: -> CAR_YELLOW
t=4003 ms: -> CAR_RED_WALK
Pedestrian request received at t=6210 ms
t=9004 ms: -> CAR_RED_WALK_FLASH
t=12006 ms: -> CAR_GREEN
```

## Design Notes

- **No blocking delays in the main loop.** A hardware timer (TIM2) interrupts once per second and
  increments a counter; the main loop only compares against that counter. This keeps the controller
  responsive to button presses at all times.
- **Flag-and-defer interrupts.** Both the timer callback and the button (EXTI) callback do the minimum
  possible work — set a flag/counter — and let the main loop handle the actual logic. This keeps
  interrupt service routines short and predictable.
- **Software debouncing.** The button callback ignores repeat triggers within 300ms of the last accepted
  press, filtering out mechanical contact bounce.

## Known Limitations / Future Work

- Single direction only — could be extended to a full 2-way intersection with a shared/mutually-exclusive
  green phase.
- Phase durations are fixed constants; a UART command interface (`set green_time <s>`) would allow live
  tuning without reflashing.
- No persistent logging — transition history is only visible live over UART, not stored on the device.

## Author

Built as a portfolio/coop-application project — [your name here].
