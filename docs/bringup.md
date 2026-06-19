# Bench bring-up checklist

The board **runs** ArduPilot but is **not yet validated for flight**. Work this
list propellerless, over USB (Mission Planner / QGC), before any tethered hover.

Each item is a `VERIFY` from the board port — most are inherited from MicoAir
and just need confirming on our exact unit.

- [ ] **Connects** — Mission Planner/QGC shows ArduCopter, board "DroneBlocksH743AIO".
- [ ] **IMUs** — both BMI088 + BMI270 detected (`INS_*`); run **gyro + accel +
      level cal**. Confirm attitude tracks reality; fix `AHRS_ORIENTATION` /
      IMU rotations if the horizon is wrong. (We currently use MicoAir's IMU
      rotations, which differ from the PX4 board's — this is the main thing to
      confirm.)
- [ ] **Baro** — DPS310 altitude sane and responsive.
- [ ] **Battery** — voltage reads correctly (`BATT_VOLT_MULT`, divider 11.0);
      tune current scale (MicoAir default 14.14) against the DroneBlocks PM.
- [ ] **Motors** — propellerless `MOTORTEST`: confirm motor numbering maps to
      the right arms and spin directions; set `FRAME_CLASS`/`FRAME_TYPE` and
      reversals.
- [ ] **RC** — bind ELRS/CRSF on USART2 (SERIAL5, proto RCIN); sticks move the
      right axes; set up a flight-mode switch.
- [ ] **SD logging** — confirm a `.bin` dataflash log is written (validates the
      SDMMC pins).
- [ ] **Flow + range** — the optical-flow sensor on the flow port: `FLOW_TYPE`,
      `RNGFND1_TYPE` report data. (Getting both flow and range up on ArduPilot
      needs a small sensor-integration step — deferred, see the README note.
      ArduPilot EKF3 indoor flow is also a separate tuning exercise from PX4's
      EKF2 — the PX4 flow work does not transfer.)

Only after all of the above passes is a tethered hover warranted. The PX4 rate
tune does **not** carry over — ArduPilot needs its own tune.
