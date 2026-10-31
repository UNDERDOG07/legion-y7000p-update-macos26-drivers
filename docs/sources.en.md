# Component sources

This repository **mirrors no kexts**. What sits in `EFI/OC/Kexts` is what this machine actually runs,
and the table below says where each piece came from and how far it has drifted from upstream.

## One thing to know first

Every kext here came out of the 1401 package. Almost all of them are **one revision above** the upstream
release — `1.7.2 → 1.7.3` and so on — with a few further ahead. The likely explanation is that the
package rebuilds from upstream master and bumps the version, but the author does not say so per item.
What follows is the measured difference.

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
| VoodooI2C / VoodooI2CHID | 2.9.1 | [VoodooI2C/VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) v2.9.1 | same |
| USBToolBox / UTBDefault | 1.2.0 | [USBToolBox/kext](https://github.com/USBToolBox/kext) 1.2.0 | same |
| RealtekRTL8111 | 3.0.4 | [Mieze/RTL8111_driver_for_OS_X](https://github.com/Mieze/RTL8111_driver_for_OS_X) v3.0.0 | three revisions ahead |
| YogaSMC | 1.5.3 | [zhen-zen/YogaSMC](https://github.com/zhen-zen/YogaSMC) 1.5.3 | same |
| SimpleHDA | 1.0.0 | [laobamac/SimpleHDA](https://github.com/laobamac/SimpleHDA) v1.0.0 | same |
| AirportItlwm | 2.4.0 | **[laobamac/itlwm](https://github.com/laobamac/itlwm)** `v2.4.0-alpha` | **fork**, see below |
| IntelBluetoothFirmware / IntelBTPatcher | 2.5.0 | **[lshbluesky/IntelBluetoothFirmware](https://github.com/lshbluesky/IntelBluetoothFirmware)** v2.5.0 | **fork**, see below |
| AMFIPass | 1.4.1 | original `dhinakg/AMFIPass` is 404 | no reachable upstream |
| XHCI-unsupported | 0.9.2 | RehabMan's repository is gone | no reachable upstream |

## Wi-Fi has to come from a fork

Upstream [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) stops at **v2.3.0
(2024-06)** and does not support macOS 26.

The 2.4.0 on this machine comes from **`v2.4.0-alpha`** in
[laobamac/itlwm](https://github.com/laobamac/itlwm). It is the same lineage — the bundle identifier is
still `com.zxystd.AirportItlwm`, and `zxystd` is the OpenIntelWireless maintainer.

**Downloading 2.3.0 from upstream will not work here.** The Wi-Fi runs on that alpha branch.

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
