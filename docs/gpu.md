# The GPU (RTX 5060 Laptop)

There is no Intel iGPU driver in macOS 26, so the internal panel is only reachable through the dGPU.
That makes the NVIDIA driver the single point of failure for the whole build: if it does not load,
there is no picture at all.

The driver is **NullMoth 2.6.5**, installed into `/Library/Extensions` — not into the EFI. That one
distinction explains most of what follows.

## What is on disk

| Piece | Path | Version |
|---|---|---|
| Kernel extensions | `/Library/Extensions/{NVRM,NVAccel,NVRMFB,NVRMAGDC}.kext` | 2.6.5 |
| Metal driver | `/Library/GPUBundles/NVMTLDriver.bundle` | 2.6.5 |
| Shared bundle | `/Library/GPUBundles/NVIDIAShared.bundle` | — |
| Metal allow list | `/Library/GPUBundles/nvmtl-allow.txt` | — |
| GSP firmware | `/Library/NullMoth/nvfw/nvidia/615.78.08/` | 615.78.08 |
| Driver state | `/Library/NullMoth/driver-version`, `os-major` | `2.6.5`, `26` |

The four kexts share the bundle identifier prefix `com.nullmoth.*`.

Hardware side, for cross-reference: `10DE:2D59`, subsystem `17AA:802F`, 8 GB VRAM (BAR1 8192 MB),
PCIe x8, Metal 3.

## Why none of it is in the EFI

The EFI only carries what has to exist before the kernel finishes booting: OpenCore itself, and the
kexts the boot path needs (trackpad, Wi-Fi, Bluetooth). The GPU driver loads later, as a system
extension.

Three consequences:

- Swapping the EFI, or editing `config.plist`, cannot fix a broken GPU driver.
- A macOS update rebuilds the kernel collection, and the driver can fall out of it.
- Reinstalling macOS means reinstalling the driver. It does not come from the stick.

> This machine got here by **upgrading** from macOS 15 to 26, and the driver survived that upgrade.
> That was luck, not a rule. A clean install starts with an empty `/Library/Extensions`.

## Install, and when it has to be done again

`1401.app` is the normal path. It writes the four kexts, rebuilds the boot kernel collection, and asks
for a restart.

Before installing, check for leftovers:

```bash
ls /Library/Extensions | grep -i nv
```

After installing, check that all four came up:

```bash
kmutil showloaded | grep -i nullmoth
```

A health check that needs none of the above: `system_profiler SPDisplaysDataType` should still print
`Metal Support: Metal 3`.

The driver has to go back on when:

- macOS has been updated (the kernel collection is rebuilt)
- the desktop drops to a low resolution, or Metal disappears
- `kmutil showloaded | grep -i nullmoth` returns fewer than four lines

## The driver repairs itself

This is new in 2.6.5 and is not in the older notes. `/Library/NullMoth/` carries three scripts that
run without being asked:

| Script | Trigger | What it does |
|---|---|---|
| `bootguard.sh` | LaunchDaemon at boot | Waits for `WindowServer`, confirms it is still alive after 45 s, then sets `nullmoth-nv-ok=1` in NVRAM |
| `lastgood.sh` | LaunchDaemon at boot | If the new driver failed to bring the display up **twice**, restores the previous driver from `/Library/NullMoth/backup-<timestamp>/` |
| `nullmoth-recover.sh` | `nullmoth-remove` flag in NVRAM | Runs `nullmoth-setup.sh --remove`, then reboots |

Two things follow.

**The "Remove NVIDIA driver" picker entry does not remove anything at boot.** It sets an NVRAM
variable; the removal happens after macOS starts, and the machine reboots itself at the end. That is
why it still works when the display is dead.

**A failed driver install usually rolls itself back.** `lastgood.log` is the record, one line per
event:

```
Tue Oct 27 17:26:16 CST 2026 the new driver brought its display up; keeping it
```

`bootguard.log` records the opposite case — the starts that counted as good, written 45 seconds after
`WindowServer` came up.

## Black screen: the two picker entries

They live in `Misc → Tools`, both pointing at `NullMothSafe.efi`, separated only by arguments:

| Entry | Arguments | Effect |
|---|---|---|
| `1401: Remove NVIDIA driver` | empty | sets the NVRAM flag → uninstall, then reboot |
| `1401: Start once without NVIDIA driver` | `once` | skips loading the driver for this start only |

Both are hidden in normal use (`Auxiliary = true`). **Press space in the picker** to reveal them.

Reach for the second one first: it gets you a desktop without touching the installed driver, and from
there you can reinstall from `1401.app`.

## The config.plist rewrite

Every time `1401.app` changes a setting — turning on internal-panel brightness, for instance — it
rewrites the whole `config.plist`. Two things get reset:

- `Misc → Boot → HideAuxiliary` goes back to `false`
- both `NullMothSafe.efi` tool entries go back to `Auxiliary = false`

Not a bug: the app does not know about flags that were set by hand. Re-apply after each change:

```
Misc → Boot → HideAuxiliary = true
Misc → Tools[Path = NullMothSafe.efi] → Auxiliary = true    (both entries)
```

Hiding does not remove anything. Pressing space still brings both back.

## Refresh rate and mirroring

On its own, the internal panel offers 2560×1600 @ 240 Hz.

With an external display mirrored, the panel follows the master and drops to 2560×1440 @ 60, while
the external runs 180 Hz as the Master Mirror.

Brightness is dead in that state — `DisplayServices` reads back 0 and writes are ignored. Unplug the
external display and it comes back. That is the compositor, not the GPU driver; see
[power.md](power.md).

## Video decode

No NVDEC on this GPU under macOS, and no iGPU driver, so there is no hardware decode block in the
path.

VideoToolbox still has a Metal-backed route. Measured on 1080p HEVC, ten-second clip,
`ffmpeg -benchmark`, three runs each:

| Path | utime |
|---|---|
| `-hwaccel videotoolbox` | 0.61 – 0.65 s |
| CPU only (ffmpeg's own decoder) | 1.67 – 2.61 s |

VideoToolbox is 2.5–4× faster. `VTDecoderXPCService` loads `NVMTLDriver` while it runs, so this is
Metal-assisted software decoding, not a decode engine.

H.264 goes the other way: videotoolbox 0.82 s against 0.59 s on the CPU.

These numbers describe command-line decoding, not what a player will do. They are here to make one
point: **no hardware decoder does not mean slow decoding.**

## Known limits

- **DRM** — Apple TV and Netflix will not play. Not tested, and not worth testing: the driver has no
  HDCP path.
- **Sleep** — the GPU does not come back after wake. Whole-machine limit, see [limits.md](limits.md).
- **External displays** hang off the dGPU's HDMI, independent of the BIOS MUX. That is also why the
  external monitor is the only usable output when the internal panel is dark.
