# 启动参数

三十二个。前十四是基础，后十八是 AirDrop 增量，按 `boot-args` 里的顺序：

```
debug=0x100 keepsyms=1 -vi2c-force-polling ctrsmt=full
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
itlwm_he=1 revpatch=sbvmm,cpuname -ibtcompatbeta

# AirDrop（ForbiddenItlwm）追加：
-btleadv -itlawdlcoexprobe -itlawdliface -itlawdlnativeup -itlawdlcoexrx
-itlawdlcoextx -itlawdlcoexmulti -itlawdlcoexdata -itlawdlmitx -itlawdlelicit
-itlawdlcaps -itlawdlsvcmerge -itlawdlawsync itlawdlintcoal=4
-itlawdlcoexbatx -itlawdlcoexpeerpause -itlawdlcoexbatrace itlawdltxqueue=4096
```

它们分成几组，可信度各不相同。有些是拆掉之后看机器坏掉才确认的，有些是从驱动包照抄过来、从没单独验证过。下面标了区别。

## 另外一切依赖的两个

**`amfi=0x80`** 关掉 AMFI。NVIDIA 的 kext 未签名，没有它就不加载，也就没有画面。`amfi_get_out_of_my_way=0x1` 是在上一层放宽同一个检查，`ipc_control_port_options=0` 挡住 Ventura 之后加的 Mach IPC 限制，免得它干扰同一条路径。三个一起，在这台机器上没有可选空间。

代价也明确：AMFI 关掉之后，macOS 不再为摄像头和麦克风弹授权窗。见 [docs/permissions.md](permissions.md)。

**`ctrsmt=full`** 由 `CpuTopologyRebuild` 读取。去掉它，机器暴露 24 个核，macOS 会把 16 个 E 核当 P 核调度。加上之后，E 核作为 SMT 兄弟呈现：8 物理、32 逻辑，也就是现在 `hw.ncpu` 报的值。

## 显卡

`nvfb=1`、`nvaccel=1`、`nvfbheads=4`、`-nvkmsnosmooth`。

四个都来自 NullMoth 包自带的说明，本仓库没有逐个移除验证过。按顺序是：启用帧缓冲、启用加速、保留四个帧缓冲 head、关掉 KMS 平滑过渡。

名字描述了意图，不等于每一个都单独验证过。排查画面问题时这四个是最先可以动的，但一次只改一个，把能用的组合记在文本里。

## 引导路径

`debug=0x100` 和 `keepsyms=1` 让 panic 信息和符号名在崩溃后仍然可用。代价是一点内存。留着——这台机器唯一那次 panic 完全靠这两个定位。

`-vi2c-force-polling` 让 VoodooI2C 轮询而不是等中断。去掉它触控板就没有反应：kext 照样加载，设备永远不报输入。见 [docs/input.md](input.md)。

## 无线与蓝牙

`itlwm_he=1` 让 AirportItlwm 声明支持 HE（802.11ax）。网卡在 `Supported PHY Modes` 里报了 ax，但这个参数在实测中没有带来变化——链路仍然以 802.11ac 起来。数据见 [docs/wireless.md](wireless.md)。

`-ibtcompatbeta` 由 `IntelBTPatcher` 读取，不是固件驱动。**当前配置下它是空的**：`IntelBTPatcher` 已禁用以给 `BTLEAdvFixup` 让位，没有程序解析这条参数。留着只为回退时少改一处。

### AirDrop 的 18 个参数

这一组只在 `ForbiddenItlwm` 下有意义。按作者 `docs/BOOT-FLAGS.md` 里的 "tested set" 原样照抄，没有逐条做过移除对比——换句话说，**知道它们各自声称做什么，但不知道哪几条在这台机器上是必需的**。

| 组 | 参数 | 作用 |
|---|---|---|
| 接口与监听器 | `-itlawdliface` `-itlawdlcoexprobe` `-itlawdlnativeup` `-itlawdlcoexrx` `-itlawdlcoextx` | 创建 `awdl0`、开第二个固件上下文、放行 macOS 的 AWDL 对等体管理器、收帧、给监听器独立队列 |
| 共存调度 | `-itlawdlcoexmulti` `itlawdlintcoal=4` `-itlawdlcoexpeerpause` `-itlawdlcoexbatx` `itlawdltxqueue=4096` | 监听器独立信道、缩短中断合并、对端离开时暂停队列、出向 Block Ack、发送队列长度 |
| 宣告与发现 | `-itlawdlmitx` `-itlawdlelicit` `-itlawdlcaps` `-itlawdlsvcmerge` `-itlawdlawsync` | 发自己的 AWDL 宣告、补齐 Apple 设备期待的字段、上报两空间流与 VHT、合并服务记录、对齐对端可用窗口 |
| 数据通路 | `-itlawdlcoexdata` `-itlawdlcoexbatrace` | 双向承载 AirDrop 数据；后者只在 I/O Registry 里记 Block Ack 事件，诊断用 |
| 蓝牙 | `-btleadv` | 打开 `BTLEAdvFixup`，iPhone 才能通过 BLE 发现这台 Mac |

两个后果要记住：

- **升级 macOS 前要去掉 `-btleadv`**，否则增量更新可能失败，见 [docs/updates.md](docs/updates.md)
- 换回 `AirportItlwm` 时这 18 个应当一并移除。留着不会伤害什么，但它们对 `AirportItlwm` 没有任何作用，只会让配置看起来比实际复杂

## 更新与 CPU 名

`revpatch=sbvmm,cpuname` 由 `RestrictEvents` 读取，同时做两件不相关的事：

- `sbvmm` 上报 VMM 安全启动模型，这是让不支持的板子拿到增量更新的前提。见 [docs/updates.md](updates.md)。
- `cpuname` 允许覆盖 CPU 名，也就是把 `i9-14900HX` 放回「关于本机」。没有它，名字从 SMBIOS 推导，推出来是空的。

两者在同一个 token 里。`boot-args` 里任何重写 `revpatch=` 的改动都会同时替换掉两条——CPU 名丢了一周就是这么来的，见 [docs/power.md](power.md)。

`NVRAM → Add` 的 RestrictEvents GUID 下还有一份 `revpatch`，里面已经带着 `cpuname`。**`boot-args` 优先。** 两份都留着无害，但容易混；NVRAM 那份是备份，不是配置。

## 削减顺序

想看清这串里有多少是必需的，按这个顺序移除并记录：

1. `debug=0x100`、`keepsyms=1` —— 安全，只影响诊断
2. `itlwm_he=1` —— 预期没有可见变化
3. `-ibtcompatbeta` —— 预期没有可见变化
4. `nvfbheads=4`、`-nvkmsnosmooth` —— 一次一个，观察内屏有没有闪烁

`amfi=0x80`、`ctrsmt=full`、`revpatch=sbvmm` 不要动，除非你打算放弃显卡驱动、CPU 拓扑或增量更新。
