# Component sources

This repository **mirrors no kexts**. What sits in `EFI/OC/Kexts` is what this machine actually runs,
and the table below says where each piece came from and how far it has drifted from upstream.

## One thing to know first

Every kext here came out of the 1401 package. Almost all of them are **one revision above** the upstream
release — `1.7.2 → 1.7.3` and so on — with a few further ahead. The likely explanation is that the
package rebuilds from upstream master and bumps the version, but the author does not say so per item.
What follows is the measured difference.

One exception is worth reading carefully. `VoodooI2CHID`, `VoodooGPIO` and `VoodooI2CServices` are
submodules of VoodooI2C, each in its own repository with its own version number. Their `1.0` and `1.1`
are the upstream numbers; they are not on the same scale as the `2.9.1` of the VoodooI2C package, so do
not read them as "an old build".

## Full comparison

| Component | Here | Upstream | Relation |
|---|---|---|---|
| Lilu | 1.7.3 | [acidanthera/Lilu](https://github.com/acidanthera/Lilu) 1.7.2 | one revision ahead |
| WhateverGreen | 1.7.2 | [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen) 1.7.1 | one ahead (currently **disabled**) |
| AppleALC | 1.9.9 | [acidanthera/AppleALC](https://github.com/acidanthera/AppleALC) 1.9.8 | one ahead |
| VirtualSMC | 1.3.9 | [acidanthera/VirtualSMC](https://github.com/acidanthera/VirtualSMC) 1.3.8 | one ahead |
| SMCBatteryManager / SMCLightSensor / SMCProcessor / SMCSuperIO | 1.3.9 | shipped with VirtualSMC | same |
| RestrictEvents | 1.1.7 | [acidanthera/RestrictEvents](https://github.com/acidanthera/RestrictEvents) 1.1.6 | one ahead |
| BrightnessKeys | 1.0.4 | [acidanthera/BrightnessKeys](https://github.com/acidanthera/BrightnessKeys) 1.0.3 | one ahead |
| NVMeFix | 1.1.4 | [acidanthera/NVMeFix](https://github.com/acidanthera/NVMeFix) 1.1.3 | one ahead |
| CpuTscSync | 1.1.2 | [acidanthera/CpuTscSync](https://github.com/acidanthera/CpuTscSync) 1.1.2 | same |
| VoodooPS2Controller | 2.3.8 | [acidanthera/VoodooPS2](https://github.com/acidanthera/VoodooPS2) 2.3.7 | one ahead |
| BlueToolFixup | 2.7.3 | [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) 2.7.2 | one ahead |
| CpuTopologyRebuild | 2.0.2 | [b00t0x/CpuTopologyRebuild](https://github.com/b00t0x/CpuTopologyRebuild) 2.0.2 | same |
| VoodooI2C | 2.9.1 | [VoodooI2C/VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) v2.9.1 | same |
| VoodooI2CHID | 1.0 | [VoodooI2C/VoodooI2CHID](https://github.com/VoodooI2C/VoodooI2CHID) 1.0 | same |
| VoodooGPIO | 1.1 | [VoodooI2C/VoodooGPIO](https://github.com/VoodooI2C/VoodooGPIO) 1.1 | same |
| VoodooI2CServices | 1.0 | shipped with VoodooI2C | numbered separately |
| VoodooInput (with VoodooPS2) | 1.1.7 | [acidanthera/VoodooInput](https://github.com/acidanthera/VoodooInput) 1.1.6 | one ahead |
| USBToolBox | 1.2.0 | [USBToolBox/kext](https://github.com/USBToolBox/kext) 1.2.0 | same |
| UTBDefault | 1.0 | shipped with USBToolBox | port-map injector, numbered separately |
| RealtekRTL8111 | 3.0.4 | [Mieze/RTL8111_driver_for_OS_X](https://github.com/Mieze/RTL8111_driver_for_OS_X) v3.0.0 | three revisions ahead |
| YogaSMC | 1.5.3 | [zhen-zen/YogaSMC](https://github.com/zhen-zen/YogaSMC) 1.5.3 | same |
| SimpleHDA | 1.0.0 | [laobamac/SimpleHDA](https://github.com/laobamac/SimpleHDA) v1.0.0 | same |
| AirportItlwm | 2.4.0 | **[laobamac/itlwm](https://github.com/laobamac/itlwm)** `v2.4.0-alpha` | **fork**, currently **disabled** (kept for rollback) |
| ForbiddenItlwm | 0.4.1-beta1 | **[retropineaple/ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm)** `v0.4.1-beta1` | **fork**, AirDrop; enabled |
| BTLEAdvFixup | 0.8.3 | same | AirDrop companion; enabled |
| IntelBluetoothFirmware / IntelBTPatcher | 2.5.0 | **[lshbluesky/IntelBluetoothFirmware](https://github.com/lshbluesky/IntelBluetoothFirmware)** v2.5.0 | **fork**, see below |
| AMFIPass | 1.4.1 | original `dhinakg/AMFIPass` is 404 | no reachable upstream |
| XHCI-unsupported | 0.9.2 | RehabMan's repository is gone | no reachable upstream |

## Components that are not in the EFI

`EFI/OC/Kexts` holds only what the boot path needs. The GPU driver that decides whether this machine
shows a picture is not in there — it lives on the system volume:

| Component | Version | Location | Source |
|---|---|---|---|
| NullMoth NVIDIA driver | 2.6.5 | `/Library/Extensions`, `/Library/GPUBundles` | [nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver) |
| GSP firmware | 615.78.08 | `/Library/NullMoth/nvfw/nvidia/` | shipped with the driver package |
| 1401 boot stick tool | 2.6.5 | Windows program | [nullmoth/1401](https://github.com/nullmoth/1401) |

None of the three is in this repository or in the EFI. Install layout and reinstall triggers are in
[gpu.en.md](gpu.en.md).

## Wi-Fi: two forks deep

Upstream [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) stops at **v2.3.0
(2024-06)** and does not support macOS 26.

**First layer**: **`v2.4.0-alpha`** in [laobamac/itlwm](https://github.com/laobamac/itlwm) is what makes
an Intel card work on Tahoe at all. The bundle identifier is still `com.zxystd.AirportItlwm`, and
`zxystd` is the OpenIntelWireless maintainer — same lineage.

**Second layer**: [retropineaple/ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm) adds
AWDL, the interface AirDrop runs on, on top of laobamac's **2.4.1**. This machine runs the Tahoe build
of `v0.4.1-beta1`, which is why the firmware string reads `itlwm: 2.4.1_forbidden`.

The second layer is the one currently active. `AirportItlwm.kext` stays in `Kexts`, disabled, for
rollback.

**Downloading 2.3.0 from upstream will not work here** — there are two forks in between.

## Bluetooth, same story

Upstream [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware)
is at **v2.4.0 (2024-02)**.

The 2.5.0 here comes from
[lshbluesky/IntelBluetoothFirmware](https://github.com/lshbluesky/IntelBluetoothFirmware), a fork that
has since moved to v2.5.2. `IntelBTPatcher` is the second kext in the same repository and follows its
version.

`BlueToolFixup` is separate: it belongs to acidanthera's BrcmPatchRAM project, is a Lilu plugin, and
exists to make a third-party controller present itself as an Apple one.

## Two with no reachable upstream

**AMFIPass 1.4.1** (`com.dhinakg.AMFIPass`). The author is dhinakg, but `github.com/dhinakg/AMFIPass`
returns 404 and the repository is no longer in that account. Search turns up third-party re-uploads
only. The copy here came from the 1401 package.

**XHCI-unsupported 0.9.2** (`org.rehabman.injector.XHCI.unsupported`). RehabMan's original repository
was deleted from GitHub years ago. Its job is to supply port properties for controllers whose firmware
does not describe them; this machine already maps ports through `USBToolBox` + `UTBDefault`, so its
actual contribution is doubtful. Removing it would need a reboot to verify, which has not been done.

## About the version gap

`RealtekRTL8111` is three revisions ahead, the rest are one — a pattern that looks like "rebuilt from
master across the board". If you want to update: **do not drop an upstream release over these files.**
They are package builds, and the paths and signing flow may be tied to the 1401 installer. Wait for a
new 1401 package instead.
