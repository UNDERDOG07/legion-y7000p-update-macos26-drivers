# 显卡（RTX 5060 Laptop）

macOS 26 里没有任何 Intel 核显驱动，所以内屏只能走独显。这让 NVIDIA 驱动成了整台机器的单点：它不加载，就没有画面。

驱动是 **NullMoth 2.6.5**，装在 `/Library/Extensions`，**不在 EFI 里**。这个区别解释了后面几乎所有问题。

## 磁盘上有什么

| 组件 | 路径 | 版本 |
|---|---|---|
| 内核扩展 | `/Library/Extensions/{NVRM,NVAccel,NVRMFB,NVRMAGDC}.kext` | 2.6.5 |
| Metal 驱动 | `/Library/GPUBundles/NVMTLDriver.bundle` | 2.6.5 |
| 共享库 | `/Library/GPUBundles/NVIDIAShared.bundle` | — |
| Metal 允许列表 | `/Library/GPUBundles/nvmtl-allow.txt` | — |
| GSP 固件 | `/Library/NullMoth/nvfw/nvidia/615.78.08/` | 615.78.08 |
| 驱动状态 | `/Library/NullMoth/driver-version`、`os-major` | `2.6.5`、`26` |

四个 kext 的 bundle identifier 前缀都是 `com.nullmoth.*`。驱动本身不随本仓库分发，来源见 [sources.md](sources.md)。

硬件侧供对照：`10DE:2D59`，subsystem `17AA:802F`，显存 8 GB（BAR1 8192 MB），PCIe x8，Metal 3。

## 为什么不在 EFI 里

EFI 只放内核启动完成之前就必须存在的东西：OpenCore 本身，以及引导路径要用的 kext（触控板、无线、蓝牙）。显卡驱动是系统扩展，在内核起来之后才加载。

三个直接后果：

- 换 EFI、改 `config.plist`，都修不了显卡
- 系统更新会重建内核集合，驱动可能掉出去
- 重装系统要重装驱动，它不来自 U 盘

> 这台机器是**从 macOS 15 升级**上来的，驱动在升级中活了下来。那是运气，不是规律。全新安装时 `/Library/Extensions` 是空的。

## 安装，以及什么时候要重装

正常路径是 `1401.app`：写入四个 kext、重建启动用内核集合、要求重启。

装之前先看有没有残留：

```bash
ls /Library/Extensions | grep -i nv
```

装完确认四个都起来了：

```bash
kmutil showloaded | grep -i nullmoth
```

不看这些也想快速判断的话：`system_profiler SPDisplaysDataType` 里还能打印 `Metal Support: Metal 3`，就说明活着。

需要重装的时机：

- macOS 更新之后（内核集合重建）
- 桌面掉到低分辨率，或者 Metal 消失
- `kmutil showloaded | grep -i nullmoth` 少于四行

## 驱动会自己修自己

这是 2.6.5 才有的，旧笔记里没有。`/Library/NullMoth/` 下有三个不需要你干预的脚本：

| 脚本 | 触发 | 做什么 |
|---|---|---|
| `bootguard.sh` | 开机时的 LaunchDaemon | 等 `WindowServer` 起来，确认 45 秒后还活着，然后把 `nullmoth-nv-ok=1` 写进 NVRAM |
| `lastgood.sh` | 开机时的 LaunchDaemon | 新驱动**连续两次**没把画面带起来，就从 `/Library/NullMoth/backup-<时间戳>/` 恢复上一个版本 |
| `nullmoth-recover.sh` | NVRAM 里有 `nullmoth-remove` 标志 | 执行 `nullmoth-setup.sh --remove`，然后自己重启 |

由此有两个结论。

**「Remove NVIDIA driver」那条 picker 选项，在启动时并不删除任何东西。** 它只是设一个 NVRAM 变量；卸载发生在 macOS 起来之后，结束时机器自己重启。这也是为什么黑屏状态下它依然有效。

**装失败的驱动通常会自己回滚。** `lastgood.log` 就是记录，一行一次：

```
Tue Oct 27 17:26:16 CST 2026 the new driver brought its display up; keeping it
```

`bootguard.log` 记的是相反的那一类——算作成功的开机，在 `WindowServer` 起来 45 秒后写入。

## 黑屏：两个 picker 选项

它们在 `Misc → Tools` 里，都指向 `NullMothSafe.efi`，只靠参数区分：

| 条目 | Arguments | 效果 |
|---|---|---|
| `1401: Remove NVIDIA driver` | 空 | 设置 NVRAM 标志 → 卸载，然后重启 |
| `1401: Start once without NVIDIA driver` | `once` | 只这一次不加载驱动 |

平时两个都隐藏（`Auxiliary = true`）。**在 picker 里按空格**才会显示。

先试第二个：它能在不动已安装驱动的前提下进桌面，从那里再用 `1401.app` 重装。

## config.plist 会被重写

每次在 `1401.app` 里改设置——比如打开内屏亮度控制——它都会整个重写 `config.plist`。有两处会被重置：

- `Misc → Boot → HideAuxiliary` 回到 `false`
- 两个 `NullMothSafe.efi` 工具条目回到 `Auxiliary = false`

不是 bug：那个 App 不知道手动设过的隐藏标记。每次改完要重新设一遍：

```
Misc → Boot → HideAuxiliary = true
Misc → Tools[Path = NullMothSafe.efi] → Auxiliary = true    （两条都改）
```

隐藏不影响使用，按空格照样出来。

## 刷新率与镜像

单独使用时，内屏的模式列表里有 2560×1600 @ 240 Hz。

接上外接屏做镜像，内屏跟随主屏降到 2560×1440 @ 60，外接屏作为 Master Mirror 跑 180 Hz。

这个状态下亮度调节是失效的——面板此时是硬件镜像。拔掉外接屏就恢复。那是合成器行为，不是显卡驱动的问题，见 [power.md](power.md)。

## 视频解码

这块 GPU 在 macOS 下没有 NVDEC，iGPU 也没有驱动，所以解码路径里没有硬件解码单元。

但 VideoToolbox 有一条走 Metal 的路。1080p HEVC、十秒素材、`ffmpeg -benchmark`、各跑三轮，实测：

| 路径 | utime |
|---|---|
| `-hwaccel videotoolbox` | 0.61 – 0.65 s |
| 纯 CPU（ffmpeg 自带解码器） | 1.67 – 2.61 s |

走 VideoToolbox 快 2.5–4 倍。运行时 `VTDecoderXPCService` 会加载 `NVMTLDriver`，所以这是 Metal 参与的软件解码，不是解码引擎。

H.264 反过来：videotoolbox 0.82 s，纯 CPU 0.59 s。

这组数字描述的是命令行解码，不代表播放器的表现。写在这里只为说明一件事：**没有硬件解码器，不等于解码慢。**

## 已知限制

- **DRM** —— Apple TV、Netflix 播不了。没测，也不值得测：驱动里没有 HDCP 路径。
- **睡眠** —— 唤醒后 GPU 回不来。整机层面的限制，见 [limits.md](limits.md)。
- **外接显示器**走 dGPU 的 HDMI，与 BIOS 的 MUX 无关。这也是内屏黑掉时唯一还能用的输出。
