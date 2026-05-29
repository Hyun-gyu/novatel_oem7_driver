# NovAtel OEM7 Driver Entrypoint

Audience: agent

This repository is the ROS 2 Humble driver snapshot used by the field_dataset
project for NovAtel OEM7 / PwrPak7 receiver monitoring. If Codex starts in this
repository, read this file first.

## Project Role

This repo provides the ROS driver and message packages only:

- `novatel_oem7_driver`
- `novatel_oem7_msgs`

It does not, by itself, turn a receiver into an RTK base station. Base-station
operation also requires receiver-side NovAtel commands for antenna position,
ICOM port configuration, and RTCM output.

In the field_dataset project, this repo is used as a git submodule at:

```text
src/sensor_driver/novatel_oem7_driver
```

When working inside the parent `field_dataset` repo, also read:

```text
../../../docs/field_laptop/NOVATEL_OEM7_BASE_STATION_SETUP.md
../../../reports/field_laptop/2026-05-13__novatel_pwrpak7_indoor_monitoring_check.md
../../../config/novatel_oem7_pwrpak7_gnss_only_override.yaml
```

If those paths do not exist, you are probably in a standalone clone of this
driver repo. In that case, treat this file as the driver-level starting point
and ask the operator for the parent project runbook.

## Current Field Assumptions

The field_dataset operating split is:

- `ICOM1` / TCP `3001`: intended RTCM correction output to rover.
- `ICOM2` / TCP `3002`: preferred monitoring and command path for this ROS
  driver.
- PwrPak7 router LAN IP may differ by setup. Recent field plans use
  `192.168.0.2`; older indoor tests used `192.168.1.10`.
- Do not mix subnets. Confirm router LAN, laptop IP, and PwrPak7 IP before
  launching the driver.

Known status from field_dataset notes:

- ROS monitoring through `ICOM2:3002` has worked with the GNSS-only override.
- RTCM byte output on `ICOM1:3001` must be verified before claiming rover RTK.
- The main dataset acquisition should not be blocked by RTK bringup unless the
  operator explicitly makes RTK the primary goal.

## Fresh Clone Setup

From the parent `field_dataset` repo:

```bash
cd /home/hyungyu/field_dataset
git submodule update --init --recursive src/sensor_driver/novatel_oem7_driver

source /opt/ros/humble/setup.bash
rosdep install --from-paths src/sensor_driver/novatel_oem7_driver/src --ignore-src -r -y
colcon build --symlink-install --packages-select novatel_oem7_msgs novatel_oem7_driver
source install/setup.bash
```

From a standalone clone of this repo:

```bash
cd /path/to/novatel_oem7_driver
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-select novatel_oem7_msgs novatel_oem7_driver
source install/setup.bash
```

## Field Monitoring Launch

Use `ICOM2:3002` for driver monitoring when `ICOM1:3001` is reserved for RTCM.

Parent field_dataset checkout example:

```bash
cd /home/hyungyu/field_dataset
source /opt/ros/humble/setup.bash
source install/setup.bash

NOVATEL_OEM7_DRIVER_PARAM_OVERRIDES_PATH="$PWD/config/novatel_oem7_pwrpak7_gnss_only_override.yaml" \
ros2 launch novatel_oem7_driver oem7_net.launch.py \
  oem7_if:=Oem7ReceiverTcp \
  oem7_ip_addr:=192.168.0.2 \
  oem7_port:=3002
```

If the PwrPak7 is still using the older indoor-test address, replace
`192.168.0.2` with `192.168.1.10`.

## Expected ROS Topics

The driver namespace is `/novatel/oem7`. Useful checks:

```bash
ros2 topic list | grep /novatel/oem7
ros2 topic hz /novatel/oem7/bestpos
ros2 topic echo --once /novatel/oem7/bestpos
ros2 topic echo --once /novatel/oem7/rxstatus
ros2 topic echo --once /novatel/oem7/time
```

Expected key topics include:

```text
/novatel/oem7/fix
/novatel/oem7/gps
/novatel/oem7/bestpos
/novatel/oem7/bestvel
/novatel/oem7/bestgnsspos
/novatel/oem7/bestgnssvel
/novatel/oem7/rxstatus
/novatel/oem7/time
/novatel/oem7/oem7raw
```

For troubleshooting and post-processing, record `/novatel/oem7/oem7raw` when
possible.

## RTK Correction Verification

Do not claim RTK is working just because the ROS driver is publishing topics.
Verify the correction stream separately.

From the base-side laptop:

```bash
nc -vz <pwrpak7_lan_ip> 3001
timeout 10 nc <pwrpak7_lan_ip> 3001 | xxd -g1 -l 64
```

For RTCM3, the stream should contain binary frames that commonly begin with
`d3`.

If the rover connects through a router public/static IP, repeat the same test
from the rover-side network:

```bash
nc -vz <router_public_or_static_ip> 3001
timeout 10 nc <router_public_or_static_ip> 3001 | xxd -g1 -l 64
```

Rover-side success requires more than open TCP:

- correction bytes visible,
- rover configured to consume the correction source,
- rover position type changes toward differential / RTK float / RTK fixed,
- rover trajectory quality improves in the expected direction.

## Receiver-Side Base Commands

Exact receiver commands depend on the confirmed antenna position and selected
ICOM output port. Do not save temporary base coordinates.

Typical base correction message set:

```text
RTCM1005 or RTCM1006: base station position
RTCM1077: GPS MSM7
RTCM1087: GLONASS MSM7
RTCM1097: Galileo MSM7
RTCM1127: BeiDou MSM7
RTCM1230: GLONASS code-phase biases
```

Common manual pattern after the base position is known:

```text
UNLOGALL ICOM1
LOG ICOM1 RTCM1006 ONTIME 10
LOG ICOM1 RTCM1077 ONTIME 1
LOG ICOM1 RTCM1087 ONTIME 1
LOG ICOM1 RTCM1097 ONTIME 1
LOG ICOM1 RTCM1127 ONTIME 1
LOG ICOM1 RTCM1230 ONTIME 10
```

Some PwrPak7 configurations may provide a helper command such as:

```text
generatertkcorrections msm7 icom1
```

After any command path, run `LOG LOGLISTA ONCE` or inspect the NovAtel
Application Suite terminal to confirm the actual logs enabled on `ICOM1`.

Only run this after rover RTK validation succeeds:

```text
SAVECONFIG
```

## Field-Day Decision Rule

For field_dataset collection, keep these goals separate:

1. Main acquisition: Stereo RGB + synchronizer + rover GPS + monitor + rosbag.
2. Optional RTK track: base router + PwrPak7 + RTCM stream + rover correction.

If RTK is not validated within the planned debug window, continue main
acquisition and record the RTK state as an experimental note. Do not block the
main 10 Hz dataset run solely on base-station bringup.

## Do Not Do

- Do not use `ICOM1:3001` for ROS monitoring if it is supposed to feed rover
  RTCM.
- Do not assume router port forwarding works until tested from the rover-side
  network.
- Do not publish or commit router passwords, public IPs that should remain
  private, or temporary receiver credentials.
- Do not run `SAVECONFIG` after approximate or unverified base coordinates.
- Do not edit parent field_dataset runbooks from this submodule unless the task
  explicitly includes the parent repo update.
