# 无线与蓝牙

两个独立的栈，两个不同的结果：Wi-Fi 好用，蓝牙能用但是弱的那一半。

## Wi-Fi

| 项 | 值 |
|---|---|
| 网卡 | Intel AX1690i（后换的卡），`8086:1692` |
| 驱动 | [`AirportItlwm.kext`](https://github.com/laobamac/itlwm) 2.4.0（**fork**） |
| 固件 | `itlwm: 2.4.0_738d4c6 fw: 68.01d30b0c.0` |
| 启动参数 | `itlwm_he=1` |
| 接口 | `en0` |

网卡是 **Intel AX1690i**，**自己换上去的**，替换了原厂的 CNVi 模块。PCI ID `8086:1692`。

旧笔记那句「fitted by the owner as a replacement for the factory card」是对的。但同一行的 ID 写的 `8086:7A70`，和这块卡对不上——AX1690i 是 `1692`。那一格大概是原厂卡留下的，或者抄错了，不用管它。

**这台机器能上 Wi-Fi，是两件事叠起来的**：

1. **换了卡** —— 原厂那块不在驱动支持范围内，换成 AX1690i
2. **换了驱动** —— 旧配置用 `itlwm.kext`，新配置用 `AirportItlwm.kext`

两者的差别在这儿：

| | `itlwm` | `AirportItlwm` |
|---|---|---|
| 呈现方式 | 伪装成以太网设备 | 系统原生 Airport 接口 |
| 系统 Wi-Fi 菜单 | 没有 | 有 |
| 扫描 / 关联 | 靠 HeliPort 之类的私有工具 | 系统自己处理 |

旧笔记里那条「Wi-Fi ❌ 驱动不存在」，记录的是 `itlwm` 时期的状态。

上游 [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) 只到 **v2.3.0（2024-06）**，不支持 macOS 26。本机的 2.4.0 来自 [laobamac/itlwm](https://github.com/laobamac/itlwm) 的 `v2.4.0-alpha`，bundle identifier 仍是 `com.zxystd.AirportItlwm`。**直接下上游 2.3.0 是装不上的**，对照见 [sources.md](sources.md)。

### 实际协商到什么

在 `-34 dBm` 下测的——信号强到不能更强，所以速率不是距离限制的：

| 字段 | 值 |
|---|---|
| PHY 模式 | **802.11ac** |
| 信道 | 157（5 GHz，**80 MHz**） |
| Tx 速率 | 867 Mbps |
| MCS / NSS | 9 / 2 |
| 国家代码 | CN |

网卡在 `Supported PHY Modes` 里报了 `802.11 a/b/g/n/ac/ax`，`itlwm_he=1` 也加了，但链路仍然以 ac / 80 MHz 起来。同一个 AP 上 ax / 160 MHz 大约快三倍。

在怪驱动之前先查两件事：路由器 5 GHz 频段有没有开 802.11ax（很多默认只开 ac 兼容），以及允不允许 160 MHz。两个都开着还是 ac，那限制在驱动的 HE 处理上。

### 检查

```bash
system_profiler SPAirPortDataType | grep -E "PHY Mode|Channel"
sudo wdutil info
```

`wdutil` 比菜单栏给的多。按住 Option 点 Wi-Fi 图标也能看到同样字段，包括 RSSI 和噪声——只是这个驱动把噪声报成 `0 dBm`，那一位没有意义。

### AirDrop 用不了

不是蓝牙问题，也不是加 kext 能解决的。AirportItlwm 没有 AWDL 接口，而 AirDrop、Handoff、Continuity 全都走它。见 [limits.md](limits.md)。

## 蓝牙

| 组件 | 版本 |
|---|---|
| [`IntelBluetoothFirmware.kext`](https://github.com/lshbluesky/IntelBluetoothFirmware) | 2.5.0（**fork**） |
| [`IntelBTPatcher.kext`](https://github.com/lshbluesky/IntelBluetoothFirmware) | 2.5.0（**fork**） |
| [`BlueToolFixup.kext`](https://github.com/acidanthera/BrcmPatchRAM) | 2.7.3 |
| 启动参数 | `-ibtcompatbeta` |

三个 kext，三件事。固件驱动上传 blob；`IntelBTPatcher` 打新版系统的兼容补丁；`BlueToolFixup` 是 Lilu 插件，让控制器伪装成 Apple 的。

最后这个是为什么系统信息里显示 `Chipset: THIRD_PARTY_DONGLE`、`Vendor 0x004C` 而不是 Intel。这是故意的——不伪装 macOS 26 根本不 attach。

蓝牙同样需要 fork：上游 [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) 最新 v2.4.0（2024-02），本机 2.5.0 来自 [lshbluesky](https://github.com/lshbluesky/IntelBluetoothFirmware) 的 fork。

`-ibtcompatbeta` 是 `IntelBTPatcher` 读的，不是固件驱动。它在这里有没有用还没定论：加了，蓝牙能连，部分 BLE 设备仍然掉；没有做过移除后的 A/B 对比。

### 检查

```bash
system_profiler SPBluetoothDataType | head -20
```

`State: On` 加上附近设备列表，说明栈起来了。固件上传本身在内核日志里能看到：

```
IOUSBHostDevice: setConfigurationGated: IntelBluetoothFirmware selected configuration 1
```

`kernelmanager_helper: Could not find identifier (com.zxystd.IntelBluetoothFirmware)` 是注入型 kext 的正常提示，不是错误。

### 能连什么，不能连什么

**手柄和耳机正常。其他类型的设备不行**——要么配对失败，要么连上立刻掉。

单讲「蓝牙能用」不准确：两类最常用的外设都在能用的那一半，而这一类里的其余大部分都不在。这台机器实际配的是手柄（GameSir G7 Pro）和耳机，两个都稳定。

2.4 GHz 共存也是因素——两个天线都在屏轴里，Wi-Fi 一忙，蓝牙距离就受影响。

Handoff 和 Continuity 同样不能，原因和 AirDrop 一样是 AWDL。
