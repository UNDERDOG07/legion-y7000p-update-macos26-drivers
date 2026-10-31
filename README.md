# 联想 Legion Y7000P IRX10（83NN）· macOS 26 Tahoe 驱动笔记

**中文** · [English](README.en.md)

**机器**：联想 Legion Y7000P IRX10（83NN），`i9-14900HX` + `NVIDIA RTX 5060 Laptop`，32 GB
**系统**：macOS 26.7.2（25G332），Darwin 25.6.0，SMBIOS `MacBookPro16,4`

这个仓库只写一件事：**系统装好之后，每个硬件是怎么修到能用的。**

安装流程不在这里。`1401` 现在能直接装 26，装机本身比后面调驱动短得多，也没什么值得记的步骤。

---

> **先说清楚这台机器的来历**
>
> 它是**从 macOS 15 Sequoia 升级到 26 的**，不是全新安装。
>
> 1401 现在可以直接装 26 了，所以全新安装会遇到的情况可能和这里不一样 —— 那台机器上没有旧驱动残留、没有旧的 TCC 授权记录、NVRAM 是干净的。本文里凡是讲「驱动失效后重装」「权限列表空白」「更新后某个功能不见」的部分，都和「有旧状态」这件事有关。
>
> 如果你在硬件驱动上卡住了，这里的排查过程和最终配置仍然可以参考，但请先对照一下自己是怎么装的。

---

## 状态表

| 硬件 | 状态 | 说明 | 文档 |
|---|---|---|---|
| RTX 5060 | ✅ | Metal 3；单屏时内屏可上 240 Hz | [docs/gpu.md](docs/gpu.md) |
| 音频 | ✅ | 扬声器与麦克风；SimpleHDA + `layout-id 99` | [docs/audio.md](docs/audio.md) |
| 触控板 | ✅ | VoodooI2CHID | [docs/input.md](docs/input.md) |
| 键盘 | ✅ | 无需任何映射 | [docs/input.md](docs/input.md) |
| Wi-Fi | ✅ | 802.11ac / 80 MHz / 867 Mbps | [docs/wireless.md](docs/wireless.md) |
| 蓝牙 | ⚠️ | 手柄与耳机正常，其他类型设备连不上 | [docs/wireless.md](docs/wireless.md) |
| 摄像头 | ✅ | UVC，可枚举可抓帧 | [docs/permissions.md](docs/permissions.md) |
| 电池 | ✅ | 需要 SMCBatteryManager + YogaSMC | [docs/power.md](docs/power.md) |
| CPU 拓扑 | ✅ | 8 物理 / 32 逻辑，`ctrsmt=full` | [docs/power.md](docs/power.md) |
| CPU 名称 | ✅ | 需要 `revpatch=...,cpuname` | [docs/power.md](docs/power.md) |
| 屏幕亮度 | ✅ | 1401 内置控制；镜像时无效 | [docs/power.md](docs/power.md) |
| 隐私授权 | ⚠️ | 不弹窗，只能手动写库 | [docs/permissions.md](docs/permissions.md) |
| 增量更新 | ✅ | 需要 `revpatch=sbvmm` | [docs/updates.md](docs/updates.md) |
| 睡眠 | ❌ | 唤醒后显卡回不来 | [docs/limits.md](docs/limits.md) |
| AirDrop | ❌ | AirportItlwm 不支持 | [docs/limits.md](docs/limits.md) |
| Secure Boot | ❌ | 自签与 shim 两条路都被固件封死 | [docs/limits.md](docs/limits.md) |

## 文档索引

| 文档 | 讲什么 |
|---|---|
| [docs/gpu.md](docs/gpu.md) | NullMoth 装在哪些路径、自愈脚本、黑屏怎么救 |
| [docs/audio.md](docs/audio.md) | SimpleHDA 与 `layout-id`，以及 26 上为什么不能只靠 AppleALC |
| [docs/input.md](docs/input.md) | 修饰键映射、触控板 |
| [docs/wireless.md](docs/wireless.md) | Wi-Fi 的 ax 协商、蓝牙三件套与 `-ibtcompatbeta` |
| [docs/power.md](docs/power.md) | 电池、CPU 拓扑、CPU 名称、亮度 |
| [docs/permissions.md](docs/permissions.md) | 系统不弹授权窗时怎么办 |
| [docs/updates.md](docs/updates.md) | 增量更新，以及 `revpatch` 顶掉 `cpuname` 的连带效应 |
| [docs/smbios.md](docs/smbios.md) | 机型与序列号，末四位决定「关于本机」显示什么 |
| [docs/limits.md](docs/limits.md) | 这台机器做不到的事，以及为什么 |
| [docs/troubleshooting.md](docs/troubleshooting.md) | 排错路径 |
| [boot-args.md](boot-args.md) | 每个启动参数干什么、为什么加、去掉会怎样 |

## 关键事实

- **macOS 26 没有任何 Intel 核显驱动**，内屏只能走独显，所以 BIOS 必须是独显直连。这不是优化项，是前提。
- **显卡驱动不在 EFI 里**，装在 `/Library/Extensions`。换 EFI 修不了显卡，系统更新却可能把它弄掉。
- **系统更新会重建内核集合**，驱动和部分 kext 需要重新装。
- **隐私授权不会弹窗**。系统判定「策略不允许弹窗」之后直接拒绝，数据库里因此永远没有记录，设置面板也就是空的。
- **Secure Boot 必须保持关闭**。EFI 里的引导文件是未签名的，打开就起不来。

## 一句话版启动参数

```
debug=0x100 keepsyms=1 -vi2c-force-polling ctrsmt=full
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
itlwm_he=1 revpatch=sbvmm,cpuname -ibtcompatbeta
```

逐条解释在 [boot-args.md](boot-args.md)，那里会把「验证过是必需的」和「从驱动包里照抄过来的」分开标注。

## 配套工具

- **1401** —— 做引导盘、装 NVIDIA 驱动、提供内屏亮度控制。EFI 和驱动的实际来源。
- **tccgrant** —— [docs/permissions.md](docs/permissions.md) 里那段脚本的最终形态，用来给系统拒绝询问的 App 补授权。

## EFI

`EFI/` 是这台机器**当前在用**的配置，直接从 U 盘复制，只做了脱敏。

和旧仓库那份相比，主要差别在启动参数（`revpatch=sbvmm,cpuname`、`-ibtcompatbeta`、`amfi=0x80` 等）、kext 版本，以及 SMBIOS。

**三码是占位符，用之前必须自己生成：**

| 字段 | 当前值 |
|---|---|
| `SystemSerialNumber` | `C02XXXXXXXXX` |
| `MLB` | `C02XXXXXXXXXXXXXX` |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` |
| `ROM` | `112233445566` |

这样能开机，但「关于本机」会显示 Unknown，iServices 也用不了。用 GenSMBIOS 按 `MacBookPro16,4` 生成，**末四位必须是 `MD6T`** —— 原因见 [docs/smbios.md](docs/smbios.md)。

原有的 17 份历史配置备份（都带真实序列号）已从仓库中移除，只保留当前这一份。

一个如实保留的细节：这份 `config.plist` 里 `Misc → Boot → HideAuxiliary` 是 `false`，因为 `1401.app` 最后一次改设置时把它重置了。后果是 picker 里会多出两个 `1401:` 开头的条目。想隐藏回去，改法见 [docs/gpu.md](docs/gpu.md) —— 这正是那篇里提到的「App 会重写 config」实例。

## 上一版

macOS 15 Sequoia 时期的完整装机笔记（EFI、安装流程、首次启动排错）：

[UNDERDOG07/hackintosh-lenovo-legion-y7000p-irx10](https://github.com/UNDERDOG07/hackintosh-lenovo-legion-y7000p-irx10)

那里的安装步骤已经过时了（1401 能直接装 26），但「这台机器怎么装成的」那段历史仍然完整。

## 免责声明

所有内容来自一台真机。硬件批次、BIOS 版本、驱动版本都会影响结果。照着做之前先备份 EFI，并且确保自己知道怎么回到能开机的状态。

本仓库不含任何可用于 Apple 服务的有效序列号。
