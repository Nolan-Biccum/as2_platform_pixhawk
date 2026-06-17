# PX4 message compatibility (this fork)

This fork exists to fix `src/pixhawk_platform.cpp` so it builds and runs
correctly against current `px4_msgs` (pinned here to `v1.17.0`, used with
PX4-Autopilot `v1.17.0`). Upstream `as2_platform_pixhawk` still targets the
pre-2022 PX4 message layout, from before `px4_msgs`' "overhaul corresponding
with new PX4-Autopilot microdds_client" rework
([px4_msgs#15](https://github.com/PX4/px4_msgs/pull/15), first released in
`px4_msgs` `v1.13.0`). None of the changes below are specific to PX4 1.17 —
they're long-overdue fixes for a message-format change from ~2022 that
upstream never picked up — but they only became build-blocking once this
project pinned `px4_msgs` to a version that actually matches current PX4
firmware (`v1.17.0`), so that's the migration that surfaced them.

## Changes

### 1. `VehicleAttitudeSetpoint`: removed `pitch_body` / `roll_body` / `yaw_body`

`resetAttitudeSetpoint()` assigned `NAN` to
`px4_attitude_setpoint_.pitch_body/.roll_body/.yaw_body`. Current
`VehicleAttitudeSetpoint.msg` only has `q_d` (quaternion) and `thrust_body` —
the separate Euler-angle fields don't exist anymore. These were dead
assignments to nonexistent struct members; removed.

### 2. `SensorGps`: renamed fields, no longer scaled integers

Old code read `msg->lat` / `msg->lon` / `msg->alt_ellipsoid` and then divided
by `1e7` / `1e7` / `1e3` to convert from PX4's old `int32` 1e7-scaled-degrees
/ millimetre representation into the `float64` degrees/metres
`sensor_msgs::NavSatFix` expects.

Current `SensorGps.msg` already stores these as `float64` degrees/metres
directly, under renamed fields: `latitude_deg`, `longitude_deg`,
`altitude_ellipsoid_m` (also `altitude_msl_m`, unused here). The platform now
reads the renamed fields directly and the `/1e7` / `/1e3` conversion is
deleted — applying it to an already-unscaled float64 value silently produced
garbage GPS positions (this was the original bug that prompted this fix:
EKF2/AS2 saw a valid-looking but wrong GPS fix, not an obvious error).

### 3. `BatteryStatus`: removed `design_capacity` / `serial_number`

Both fields no longer exist on `BatteryStatus.msg`. `design_capacity` is now
left as `NAN` (matches the pattern already used for `charge` just above it in
the same function) and `serial_number` as an empty string, instead of
referencing nonexistent struct members.

## Context

Made while migrating RADSwarm (a multi-drone SITL search-and-rescue swarm
project) from PX4 `v1.15.4` to `v1.17.0`. This file covers only the
`as2_platform_pixhawk`-side fix.
