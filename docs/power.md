# Power, CPU and brightness

## Battery

| Piece | Version |
|---|---|
| `VirtualSMC.kext` | 1.3.9 |
| `SMCBatteryManager.kext` | 1.3.9 |
| `SMCProcessor.kext` | 1.3.9 |
| `SMCSuperIO.kext` | 1.3.9 |
| `YogaSMC.kext` | 1.5.3 |
| `SMCLightSensor.kext` | 1.3.9 |

Battery reads correctly: design capacity 5545 mAh, current maximum 5154 mAh (93%), cycle count 76.

`YogaSMC` handles the Lenovo-specific embedded controller — charge thresholds, fan tables, the extra
sensors. It needs `SSDT-EC.aml` to expose the EC properly; without it the kext loads and does nothing.

`SMCSuperIO` supplies fan and temperature data but has no interface of its own. `HWMonitorSMC2` is what
makes it visible.

## CPU topology

The i9-14900HX has 8 P-cores, 16 E-cores, 32 threads. macOS has no concept of an E-core, so left alone
it treats all 24 cores as equivalent and schedules single-threaded work onto efficiency cores.

`CpuTopologyRebuild.kext` with `ctrsmt=full` fixes the presentation: E-cores become SMT siblings of
P-cores.

```bash
sysctl -n hw.physicalcpu hw.logicalcpu    # 8  32
sysctl -n hw.ncpu                          # 32
```

The `8` is not missing cores. It is the intended result, and it is why the machine behaves correctly
under load.

## The CPU name, and how it was lost

This one is worth recording in full because the cause was not where it looked.

`RestrictEvents` takes the CPU name from `revcpuname`, but only when `cpuname` appears in `revpatch`.
The config carried both, in two different places:

```
NVRAM[RestrictEvents GUID]   revpatch   = sbvmm,cpuname
                             revcpuname = Intel(R) Core(TM) i9-14900HX
boot-args                    revpatch   = sbvmm
```

`boot-args` wins over NVRAM. So the `cpuname` half was never active; the name fell back to whatever
SMBIOS reported; SMBIOS reported nothing useful; About This Mac said `Processor Name: Unknown`.

The fix is one word:

```
revpatch=sbvmm,cpuname
```

Two lessons worth carrying.

First, `revpatch` is a **single token**. Adding `sbvmm` to `boot-args` to fix delta updates silently
replaced the NVRAM value that carried `cpuname`. Neither change looked related to the other.

Second, the symptom appeared in the same week as an OS update, which made it look like an Apple
regression. It was not — the SMBIOS table had the correct string all along.

## Brightness

There is no `AppleBacklightDisplay` on this machine, so the normal brightness path does not exist. What
works is the private `DisplayServices` framework:

```python
DisplayServicesSetBrightness(CGMainDisplayID(), 0.5)
```

Two caveats.

**Mirroring disables it.** With an external display mirrored, the internal panel is a hardware mirror:
`DisplayServicesGetBrightness` reads back 0 and writes are ignored. Unplug the external display and it
comes back. That is the compositor, not the GPU driver.

**Every boot starts at 100%.** The driver sets full brightness at init and macOS has no stored value to
restore. A LaunchAgent applies 50% six and sixteen seconds after login:

```
~/Library/LaunchAgents/com.user.brightness.plist
```

Two applications, because the driver occasionally initialises after the first one has already run.

The `br` helper does the same from a terminal:

```bash
br        # read current
br 50     # set 50%
br up     # +10%
br down   # -10%
```

It picks the built-in display by `CGDisplayIsBuiltin` rather than by "main display". That matters once
an external monitor is attached, because the main display in that case has no backlight to set.

## Sleep

Disabled deliberately, and not because of a `pmset` setting. See [limits.md](limits.md) — the GPU does
not survive wake, so sleeping means losing the session.
