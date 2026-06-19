# Flash runbook

How we flashed ArduPilot onto a DEXI-3 board on 2026-06-19.

## ⚠️ This is NOT the droneblocks-mavlink-tool path

`droneblocks-mavlink-tool` (`flash_new_fc.py`, `params.py`, `provision_dexi3_flow.py`)
is **PX4-only** — it flashes the PX4 bootloader + `.px4` firmware + PX4 params.
It was **not** used to flash ArduPilot.

The ArduPilot flash is its own two-tool path:

| Step | Tool | Artifact |
|---|---|---|
| Bootloader | `dfu-util` (over USB DFU) | `DroneBlocksH743AIO_bl.bin` |
| Firmware | ArduPilot `Tools/scripts/uploader.py` (python3.12) | `arducopter.apj` |

The only use of `droneblocks-mavlink-tool` in this whole effort was a **PX4
param backup** (`dump_params.py`) before wiping the board — saved at
`droneblocks-mavlink-tool/logs/pre-ardupilot-backup_20260619.params`.

## Before you flash

- **Use a SPARE / dev board.** Flashing ArduPilot **overwrites PX4**.
- **Fully reversible:** to restore PX4, enter DFU and run
  `droneblocks-mavlink-tool/flash_new_fc.py`.
- **No props.** USB power is enough for flashing + bench bring-up.
- You can never brick it: **holding BOOT on plug-in always forces the STM32 ROM
  DFU** regardless of what's in flash.

## Step 1 — bootloader over DFU

```bash
# Enter DFU: hold the BOOT button while plugging in USB. Enumerates as 0483:df11.
dfu-util -l | grep df11   # confirm

dfu-util -a 0 -d 0483:df11 -s 0x08000000:leave \
  -D Tools/bootloaders/DroneBlocksH743AIO_bl.bin
```

- `Error during download get_status` on the `:leave` is **benign** as long as
  `File downloaded successfully` printed.

## Step 2 — firmware over the new bootloader

```bash
python3.12 Tools/scripts/uploader.py build/DroneBlocksH743AIO/bin/arducopter.apj
```

The uploader waits for the board's bootloader port. **Unplug, wait ~3 s, replug
normally (no BOOT)** and it grabs the bootloader window and flashes. Successful
run shows `board_type: 5290`, then `Erase 100% -> Program 100% -> Verify 100%
-> Rebooting.`

## Step 3 — confirm it's running ArduPilot

After flashing, **leave the board plugged in ~10 s**, then connect. The board
re-enumerates as a serial port (e.g. `/dev/cu.usbmodem101`):

```bash
python3.12 - <<'PY'
import glob, time
from pymavlink import mavutil
p = sorted(glob.glob('/dev/cu.usbmodem*'))[0]
m = mavutil.mavlink_connection(p, baud=115200)
hb = m.wait_heartbeat(timeout=8)
print("autopilot", hb.autopilot, "(3 = ArduPilot)", "type", hb.type)
PY
```

Expected: `autopilot 3 (ARDUPILOTMEGA), type 2 (QUADROTOR)`.

Or just connect **Mission Planner / QGroundControl** to the serial port.

### Gotchas
- **Don't poll for the USB port during the bootloader→app handoff** — you'll
  catch the bootloader's brief window and get `[Errno 6] Device not configured`.
  Leave it plugged and connect to the stable app port.
- `uploader.py` needs `pyserial` installed for python3.12.

## Restore PX4 (undo)

```bash
# enter DFU (hold BOOT + plug), then:
cd ~/_dev/droneblocks-mavlink-tool && ./venv/bin/python flash_new_fc.py
```
