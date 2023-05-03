# System76 Scheduler

## Changes compared to original version

1. Latest kernel and upower seem to have fixed this issue, thus following changes are reverted now. ~~On my laptop, upower still shows `on-battery = false` when the battery is actually discharing:~~

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
2. Update rust-toolchain version to 1.69.0: my local env uses sparse registry and only latest toolchain supports this.
3. Add "/user.slice/*/session.slice/dbus.service" to session service exclusion, as some GNOME graphical apps (e.g. nautilus, eog) are started in this service.
---

Scheduling service which optimizes Linux's CPU scheduler and automatically assigns process priorities for improved desktop responsiveness. Low latency CPU scheduling will be activated automatically when on AC, and the default scheduling latencies set on battery. Processes are regularly sweeped and assigned process priorities based on configuration files. When combined with [pop-shell](https://github.com/pop-os/shell/), foreground processes and their sub-processes will be given higher process priority.

These changes result in a noticeable improvement in the experienced smoothness and performance of applications and games. The improved responsiveness of applications is most noticeable on older systems with budget hardware, whereas games will benefit from higher framerates and reduced jitter. This is because background applications and services will be given a smaller portion of leftover CPU budget after the active process has had the most time on the CPU.

## Install

Requires dependencies as defined in the [debian/control](./debian/control) file:

- cargo & rustc
- clang
- just
- libclang-dev
- libpipewire-0.3-dev
- pkg-config

Then the included justfile can be used to build and install:

```sh
just execsnoop=$(which execsnoop-bpfcc) build-release
sudo just sysconfdir=/usr/share install
```

## DBus

- Interface: `com.system76.Scheduler`
- Path: `/com/system76/Scheduler`

The `SetForeground(u32)` method can be called to change the active foreground process.

## Scheduler Config

The configuration file is stored at the following locations:

- System: `/etc/system76-scheduler/config.kdl`
- Distribution: `/usr/share/system76-scheduler/config.kdl`

Presence of the system configuration will override the distribution configuration. The documented [default configuration can be found here](./data/config.kdl).

Note that if the `background` and `foreground` assignment profiles are defined, then foreground process management will be enabled. Likewise, if a `pipewire` profile is defined, then pipewire process monitoring will be enabled.

## Process Priority Assignments

In addition to `config.kdl`, additional process scheduling profiles are stored in:

- User-config: `/etc/system76-scheduler/process-scheduler/`
- Distribution: `/usr/share/system76-scheduler/process-scheduler/`

An [example configuration is provided here](./data/pop_os.kdl). It is parsed the same as the assignments and exceptions nodes in the main config, and profiles can inherit values from the previous assignment of the same name.

### Profile

```
assignments {
    {{profile-name}} {{profile-properties}}
}
```

The `profile-name` can refer to any name of your choice. If the name matches a previous assignment, it will inherit the values from that assignment, plus any additional profile properties assigned.

The `profile-properties` may contain any of

- Niceness priority, defined as `nice=-20` through `nice=19`

- A scheduler policy defined as one of:
    - `sched="batch"`
    - `sched="idle"`,
    - `sched="other"`
    - `sched=(fifo)1` through `sched=(fifo)99`
    - `sched=(rr)1` through `sched=(rr)99`

> Realtime scheduler policies assign a priority level between 1 and 99. Higher values have higher priority. It is recommended not to set a higher priority than hardware IRQs (>49)

- An I/O priority defined as one of
    - `io="idle"`
    - `io=(best-effort)0` through `io=(best-effort)7`
    - `io=(realtime)0` through `io=(realtime)7`

> The best-effort and realtime classes have priority levels between 0 and 7, where 7 has the least priority, and 0 is the highest priority

### Assignments

Each child element of a profile defines th process(es) to assign to the profile.

```kdl
{{profile-name}} {{profile-properties}} {
    "/match/by/cmdline" {{profile-properties}}
    match-by-name {{profile-properties}}
    * {{condition-properties}} {{profile-properties}}
}
```

- A node name starting with a `/` is a match by command line path
- A node name otherwise is a match by process name
- `*` matches all processes, used with additional `condition-properties`
    - properties are [wild-match'd](https://github.com/becheran/wildmatch)
    - properties may start with `!` to exclude results matching the condition
    - `cgroup="cgroup-path"` matches processes by a cgroup
    - `parent="name"` matches processes by the process name of the parent


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
