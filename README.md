## AMDGPU Clocks

### Introduction

This is a simple script that can be used to set custom power states for recent
AMD GPUs that are driven by **amdgpu** Linux kernel driver. The script is able
to set custom clocks, voltages and some other power states, assuming that
Radeon OverDrive is enabled in kernel boot opions. The OverDrive is not enabled
by default as of Linux 5.x, it can be enabled by setting 14th bit (0x4000) of a
**ppfeaturemask** amdgpu driver to 1. For example, setting
**amdgpu.ppfeaturemask=0xfffd7fff** or **amdgpu.ppfeaturemask=0xffffffff**
kernel boot option will do the trick. This would enable amdgpu driver sysfs
API that allows fine grain control of GPU power states (GPU & VRAM clocks &
voltages, depending on the actual hardware).
It should work on Polaris, Vega (unfortunately Vega found on AMD APUs does
not expose this API), Navi and RDNA cards, and it can be used to easily manage
multiple AMD graphics cards.

### How does it work

By default, custom power states for a particular GPU can be defined in
`/etc/default/amdgpu-custom-state.cardX` file, which is expected to be created
by user, and where X corresponds to a card in `/sys/class/drm/cardX`. The
custom state file have same format as the actual
`/sys/class/drm/cardX/device/pp_od_clk_voltage`, with option to add
newlines, comments (lines starting with `#`) and few additional parameters used
to set extra power state parameters. These are **FORCE_SCLK** & **FORCE_MCLK**,
that can be used to limit GPU and memory power states to a particular subset
of states, **FORCE_POWER_CAP** that can be used to set desired power cap,
**FORCE_PERF_LEVEL** that can be used to force desired
`power_dpm_force_performance_level` for a card (which can be `auto`, `low`,
`high`, `manual`, etc) and **FORCE_POWER_PROFILE** used for manually setting
the profile found in card's `pp_power_profile_mode`.

Since some Linux kernel versions are known to enumerate the very same card to
a different `cardX` identifier (X randomly toggles between 0 or 1 on reboot),
one can alternatively define the custom state using
`/etc/default/amdgpu-custom-state.pci:xxxx:xx:xx.x` file instead, where
`xx:xx.x` corrsponds to intended card's PCI `<domain>:<bus>:<dev>.<function>`
numbers. For example `/etc/default/amdgpu-custom-state.pci:0000:03:00.0`.

Here is an example how custom power state file may look like for Polaris cards:

```shell
$ cat /etc/default/amdgpu-custom-states.card0
# Set custom GPU states 6 & 7:
OD_SCLK:
6:       1000MHz        860mV
7:       1050MHz        890mV
# Set custom memory states 1 & 2:
OD_MCLK:
1:        900MHz        800mV
2:       1600MHz        900mV
# Only allow SCLK states 5, 6 & 7:
FORCE_SCLK: 5 6 7
# Force fixed memory state:
FORCE_MCLK: 2
# Force power limit (in micro watts):
FORCE_POWER_CAP: 90000000
# In order to allow FORCE_SCLK & FORCE_MCLK:
FORCE_PERF_LEVEL: manual
```

Here is an example how custom power state file may look like for Navi cards:
```
# For Navi (and Radeon7) we can only set highest SCLK & MCLK, "state 1":
OD_SCLK:
1: 1550MHz
OD_MCLK:
1: 750MHz
# More fine-grain control of clocks and voltages are done with VDDC curve:
OD_VDDC_CURVE:
0: 800MHz @ 800mV
1: 1125MHz @ 820mV
2: 1550MHz @ 850mV
# Force power limit (in micro watts):
FORCE_POWER_CAP: 87000000
FORCE_PERF_LEVEL: manual
```

Here is an example how custom power state file may look like for RDNA2 cards:
```
OD_VDDGFX_OFFSET:
-75mV
FORCE_PERF_LEVEL: manual
FORCE_POWER_CAP: 99000000
```

With RDNA4, `pp_od_clk_voltage` exposes two `SCLK` offsets but only `1`
(max `SCLK` offset) can be adjusted. Example of custom power state file:
```
OD_SCLK_OFFSET:
1: 200Mhz
OD_MCLK:
0: 97MHz
1: 2519MHz
OD_VDDGFX_OFFSET:
-60mV
FORCE_POWER_CAP: 25000000
FORCE_PERF_LEVEL: manual
# compute
FORCE_POWER_PROFILE: 5
```

### Installing and manually running the script

Simply place the [script](amdgpu-clocks) in `/usr/local/bin/amdgpu-clocks`:
```shell
$ sudo ln -s $(pwd)/amdgpu-clocks /usr/local/bin/amdgpu-clocks
```

and specify custom power states in `/etc/default/amdgpu-custom-states.card0`:
```shell
$ sudo amdgpu-clocks

Detecting the state values at /sys/class/drm/card0/device/pp_od_clk_voltage:
    SCLK state 0: 700Mhz
    SCLK state 1: 2539Mhz
    MCLK state 0: 97Mhz
    MCLK state 1: 1000MHz
    VDD GFX Offset: 0mV
    Maximum clocks & voltages:
    SCLK clock 3150Mhz
    MCLK clock 1200Mhz
    Curent power cap: 130W
Verifying user state values at /etc/default/amdgpu-custom-state.card0:
    VDD GFX Offset: -75mV
    Force performance level to manual
    Force power cap to 99W
Committing custom states to /sys/class/drm/card0/device/pp_od_clk_voltage:
    Done
```

The script can also be invoked with specific custom state file prefix (can be
a directory, case trailing slash is supplied), for example:
```shell
$ sudo USER_STATES_PATH=custom-states amdgpu-clocks
```

This will load and apply custom states from all `custom-states.card*` files
in the current directory. Script can also be used with an additional 'restore'
parameter that can be used to restore all states to the initial defaults
(states before script was executed for the first time), with exception of
`pp_power_profile_mode` which will be set to 0.

### Making the custom power states persist

This can be achieved by placing provided *systemd* service
[file](amdgpu-clocks.service) in `/etc/systemd/system/` directory,
and enable it:
```shell
$ sudo systemctl enable --now amdgpu-clocks
```

However, if your system goes to suspend state, the above service will not
auto-restart (due to RemainAfterExit parameter). To fix that copy provided
[file](amdgpu-clocks-resume) into `/usr/lib/systemd/system-sleep`
```shell
$ cp amdgpu-clocks-resume /usr/lib/systemd/system-sleep/
```

Of course, one should not forget to place the actual custom power states in
`/etc/default/amdgpu-custom-state.cardX`.
