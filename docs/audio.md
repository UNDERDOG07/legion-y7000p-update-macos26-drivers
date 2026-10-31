# 音频

扬声器和内置麦克风都能用。两个 kext 干活，一个设备属性把它们绑到 codec 上。

## 在跑的东西

| 组件 | 版本 | 位置 |
|---|---|---|
| [`AppleALC.kext`](https://github.com/acidanthera/AppleALC) | 1.9.9 | `EFI/OC/Kexts` |
| [`SimpleHDA.kext`](https://github.com/laobamac/SimpleHDA) | 1.0.0 | `EFI/OC/Kexts` |
| `layout-id` | `99` | `DeviceProperties → Add → PciRoot(0x0)/Pci(0x1f,0x3)` |

`SimpleHDA` 的 `MinKernel 25.0.0` / `MaxKernel 25.99.99` 把它限定在 Darwin 25，也就是只有在 macOS 26 上才加载。它是这台机器上唯一与系统版本强绑定的音频组件。

它的作用在项目描述里写得很直白：**restore native AppleHDA on macOS Tahoe through OpenCore and Lilu** —— 在 Tahoe 上把原生 `AppleHDA` 恢复回来。也就是说 macOS 26 改了 `AppleHDA` 的加载路径，只靠 AppleALC 补不上这一段，需要它先把框架装回去。

版本对照见 [sources.md](sources.md)。

## 验证

```bash
system_profiler SPAudioDataType | grep -E "Default (Output|Input)"
```

两个都该有返回值。只有输出没有输入，往下查 codec：

```bash
ioreg -l | grep -i "IOAudioDeviceName"
```

HDA 设备在、但没建出音频设备的，是 layout 不对。这块板子是 ALC257，走 Intel SST，先试标准 layout（`11`、`13`、`99`）再考虑别的。

## 不属于音频的那部分

麦克风**权限**是另一回事，而且麻烦得多。命令行能录、所有 GUI 应用被拒，原因在 TCC 层不弹窗。如果某个 App 用不了麦但 `ffmpeg` 能录，音频栈是好的，去看 [permissions.md](permissions.md)。

## 已知情况

- 输入源显示为「内置麦克风」，这个名字不能通过 `layout-id` 改。
- 音量键走标准 HID 路径，OSD 正常。
- 耳机孔检测没测过。这台机器平时用内置扬声器和一个 USB 声卡（FRANSUN KT02H20），后者作为独立设备枚举，不需要 kext。
