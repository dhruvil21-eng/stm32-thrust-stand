# Static Thrust Stand: STM32 + HX711 Load Cell

A DIY static thrust stand that measures and logs the force a small motor
produces over time. Built before the start of the school year to understand
microcontrollers at a deeper level and to get hands-on with the fundamentals
of robotics: sensor interfacing, calibration, and real-time data capture.

## What it does

Reads a load cell in real time via an STM32F411 microcontroller and an HX711
amplifier, converting the raw sensor signal into calibrated force
measurements. Designed to capture a rocket motor's thrust curve during a
static test.

## Hardware

- STM32F411CEU6 ("Black Pill") development board
- HX711 load cell amplifier module
- 5kg straight-bar load cell
- Custom cantilever-beam mount (wood base, fixed post, load cell bolted at
  one end, free end holds the motor bracket)
- ST-Link V2 programmer/debugger
- Estes B6-4 model rocket engine (18mm class) for the live static test

## How it works

The load cell is mounted as a cantilever beam: one end rigidly fixed, the
other end free to flex under load. A motor mount bracket sits on the free
end; when the motor produces thrust, it pushes down on the bracket, flexing
the beam. The HX711 amplifies the load cell's tiny analog signal into a
digital reading the STM32 can read directly.

Since the HX711 doesn't use a standard peripheral (like I2C or SPI), the
firmware bit-bangs its 2-wire protocol manually: pulsing a clock pin and
reading 24 bits of data back on each cycle.

## Firmware

Written in C using STM32CubeIDE and the STM32 HAL. Key logic lives in
`HX711_Read()` in `firmware/Core/Src/main.c`, which:
1. Waits for the HX711 to signal data is ready
2. Pulses the clock pin 24 times, reading one bit per pulse (read while the
   clock line is held high, matching the HX711's actual timing protocol)
3. Sign-extends the resulting 24-bit value to a signed 32-bit integer

## Calibration

Raw HX711 output is calibrated against known reference weights to convert
readings into real force units.

## Known limitations

- The HX711 outputs new readings at 10 samples/second by default, which
  limits temporal resolution during a fast motor burn (typically 1-3
  seconds total). That is enough to capture the overall shape of a thrust
  curve, but not a lab-grade high-resolution capture.
- This is a hobby-grade instrument, calibrated with household reference
  weights rather than certified lab standards.
- The engine mount is a DIY thrust flange rather than a manufactured mount
  kit, built to keep the project moving on a tight timeline. It is
  mechanically sound for a single static test, though less refined than a
  purpose-built mount.
