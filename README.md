# System76 Scheduler

## Changes compared to original version

1. On my laptop, upower still shows `on-battery = false` when the battery is actually discharing:

    ```
    Device: /org/freedesktop/UPower/devices/DisplayDevice
      power supply:         yes
      updated:              Wed 14 Sep 2022 01:21:38 AM CST (25 seconds ago)
      has history:          no
      has statistics:       no
      battery
          present:             yes
          state:               discharging
          warning-level:       none
          energy:              37.44 Wh
          energy-full:         51.18 Wh
          energy-rate:         13.594 W
          charge-cycles:       N/A
          time to empty:       2.8 hours
          percentage:          73%
          icon-name:          'battery-full-symbolic'

    Daemon:
      daemon-version:  0.99.20
      on-battery:      no
      lid-is-closed:   no
      lid-is-present:  yes
      critical-action: HybridSleep
    ```

    In this modified version state of DisplayDevice is used instead. When `state == Discharing` s76-scheduler will consider it as on battery and use default profile for CFS.

    According to source code of upower (up-daemon.c), this is what it should actually do:

    ```c
    /**
    * up_daemon_get_on_battery_local:
    *
    * As soon as _any_ battery goes discharging, this is true
    **/
    static gboolean
    up_daemon_get_on_battery_local (UpDaemon *daemon)
    {
        /* Use the cached composite state cached from the display device */
        return daemon->priv->state == UP_DEVICE_STATE_DISCHARGING;
    }
    ```

    So this change should make sense, though may not suitable for all laptops. And also notice that it will NOT turn to discharing state immediately after unplugging: wait for a few seconds to see it.
---

Scheduling service which optimizes Linux's CPU scheduler and automatically assigns process priorities for improved desktop responsiveness. Low latency CPU scheduling will be activated automatically when on AC, and the default scheduling latencies set on battery. Processes are regularly sweeped and assigned process priorities based on configuration files. When combined with [pop-shell](https://github.com/pop-os/shell/), foreground processes and their sub-processes will be given higher process priority.

These changes result in a noticeable improvement in the experienced smoothness and performance of applications and games. The improved responsiveness of applications is most noticeable on older systems with budget hardware, whereas games will benefit from higher framerates and reduced jitter. This is because background applications and services will be given a smaller portion of leftover CPU budget after the active process has had the most time on the CPU.

## DBus

- Interface: `com.system76.Scheduler`
- Path: `/com/system76/Scheduler`

The `SetForeground(u32)` method can be called to change the active foreground process.

## Scheduler Config

The configuration file is stored at the following locations:

- User-config: `/etc/system76-scheduler/config.ron`
- Distribution: `/usr/share/system76-scheduler/config.ron`

```rs
{
    // The priority to assign background tasks.
    background: Some(5),

    // The priority to assign foreground tasks.
    foreground: Some(-5),
}
```

- When `background` is set to `None`, background process priorities will not be assigned.
- When `foreground` is set to `None`, foreground and background priorities will not be assigned.

## Process Priority Assignments

RON configuration files are stored at the following locations:

- User-config: `/etc/system76-scheduler/assignments/`
- Distribution: `/usr/share/system76-scheduler/assignments/`

They define the default priorities for processes scanned. The syntax of `.ron` configuration files in these directories is as follows:

```rs
{
    (CPU_PRIORITY, IO_PRIORITY): [
        "exe_name1",
        "exe_name2"
    ],
    CPU_PRIORITY: [
        "exe_name3",
        "exe_name4"
    ],
    IO_PRIORITY: [
        "exe_name5"
    ]
}
```

Where:

- `CPU_PRIORITY` is a number between `-20` and `19`
- `IO_PRIORITY` is one of:
    - `Idle`
    - `Standard`
    - `BestEffort(PRIORITY_LEVEL)`
    - `Realtime(PRIORITY_LEVEL)`
- `PRIORITY_LEVEL` is a number between `0` and `7`


A real world example below:

```rs
{
    // Very high
    (-9, BestEffort(0)): [
        "easyeffects",
    ],
    // High priority
    (-5, BestEffort(4)): [
        "gnome-shell",
        "kwin",
        "Xorg"
    ],
    // Default
    0: [ "dbus", "dbus-daemon", "systemd"],
    // Absolute lowest priority
    (19, Idle): [
        "c++",
        "cargo",
        "clang",
        "cpp",
        "g++",
        "gcc",
        "lld",
        "make",
        "rustc",
    ]
}
```

## Process Priority Exceptions

RON configuration files are stored at the following locations:

- User-config: `/etc/system76-scheduler/exceptions/`
- Distribution: `/usr/share/system76-scheduler/exceptions/`

The files contain a list of process names that are prohibited from having priority adjustments.

```rs
([
"pipewire",
"pipewire-pulse",
"wireplumber"
])
```

## CPU Scheduler Latency Configurations

### Default

The default settings for CFS by the Linux kernel. Achieves a high level of throughput for CPU-bound tasks at the cost of increased latency for inputs. This setting is ideal for servers and laptops on battery, because low-latency scheduling sacrifices some energy efficiency for improved responsiveness.

```yaml
latency: 6ns
minimum_granularity: 0.75ms
wakeup_granularity: 1.0ms
bandwidth_size: 5us
```

### Responsive

Slightly reduces time given to CPU-bound tasks to give more time to other processes, particularly those awaiting and responding to user inputs. This can significantly improve desktop responsiveness for a slight penalty in throughput on CPU-bound tasks.

```yaml
latency: 4ns
minimum_granularity: 0.4ms
wakeup_granularity: 0.5ms
bandwidth_size: 3us
```

## License

Licensed under the [Mozilla Public License 2.0](https://choosealicense.com/licenses/mpl-2.0/). Permissions of this copyleft license are conditioned on making available source code of licensed files and modifications of those files under the same license (or in certain cases, one of the GNU licenses). Copyright and license notices must be preserved. Contributors provide an express grant of patent rights. However, a larger work using the licensed work may be distributed under different terms and without source code for files added in the larger work.

### Contribution

Any contribution intentionally submitted for inclusion in the work by you shall be licensed under the Mozilla Public License 2.0 (MPL-2.0). It is required to add a boilerplate copyright notice to the top of each file:

```rs
// Copyright {year} {person OR org} <{email}>
// SPDX-License-Identifier: MPL-2.0
```
