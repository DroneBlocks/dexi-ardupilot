# Board config — DroneBlocksH743AIO

The ArduPilot board target for the DEXI-3 flight controller.

## Lineage: it's a MicoAir H743-AIO

The DroneBlocks H743-AIO is a fork of the **MicoAir H743-AIO**, and ArduPilot
already ships a first-class `MicoAir743-AIO` target. Our board is **rebased on
it** rather than hand-ported. They share:

- STM32H743VI MCU, 8 MHz HSE
- IMUs: **BMI088 + BMI270** on SPI2 (BMI270_CS = PA15)
- Baro: **DPS310** on internal I2C2 @ 0x76
- USB on OTG_FS (PA11/PA12), PA9 = GPS UART
- Same PWM timer map, SD on SDMMC1, LEDs on PE4/5/6

## The only DroneBlocks deltas

Starting from `MicoAir743-AIO/hwdef.dat` + `hwdef-bl.dat`, we change **two**
things:

1. `APJ_BOARD_ID` → `AP_HW_DroneBlocksH743AIO` (**5290**), registered in
   `Tools/AP_Bootloader/board_types.txt`. (1240, our PX4 board id, is reserved
   for YARI Robotics in ArduPilot's namespace.)
2. `HAL_BATT_VOLT_SCALE` → **11.0** (DroneBlocks PM, = PX4 `BAT1_V_DIV`), vs
   MicoAir's 21.12.

Everything else is MicoAir's proven config.

## The gotcha that cost us the day: the systick timer

The first attempt was a **hand-port from the PX4 board definition** (pin maps
transcribed from `boards/droneblocks/h743-aio`). It built and the bootloader
*ran* (LED blinked ~5 s) but **USB never enumerated** — the board was invisible
on the bus.

Root cause: the hand-port left the **ChibiOS system-tick timer at its
default**, which collided with the PWM timers (TIM1/3/4) and broke the RTOS
tick that USB enumeration timing depends on. MicoAir explicitly pins it:

```
STM32_ST_USE_TIMER 12
define CH_CFG_ST_RESOLUTION 16
```

Rebasing onto MicoAir (which carries this) fixed USB immediately.

> **Lesson: for a board derived from a known FC, start from the upstream
> ArduPilot hwdef and apply deltas. Do NOT hand-port from the PX4 board def.**
> A red herring along the way was OTG VBUS sensing (`BOARD_OTG_NOVBUSSENS`) —
> it was never the problem; MicoAir works with PA9-as-UART and no VBUS handling.

## Things still to VERIFY on hardware

- **IMU orientation**: we use MicoAir's rotations
  (`BMI088 ROTATION_ROLL_180_YAW_270`, `BMI270 ROTATION_ROLL_180`). These differ
  from the PX4 board's (`-R 6` / `-R 0`). Confirm via accel/level cal — MicoAir's
  are likely correct since it's the same physical board.
- **Battery current scale**: still MicoAir's `14.14`. Verify against the
  DroneBlocks power module.
- `defaults.parm` here is an indoor-optical-flow starting point (quad, no
  GPS/compass, EKF3 flow + rangefinder height), mirroring the PX4 4701 intent —
  **not** a flight tune.
