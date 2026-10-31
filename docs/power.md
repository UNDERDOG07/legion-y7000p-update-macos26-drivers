# 电源、CPU 与亮度

## 电池

| 组件 | 版本 |
|---|---|
| [`VirtualSMC.kext`](https://github.com/acidanthera/VirtualSMC) | 1.3.9 |
| `SMCBatteryManager.kext` | 1.3.9 |
| `SMCProcessor.kext` | 1.3.9 |
| `SMCSuperIO.kext` | 1.3.9 |
| [`YogaSMC.kext`](https://github.com/zhen-zen/YogaSMC) | 1.5.3 |
| `SMCLightSensor.kext` | 1.3.9 |

电池读数是准的：设计容量 5545 mAh，当前最大 5154 mAh（93%），循环 76 次。

`YogaSMC` 处理联想自己的 EC：充电阈值、风扇表、额外的传感器。它需要 `SSDT-EC.aml` 把 EC 暴露出来，否则 kext 加载了也什么都不做。

`SMCSuperIO` 提供风扇和温度数据，但自己没有界面，要靠 `HWMonitorSMC2` 之类才能看到。

## CPU 拓扑

i9-14900HX 是 8 个 P 核、16 个 E 核、32 个线程。macOS 没有 E 核的概念，不管它的话会把 24 个核当成一样的来调度，单线程任务可能被扔到能效核上。

[`CpuTopologyRebuild.kext`](https://github.com/b00t0x/CpuTopologyRebuild) 配合 `ctrsmt=full` 修正呈现方式：E 核作为 P 核的 SMT 兄弟。

```bash
sysctl -n hw.physicalcpu hw.logicalcpu    # 8  32
sysctl -n hw.ncpu                          # 32
```

那个 `8` 不是少了核。它是预期的结果，也是这台机器在高负载下表现正常的原因。

## CPU 名字，以及它是怎么丢的

这个值得完整记下来，因为原因不在看起来的地方。

`RestrictEvents` 从 `revcpuname` 取 CPU 名，但前提是 `revpatch` 里有 `cpuname`。配置里两处都有，但分在两个地方：

```
NVRAM[RestrictEvents GUID]   revpatch   = sbvmm,cpuname
                             revcpuname = Intel(R) Core(TM) i9-14900HX
boot-args                    revpatch   = sbvmm
```

`boot-args` 优先于 NVRAM。于是 `cpuname` 那一半从来没生效，名字回退到 SMBIOS 报的，SMBIOS 报了个空的，「关于本机」就显示 `Processor Name: Unknown`。

修法是一个词：

```
revpatch=sbvmm,cpuname
```

两个教训。

第一，`revpatch` 是**一个 token**。为了修增量更新往 `boot-args` 里加 `sbvmm`，等于整体替换掉了 NVRAM 里那个带 `cpuname` 的值。两次改动看起来毫无关系。

第二，症状和系统更新出现在同一周，很容易被当成 Apple 的回归。实际不是——SMBIOS 表里那个字符串一直是正确的。

## 亮度

`1401.app` 里内置了内屏亮度控制。在那里打开之后，滑块就正常工作了，macOS 这边不需要任何配置。

这个控制之所以存在，是因为这台机器没有 `AppleBacklightDisplay` —— 平台的亮度通道根本不存在，没人供给的话，面板就停在驱动初始化时设定的那个值。

一个限制值得知道：接外接屏做镜像时，这个控制对内屏不起作用，因为此时面板是硬件镜像。拔掉外接屏就恢复。

## 睡眠

主动关掉的，不是 `pmset` 里的某个设置。见 [limits.md](limits.md) —— GPU 熬不过唤醒，睡下去就等于丢会话。
