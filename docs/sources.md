# 组件来源

这个仓库**不镜像任何 kext**。`EFI/OC/Kexts` 里是这台机器实际在跑的文件，下表说明每一样从哪来、和上游差多少。

## 整体情况

本机的 kext 全部来自 1401 的打包。其中绝大多数比上游 release **高一个修订号**（`1.7.2 → 1.7.3` 这种），少数几个更高。推测是打包时基于上游 master 重新构建并打了版本号，作者没有逐项说明，这里只记录差异。

有一类例外要看清楚：`VoodooI2CHID`、`VoodooGPIO`、`VoodooI2CServices` 是 VoodooI2C 的子模块，各自有独立仓库和自己的版本号。它们的 `1.0`、`1.1` 是上游本来的编号，和 VoodooI2C 主包的 `2.9.1` 不是一套，别当成「版本很旧」。

## 完整对照

| 组件 | 本机 | 上游 | 关系 |
|---|---|---|---|
| Lilu | 1.7.3 | [acidanthera/Lilu](https://github.com/acidanthera/Lilu) 1.7.2 | 高 1 个修订 |
| WhateverGreen | 1.7.2 | [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen) 1.7.1 | 高 1（当前**禁用**）|
| AppleALC | 1.9.9 | [acidanthera/AppleALC](https://github.com/acidanthera/AppleALC) 1.9.8 | 高 1 |
| VirtualSMC | 1.3.9 | [acidanthera/VirtualSMC](https://github.com/acidanthera/VirtualSMC) 1.3.8 | 高 1 |
| SMCBatteryManager / SMCLightSensor / SMCProcessor / SMCSuperIO | 1.3.9 | 随 VirtualSMC 发布 | 同上 |
| RestrictEvents | 1.1.7 | [acidanthera/RestrictEvents](https://github.com/acidanthera/RestrictEvents) 1.1.6 | 高 1 |
| BrightnessKeys | 1.0.4 | [acidanthera/BrightnessKeys](https://github.com/acidanthera/BrightnessKeys) 1.0.3 | 高 1 |
| NVMeFix | 1.1.4 | [acidanthera/NVMeFix](https://github.com/acidanthera/NVMeFix) 1.1.3 | 高 1 |
| CpuTscSync | 1.1.2 | [acidanthera/CpuTscSync](https://github.com/acidanthera/CpuTscSync) 1.1.2 | 一致 |
| VoodooPS2Controller | 2.3.8 | [acidanthera/VoodooPS2](https://github.com/acidanthera/VoodooPS2) 2.3.7 | 高 1 |
| BlueToolFixup | 2.7.3 | [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) 2.7.2 | 高 1 |
| CpuTopologyRebuild | 2.0.2 | [b00t0x/CpuTopologyRebuild](https://github.com/b00t0x/CpuTopologyRebuild) 2.0.2 | 一致 |
| VoodooI2C | 2.9.1 | [VoodooI2C/VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) v2.9.1 | 一致 |
| VoodooI2CHID | 1.0 | [VoodooI2C/VoodooI2CHID](https://github.com/VoodooI2C/VoodooI2CHID) 1.0 | 一致 |
| VoodooGPIO | 1.1 | [VoodooI2C/VoodooGPIO](https://github.com/VoodooI2C/VoodooGPIO) 1.1 | 一致 |
| VoodooI2CServices | 1.0 | 随 VoodooI2C 发布 | 独立编号 |
| VoodooInput（随 VoodooPS2） | 1.1.7 | [acidanthera/VoodooInput](https://github.com/acidanthera/VoodooInput) 1.1.6 | 高 1 |
| USBToolBox | 1.2.0 | [USBToolBox/kext](https://github.com/USBToolBox/kext) 1.2.0 | 一致 |
| UTBDefault | 1.0 | 随 USBToolBox 发布 | 端口映射 injector，版本独立 |
| RealtekRTL8111 | 3.0.4 | [Mieze/RTL8111_driver_for_OS_X](https://github.com/Mieze/RTL8111_driver_for_OS_X) v3.0.0 | 高 3 个修订 |
| YogaSMC | 1.5.3 | [zhen-zen/YogaSMC](https://github.com/zhen-zen/YogaSMC) 1.5.3 | 一致 |
| SimpleHDA | 1.0.0 | [laobamac/SimpleHDA](https://github.com/laobamac/SimpleHDA) v1.0.0 | 一致 |
| AirportItlwm | 2.4.0 | **[laobamac/itlwm](https://github.com/laobamac/itlwm)** `v2.4.0-alpha` | **fork**，见下 |
| IntelBluetoothFirmware / IntelBTPatcher | 2.5.0 | **[lshbluesky/IntelBluetoothFirmware](https://github.com/lshbluesky/IntelBluetoothFirmware)** v2.5.0 | **fork**，见下 |
| AMFIPass | 1.4.1 | 原仓库 `dhinakg/AMFIPass` 已 404 | 无可用上游 |
| XHCI-unsupported | 0.9.2 | RehabMan 原仓库已删除 | 无可用上游 |

## 不在 EFI 里的组件

`EFI/OC/Kexts` 只放引导阶段要用的东西。决定这台机器能不能出画面的显卡驱动不在里面，它装在系统卷上：

| 组件 | 版本 | 位置 | 来源 |
|---|---|---|---|
| NullMoth NVIDIA 驱动 | 2.6.5 | `/Library/Extensions`、`/Library/GPUBundles` | [nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver) |
| GSP 固件 | 615.78.08 | `/Library/NullMoth/nvfw/nvidia/` | 随驱动包提供 |
| 1401 引导盘工具 | 2.6.5 | Windows 程序 | [nullmoth/1401](https://github.com/nullmoth/1401) |

三者都不在本仓库，也不在 EFI 里。装载位置和重装时机见 [gpu.md](gpu.md)。

## Wi-Fi：必须走 fork

上游 [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) 的最新发布是 **v2.3.0（2024-06）**，不支持 macOS 26。

本机的 2.4.0 来自 [laobamac/itlwm](https://github.com/laobamac/itlwm) 的 **`v2.4.0-alpha`**。这是同一作者谱系的延续——bundle identifier 仍然是 `com.zxystd.AirportItlwm`，作者 `zxystd` 就是 OpenIntelWireless 的维护者。

**直接去上游下 2.3.0 装不上去**，这台机器能连 Wi-Fi 靠的是这个 alpha 分支。

## 蓝牙：同样必须走 fork

上游 [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) 最新是 **v2.4.0（2024-02）**。

本机的 2.5.0 来自 [lshbluesky/IntelBluetoothFirmware](https://github.com/lshbluesky/IntelBluetoothFirmware)，该 fork 目前已经到 v2.5.2。`IntelBTPatcher` 是同一个仓库里的第二个 kext，版本随它。

`BlueToolFixup` 是另一回事：它属于 acidanthera 的 BrcmPatchRAM 项目，是 Lilu 插件，负责把第三方控制器伪装成 Apple 的。

## 两个查不到上游的

**AMFIPass 1.4.1**（identifier `com.dhinakg.AMFIPass`）。作者是 dhinakg，但 `github.com/dhinakg/AMFIPass` 现在返回 404，账号下也没有这个仓库了。搜索结果里只有几个第三方转载。本机这一份来自 1401 包。

**XHCI-unsupported 0.9.2**（identifier `org.rehabman.injector.XHCI.unsupported`）。RehabMan 的原始仓库已从 GitHub 删除多年。这个 kext 的作用是给缺少端口定义的控制器补上属性；本机已经用 `USBToolBox` + `UTBDefault` 做端口映射，它的实际作用存疑，但测试移除需要重启验证，暂未做。

## 关于版本差异

`RealtekRTL8111` 差三个修订、其余差一个，这个模式很像「统一从 master 重新构建」。如果你要自己更新，**不要直接拿上游 release 覆盖**：这些是打包版，路径和签名流程可能与 1401 的安装脚本配合，覆盖后可能装不上。要更新就等 1401 出新包。
