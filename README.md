# dexi-ardupilot

Running **ArduPilot** on the DroneBlocks **DEXI-3** flight controller (the
DroneBlocks H743-AIO), as a firmware-agnostic alternative to our PX4 stack.

> **Status (2026-06-19): WORKING.** ArduCopter 4.8.0 builds for our board,
> flashes, boots, and talks MAVLink over USB on real hardware
> (`HEARTBEAT autopilot=ARDUPILOTMEGA, type=QUADROTOR`). The board is *proven
> to run ArduPilot*; it is **not yet validated for flight** — see
> [docs/bringup.md](docs/bringup.md).

## Why this exists

A customer asked whether the DEXI-3 could run ArduPilot instead of PX4. It can.
The board is a fork of the **MicoAir H743-AIO**, which ArduPilot already
supports first-class — so the hardware is a clean ArduPilot target.

**Strategy: be firmware-agnostic at the hardware layer, opinionated at the
product layer.** The board running either stack is a real selling point and
costs little. Our DEXI curriculum / DEXI-OS / web-configurator stay
PX4-specific (that's the moat) unless a paying customer needs ArduPilot parity.

## What's here

| Path | What |
|---|---|
| `board/DroneBlocksH743AIO/` | The ArduPilot board target (source-of-truth mirror): `hwdef.dat`, `hwdef-bl.dat`, `defaults.parm` |
| `artifacts/` | Built `DroneBlocksH743AIO_bl.bin` (bootloader) + `arducopter.apj` (firmware). Both reproducible from the fork. |
| `docs/board-config.md` | The board definition explained — MicoAir lineage, the deltas, and the systick gotcha that cost us the day |
| `docs/build.md` | How to build the bootloader + firmware |
| `docs/flash.md` | How to flash a board (and why this is **not** the `droneblocks-mavlink-tool` path) |
| `docs/bringup.md` | First-boot / bench bring-up VERIFY checklist |

## The repos

- **`DroneBlocks/ardupilot`** (fork of `ArduPilot/ardupilot`), branch
  **`dexi-h743-aio`** — the buildable ArduPilot tree with our board target
  committed in `libraries/AP_HAL_ChibiOS/hwdef/DroneBlocksH743AIO/` +
  `Tools/AP_Bootloader/board_types.txt` (id **5290**, `AP_HW_DroneBlocksH743AIO`).
- **`DroneBlocks/dexi-ardupilot`** (this repo) — documentation, runbooks, and a
  source-of-truth mirror of the board files + built artifacts.

The board files live in both places: authoritative for *building* is the fork;
this repo mirrors them so the docs and the config never drift apart and nothing
is lost.

## TL;DR build + flash

```bash
# build (from the fork) — MUST use python3.12, NOT 3.14
git clone https://github.com/DroneBlocks/ardupilot -b dexi-h743-aio
cd ardupilot && git submodule update --init --recursive
python3.12 Tools/scripts/build_bootloaders.py DroneBlocksH743AIO
python3.12 ./waf configure --board DroneBlocksH743AIO && python3.12 ./waf copter

# flash a SPARE board (overwrites PX4; reversible)
# 1) DFU: hold BOOT + plug USB  -> 0483:df11
dfu-util -a 0 -d 0483:df11 -s 0x08000000:leave -D Tools/bootloaders/DroneBlocksH743AIO_bl.bin
# 2) upload firmware over the new bootloader
python3.12 Tools/scripts/uploader.py build/DroneBlocksH743AIO/bin/arducopter.apj
```

See `docs/` for the full detail and the hard-won gotchas.

## Relationship to PX4 / droneblocks-mavlink-tool

This is a **separate toolchain** from our PX4 work. `droneblocks-mavlink-tool`
(flash_new_fc.py, params.py, etc.) is PX4-only and was **not** used to flash
ArduPilot — see [docs/flash.md](docs/flash.md). To put a board back on PX4, use
`droneblocks-mavlink-tool/flash_new_fc.py` over DFU.
