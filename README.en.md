# Lenovo Legion Y7000P IRX10 (83NN) — macOS 26 Tahoe driver notes

[中文](README.md) · **English**

**Machine**: Lenovo Legion Y7000P IRX10 (83NN), `i9-14900HX` + `NVIDIA RTX 5060 Laptop`, 32 GB
**System**: macOS 26.7.2 (25G332), Darwin 25.6.0, SMBIOS `MacBookPro16,4`

What this repo covers: **how each piece of hardware was made to work once the system was installed.**
What it does not cover: how to install macOS.

---

> **Where this machine came from**
>
> It was **upgraded from macOS 15 Sequoia to 26**, not installed fresh.
>
> The `1401` tool can install 26 directly now, so a clean install may behave differently — no leftover
> drivers in `/Library/Extensions`, no pre-existing TCC records, a clean NVRAM. Sections about
> reinstalling a driver after an update, an empty privacy list, or a feature disappearing after an OS
> update all assume that history.
>
> If you are stuck on a driver, the diagnostics and the final configuration here still apply. Check
> how your machine was installed first.

---

## Status

| Hardware | State | Notes | Doc |
|---|---|---|---|
| RTX 5060 | ✅ | Metal 3; internal panel does 240 Hz alone | [docs/gpu.md](docs/gpu.md) |
| Audio | ✅ | Speakers and mic; SimpleHDA + `layout-id 99` | [docs/audio.md](docs/audio.md) |
| Trackpad | ✅ | VoodooI2CHID | [docs/input.md](docs/input.md) |
| Keyboard | ✅ | modifier mapping needed | [docs/input.md](docs/input.md) |
| Wi-Fi | ✅ | 802.11ac / 80 MHz / 867 Mbps | [docs/wireless.md](docs/wireless.md) |
| Bluetooth | ⚠️ | connects; some BLE devices are flaky | [docs/wireless.md](docs/wireless.md) |
| Camera | ✅ | UVC, enumerates and captures | [docs/permissions.md](docs/permissions.md) |
| Battery | ✅ | needs SMCBatteryManager + YogaSMC | [docs/power.md](docs/power.md) |
| CPU topology | ✅ | 8 physical / 32 logical via `ctrsmt=full` | [docs/power.md](docs/power.md) |
| CPU name | ✅ | needs `revpatch=...,cpuname` | [docs/power.md](docs/power.md) |
| Brightness | ⚠️ | works alone, dead while mirroring | [docs/power.md](docs/power.md) |
| Privacy prompts | ⚠️ | never appear; grants must be written by hand | [docs/permissions.md](docs/permissions.md) |
| Delta updates | ✅ | needs `revpatch=sbvmm` | [docs/updates.md](docs/updates.md) |
| Sleep | ❌ | GPU does not come back | [docs/limits.md](docs/limits.md) |
| AirDrop | ❌ | AirportItlwm limitation | [docs/limits.md](docs/limits.md) |
| Secure Boot | ❌ | both the self-signed and the shim route are closed by firmware | [docs/limits.md](docs/limits.md) |

## Documentation

| Doc | What is in it |
|---|---|
| [docs/gpu.md](docs/gpu.md) | NullMoth install layout, the self-repair scripts, black-screen recovery |
| [docs/audio.md](docs/audio.md) | SimpleHDA and `layout-id`, and why AppleALC alone is not enough on 26 |
| [docs/input.md](docs/input.md) | modifier-key mapping, trackpad |
| [docs/wireless.md](docs/wireless.md) | Wi-Fi ax negotiation, the Bluetooth trio and `-ibtcompatbeta` |
| [docs/power.md](docs/power.md) | battery, CPU topology, CPU name, brightness |
| [docs/permissions.md](docs/permissions.md) | what to do when TCC never shows a prompt |
| [docs/updates.md](docs/updates.md) | delta updates, and how `revpatch` shadows `cpuname` |
| [docs/smbios.md](docs/smbios.md) | model and serial; the last four characters decide what About This Mac says |
| [docs/limits.md](docs/limits.md) | what this machine cannot do, and why |
| [docs/troubleshooting.md](docs/troubleshooting.md) | diagnosis paths |
| [boot-args.md](boot-args.md) | every boot argument: what it does, why it is there, what happens without it |

## Boot arguments, in one block

```
debug=0x100 keepsyms=1 -vi2c-force-polling ctrsmt=full
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
itlwm_he=1 revpatch=sbvmm,cpuname -ibtcompatbeta
```

[Boot-args.md](boot-args.md) goes through them one at a time, and separates the ones that were
verified as necessary from the ones that are simply carried over from the driver package.

## Companion tools

- **1401** — builds the boot stick and installs the NVIDIA driver. The actual source of both the EFI
  and the driver.
- **tccgrant** — the finished form of the script in [docs/permissions.md](docs/permissions.md), for
  granting permissions that the system refuses to ask about.
- **br** — sets internal-panel brightness through the private `DisplayServices` framework.

## The older repo

The complete macOS 15 Sequoia build notes — EFI, install flow, first-boot diagnosis:

[UNDERDOG07/hackintosh-lenovo-legion-y7000p-irx10](https://github.com/UNDERDOG07/hackintosh-lenovo-legion-y7000p-irx10)

The install steps there are out of date (`1401` installs 26 directly now), but the story of how this
machine was put together is still intact.

## Disclaimer

Everything here came off one real machine. Hardware revisions, BIOS versions and driver versions all
change the outcome. Back up the EFI before following anything, and know how to get back to a bootable
state.

This repo contains no valid serial number for any Apple service.
