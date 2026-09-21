# CoolerControl on MOS: PWM fan control with hwmon

CoolerControl can detect hardware sensors through Linux `hwmon`, but **fan control requires write access to the specific hwmon device that exposes the PWM channels**.

For security and portability, the MOS template should keep `Privileged` mode disabled and avoid hard-coding a motherboard-specific device path.

## After installation

1. Open **CoolerControl → Devices**.
2. Find the hardware device that exposes your fan channels (`fan1`, `fan2`, `fan3`, etc.).
3. Note the device's **real sysfs location** shown by CoolerControl.

Example from an NCT6779 controller using the `nct6775` kernel driver:

```text
/sys/devices/platform/nct6775.656
```

> **Do not copy this example path blindly.**  
> The correct path depends on your motherboard, Super-I/O controller and kernel driver.

## Add the fan-control device as read/write

Edit the CoolerControl container in MOS and add the real sysfs device path as an additional bind mount.

Example:

```text
Name:
Fan Controller

Host:
/sys/devices/platform/nct6775.656

Container:
/sys/devices/platform/nct6775.656

Mode:
rw
```

Keep:

```text
Privileged: OFF
```

This gives CoolerControl write access only to the fan controller instead of granting broad privileged access to the host.

## Why this is needed

Mounting `/sys/class/hwmon` is useful for discovery and monitoring, but those entries are normally links to the real hardware device under `/sys/devices`.

As a result, CoolerControl may be able to display:

- temperatures;
- fan RPM;
- PWM percentages;
- the detected Super-I/O controller;

while still failing when it tries to switch a fan into manual PWM control.

A typical error during calibration is:

```text
Unable to set pwm1_enable to 1
Read-only file system (os error 30)
```

Adding the **real device path** as `rw` resolves this without requiring `Privileged` mode.

## Verify write access

After MOS recreates the container, verify that the required PWM control files are writable.

Example for fan channels `1`, `2`, `4` and `5`:

```bash
docker exec CoolerControl sh -lc 'for n in 1 2 4 5; do for f in /sys/devices/platform/nct6775.656/hwmon/hwmon*/pwm${n}_enable; do printf "%s: " "$f"; [ -w "$f" ] && echo WRITE_OK || echo READ_ONLY; done; done'
```

Expected result:

```text
.../pwm1_enable: WRITE_OK
.../pwm2_enable: WRITE_OK
.../pwm4_enable: WRITE_OK
.../pwm5_enable: WRITE_OK
```

If the result is `READ_ONLY`, check that:

- you used the correct real sysfs path for your hardware;
- the additional path is mounted with `rw` access;
- MOS recreated or restarted the container after the change.

## Template recommendation

A public MOS template should **not** hard-code a path such as:

```text
/sys/devices/platform/nct6775.656
```

That path is specific to one hardware/driver combination.

The template should provide the generic CoolerControl container configuration, while this document explains how each user can identify and add their own writable fan-controller path.

This keeps the template portable across different motherboards and avoids using `Privileged` mode unnecessarily.
