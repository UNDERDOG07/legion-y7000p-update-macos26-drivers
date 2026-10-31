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
| RTX 5060 | ✅ | Metal 3; internal panel does 240 Hz alone | [docs/gpu.en.md](docs/gpu.en.md) |
| Audio | ✅ | SimpleHDA restores AppleHDA so AppleALC + `layout-id 99` can take over | [docs/audio.en.md](docs/audio.en.md) |
| Trackpad | ✅ | VoodooI2CHID, with VoodooInput taken from VoodooPS2 | [docs/input.en.md](docs/input.en.md) |
| Keyboard | ✅ | no remapping needed | [docs/input.en.md](docs/input.en.md) |
| Wi-Fi | ✅ | AX1690i (user-replaced) + `AirportItlwm` | [docs/wireless.en.md](docs/wireless.en.md) |
| Bluetooth | ⚠️ | controllers and headphones work; other device types do not | [docs/wireless.en.md](docs/wireless.en.md) |
| Camera | ✅ | UVC, enumerates and captures | [docs/permissions.en.md](docs/permissions.en.md) |
| Battery | ✅ | needs SMCBatteryManager + YogaSMC | [docs/power.en.md](docs/power.en.md) |
| CPU topology | ✅ | 8 physical / 32 logical via `ctrsmt=full` | [docs/power.en.md](docs/power.en.md) |
| CPU name | ✅ | needs `revpatch=...,cpuname` | [docs/power.en.md](docs/power.en.md) |
| Brightness | ✅ | built into 1401.app; no effect while mirroring | [docs/power.en.md](docs/power.en.md) |
| Privacy prompts | ⚠️ | never appear; grants must be written by hand | [docs/permissions.en.md](docs/permissions.en.md) |
| Delta updates | ✅ | needs `revpatch=sbvmm` | [docs/updates.en.md](docs/updates.en.md) |
| Sleep | ❌ | GPU does not come back | [docs/limits.en.md](docs/limits.en.md) |
| AirDrop | ❌ | AirportItlwm limitation | [docs/limits.en.md](docs/limits.en.md) |
| Secure Boot | ❌ | both the self-signed and the shim route are closed by firmware | [docs/limits.en.md](docs/limits.en.md) |

## Documentation

| Doc | What is in it |
|---|---|
| [docs/gpu.en.md](docs/gpu.en.md) | NullMoth install layout, the self-repair scripts, black-screen recovery |
| [docs/audio.en.md](docs/audio.en.md) | SimpleHDA and `layout-id`, and why AppleALC alone is not enough on 26 |
| [docs/input.en.md](docs/input.en.md) | modifier-key mapping, trackpad |
| [docs/wireless.en.md](docs/wireless.en.md) | Wi-Fi ax negotiation, the Bluetooth trio and `-ibtcompatbeta` |
| [docs/power.en.md](docs/power.en.md) | battery, CPU topology, CPU name, brightness |
| [docs/permissions.en.md](docs/permissions.en.md) | what to do when TCC never shows a prompt |
| [docs/updates.en.md](docs/updates.en.md) | delta updates, and how `revpatch` shadows `cpuname` |
| [docs/smbios.en.md](docs/smbios.en.md) | model and serial; the last four characters decide what About This Mac says |
| [docs/limits.en.md](docs/limits.en.md) | what this machine cannot do, and why |
| [docs/troubleshooting.en.md](docs/troubleshooting.en.md) | diagnosis paths |
| [docs/sources.en.md](docs/sources.en.md) | version and upstream for every kext, and which two must come from a fork |
| [boot-args.md](boot-args.md) | every boot argument: what it does, why it is there, what happens without it |

Every document has both languages: `xxx.md` is Chinese, `xxx.en.md` is English.

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

- **1401** — builds the boot stick, installs the NVIDIA driver, and provides the internal-panel
  brightness control. The actual source of both the EFI and the driver.
- **tccgrant** — the finished form of the script in [docs/permissions.en.md](docs/permissions.en.md), for
  granting permissions that the system refuses to ask about.

## EFI

`EFI/` is the configuration this machine is **currently running**, copied straight off the stick and
sanitised.

Compared with the older repo's copy, the differences are in `boot-args` (`revpatch=sbvmm,cpuname`,
`-ibtcompatbeta`, `amfi=0x80`), the kext versions, and the SMBIOS block.

**The identity fields are placeholders. Generate your own before using it:**

| Field | Current value |
|---|---|
| `SystemSerialNumber` | `C02XXXXXXXXX` |
| `MLB` | `C02XXXXXXXXXXXXXX` |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` |
| `ROM` | `112233445566` |

It will boot like this, but About This Mac shows Unknown and iServices will not work. Generate a pair
for `MacBookPro16,4` with GenSMBIOS — **the last four characters must be `MD6T`**, see
[docs/smbios.en.md](docs/smbios.en.md) for why.

The seventeen historical config backups that came with the stick (all carrying the real serial) are not
included. Only the current configuration is here.

One detail kept as-is: `Misc → Boot → HideAuxiliary` is `false` in this file, because `1401.app` reset
it during its last settings change. The effect is two extra `1401:` entries in the picker. How to hide
them again is in [docs/gpu.en.md](docs/gpu.en.md) — this is exactly the rewrite behaviour described there.

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
