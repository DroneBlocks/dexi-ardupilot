# DEXI-OS on ArduPilot — ROS 2 / DDS

Analysis + the firmware-side enablement for making DEXI-OS talk to ArduPilot.

## The key insight: the transport is shared

ArduPilot's ROS 2 support (`AP_DDS`) uses the **same eProsima Micro-XRCE-DDS
client + `MicroXRCEAgent`** that PX4's uXRCE-DDS uses. DEXI-OS already runs that
agent for PX4. So the DDS layer is **not a new unknown transport** — it's the
same pipe with a different topic graph on top.

## Two layers, two efforts

### MAVLink / GCS path — ~works, low effort
ArduPilot is MAVLink-native:
- **mavlink-router** fans out the FC link unchanged.
- **MAVSDK** supports ArduPilot; curriculum lessons (arm/takeoff/goto) largely work.
- **mavlink2rest / QGC** work.
- Deltas: param **names** differ (ArduPilot `FLTMODE*`/`EK3_*` vs PX4
  `COM_*`/`EKF2_*`) and a few behaviors. Days of mapping, not a rebuild.

### ROS 2 / DDS path — the real work
1. **Enable AP_DDS in firmware** — done (see below).
2. **Reuse the agent** — repoint DEXI-OS's existing `MicroXRCEAgent` at the
   ArduPilot serial port. No new infra.
3. **Re-point the flight nodes** (the actual cost). PX4 path uses
   `px4_msgs`/uORB (`TrajectorySetpoint`, `OffboardControlMode`,
   `VehicleLocalPosition`). ArduPilot exposes **standard ROS types** under `/ap`:
   - `/ap/pose/filtered` (PoseStamped), `/ap/twist/filtered`
   - **`/ap/cmd_vel` (TwistStamped)** for offboard velocity control
   - `/ap/navsat`, `/ap/battery`, `/ap/clock`, a `/takeoff` service, arm/mode services
   - So `dexi_offboard` (PX4 TrajectorySetpoints) needs an ArduPilot variant
     against `cmd_vel` + GUIDED mode + arm/takeoff services. **The velocity
     model maps almost 1:1 onto how we already fly DEXI on flow** — the control
     paradigm transfers; the messages/API change.
   - Drop `px4_msgs`, add `ardupilot_msgs` + `geometry_msgs`. The rosbridge /
     browser layer sits above and is reusable once nodes are repointed.

### Separate work
- **Indoor flow / EKF** = ArduPilot EKF3 re-tune (`EK3_SRC*`, `FLOW_TYPE`) —
  doesn't carry over from PX4 EKF2.
- **DEXI-OS image** = an "ardupilot platform" overlay in the bringup config.

## Rough effort
- MAVLink layer: **days**.
- ROS 2 flight-node port: **~1–2 weeks** focused (offboard velocity flight +
  pose read, then re-validate flow). The long pole.

## Firmware enablement (done)

`AP_DDS` is compiled into our board by a per-board define in the hwdef:

```
# libraries/AP_HAL_ChibiOS/hwdef/DroneBlocksH743AIO/hwdef.dat
define AP_DDS_ENABLED 1
```

(The waf `--enable-DDS` flag has a quirk; the hwdef define is the canonical,
durable per-board enable — `Tools/ardupilotwaf/boards.py` reads it from the
generated `hwdef.h`.)

`defaults.parm` puts DDS on **UART4 (SERIAL4)** at 115200, separate from the
TELEM1 MAVLink companion link:

```
SERIAL4_PROTOCOL 45    # DDS (SerialProtocol_DDS_XRCE)
SERIAL4_BAUD 115
```

## The de-risking probe

Before porting any nodes, prove the transport end-to-end on real hardware:

1. Flash the DDS-enabled `arducopter.apj`.
2. Wire the companion (or a USB-serial adapter) to **UART4 (PA0/PA1)**.
3. Run the micro-ROS agent:
   ```bash
   MicroXRCEAgent serial --dev /dev/ttyUSB0 -b 115200
   # (or the DEXI-OS agent container, pointed at the UART4 device)
   ```
4. From a ROS 2 env:
   ```bash
   ros2 topic list          # expect /ap/pose/filtered, /ap/clock, /ap/battery, ...
   ros2 topic echo /ap/clock
   ```

Seeing the `/ap/*` topics confirms the whole DDS path on our board and kills the
"uXRCE-DDS unknown" before committing to the node port.
