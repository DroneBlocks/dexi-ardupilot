# Build runbook

Builds the ArduPilot bootloader + ArduCopter firmware for `DroneBlocksH743AIO`.

## Prerequisites (macOS, one-time)

- `arm-none-eabi-gcc` (ArduPilot wants 10.x; we used 10.3.1) and `dfu-util`:
  `brew install dfu-util` (+ the ARM toolchain).
- **Python 3.12** — *not* 3.14. See the gotcha below.
- Python build deps for 3.12:
  ```bash
  python3.12 -m pip install --break-system-packages empy==3.3.4 future pexpect dronecan pymavlink pyserial
  ```

## ⚠️ Use python3.12, not python3.14

ArduPilot's in-process MAVLink header codegen **silently no-ops on Python
3.14** (the Homebrew default), producing:

```
fatal error: include/mavlink/v2.0/all/version.h: No such file or directory
```

Build everything with an explicit `python3.12`. If you hit the error after
switching, do a clean rebuild: `python3.12 ./waf distclean` then reconfigure.

## Get the tree

```bash
git clone https://github.com/DroneBlocks/ardupilot -b dexi-h743-aio
cd ardupilot
git submodule update --init --recursive    # ChibiOS, mavlink, etc.
```

The board target is already in-tree at
`libraries/AP_HAL_ChibiOS/hwdef/DroneBlocksH743AIO/` with the id registered in
`Tools/AP_Bootloader/board_types.txt`.

## Build

```bash
# bootloader (one-time DFU artifact)
python3.12 Tools/scripts/build_bootloaders.py DroneBlocksH743AIO
#   -> Tools/bootloaders/DroneBlocksH743AIO_bl.bin
#   (a trailing "intelhex" ModuleNotFoundError on the .hex step is harmless —
#    we flash the .bin over DFU)

# firmware
python3.12 ./waf configure --board DroneBlocksH743AIO
python3.12 ./waf copter
#   -> build/DroneBlocksH743AIO/bin/arducopter.apj   (~1.27 MB, board_id 5290)
```

Flash usage is ~1.42 MB of ~1.79 MB app space (~370 KB headroom).

## Iterating on the board config

Edit the files in `libraries/AP_HAL_ChibiOS/hwdef/DroneBlocksH743AIO/`. To
catch errors fast without a full compile:

```bash
python3.12 ./waf configure --board DroneBlocksH743AIO   # runs the hwdef parser
```

Keep the source-of-truth mirror in `DroneBlocks/dexi-ardupilot`
(`board/DroneBlocksH743AIO/`) in sync with the fork.
