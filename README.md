## AMDGPU Clocks

### Introduction

This is a simple script that can be used to set custom power states for recent
AMD GPUs that are driven by **amdgpu** Linux kernel driver. The script is able
to set custom clocks, voltages and some other power states, assuming that
Radeon OverDrive is enabled in the driver. The OverDrive is not enabled by
default as of Linux 5.0, it can be enabled by setting 14th bit (0x4000) of a
**ppfeaturemask** amdgpu driver to 1. For example, setting
**amdgpu.ppfeaturemask=0xfffd7fff** or **amdgpu.ppfeaturemask=0xffffffff**
kernel boot option will do the trick. This would enable amdgpu driver sysfs
API that allows fine grain control of GPU power states (GPU & VRAM clocks &
voltages, depending on actual hardware).
It should work on Polaris, Vega (unfortunately Vega found on AMD APUs does
not expose this API) and Navi cards, and it can be used to easily manage
multiple AMD graphics cards.

### How does it work

By default, custom power states are defined in files that are expected to
reside in `/etc/default/amdgpu-custom-state.cardX`, where X corresponds to a
card in `/sys/class/drm/cardX`. The custom state files have same format as the
actual `/sys/class/drm/cardX/device/pp_od_clk_voltage`, with option to add
newlines, comments (lines starting with `#`) and three extra parameters, used
to set extra power state details. These are **FORCE_SCLK** & **FORCE_MCLK**,
that can be used to limit GPU and memory power states to a particular subset
of states, **FORCE_POWER_CAP** that can be used to set desired power cap, and
**FORCE_PERF_LEVEL** that can be used to force desired
`power_dpm_force_performance_level` of a card (which can be `auto`, `low`,
`high`, `manual`, etc).

Here is an example how custom power state file may look like for Polaris cards:

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

Here is an example how custom power state file may look like for Navi cards:

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

### Installing and manually running the script

Simply place the [script](amdgpu-clocks) in `/usr/local/bin/amdgpu-clocks`:

    sudo ln -s $(pwd)/amdgpu-clocks /usr/local/bin/amdgpu-clocks

and specify custom power states in `/etc/default/amdgpu-custom-states.card0`:

    $ sudo amdgpu-clocks
    Detecting the state values at /sys/class/drm/card0/device/pp_od_clk_voltage:
      SCLK state 0: 300MHz, 750mV
      SCLK state 1: 500MHz, 775mV
      SCLK state 2: 600MHz, 785mV
      SCLK state 3: 700MHz, 800mV
      SCLK state 4: 800MHz, 820mV
      SCLK state 5: 900MHz, 840mV
      SCLK state 6: 950MHz, 850mV
      SCLK state 7: 1000MHz, 860mV
      MCLK state 0: 300MHz, 750mV
      MCLK state 1: 1000MHz, 800mV
      MCLK state 2: 1750MHz, 900mV
      Maximum clocks & voltages:
        SCLK clock 2000MHz
        MCLK clock 2250MHz
        VDDC voltage 1150mV
      Curent power cap: 87W
    Verifying user state values at amdgpu-custom-states.card0:
      SCLK state 6: 1000MHz, 860mV
      SCLK state 7: 1050MHz, 890mV
      MCLK state 2: 1600MHz, 900mV
      Force SCLK state to 5 6 7
      Force MCLK state to 2
      Force power cap to 90W
      Force performance level to manual
    Committing custom states to /sys/class/drm/card0/device/pp_od_clk_voltage:
      Done

The script can also be invoked with specific custom state file prefix (can be
a directory, case trailing slash is supplied), for example:

    $ sudo USER_STATES_PATH=custom-states amdgpu-clocks

This will load and apply custom states from all `custom-states.card*` files
in the current directory. Script can also be used with an additional 'restore'
parameter that can be used to restore all states to the initial defaults
(states before script was executed for the first time).

### Making the custom power states persist

This can be achieved by placing provided *systemd* service
[file](amdgpu-clocks.service) in `/lib/systemd/system/` directory,
and enable it:

    $ sudo systemctl enable --now amdgpu-clocks

However, if your system goes to suspend state, the above service will not
auto-restart (due to RemainAfterExit parameter). To fix that copy provided
[file](amdgpu-clocks-resume into `/usr/lib/systemd/system-sleep`

    $ cp amdgpu-clocks-resume /usr/lib/systemd/system-sleep/

Of course, one should not forget to place the actual custom power states in
`/etc/default/amdgpu-custom-state.cardX`.
