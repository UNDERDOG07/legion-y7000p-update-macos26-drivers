# 这台机器做不到的事

下面四条不是「还没配置」，而是卡在配置层以下。原因比绕法更有价值。

## 睡眠

GPU 回不来。唤醒之后内屏是黑的，外接输出要么没有、要么是花的。

没有任何 `hibernatemode` 值、USB 供电设置或 `pmset` 参数能修好，因为故障在驱动的重新初始化路径里，不在电源管理。`/var/vm/sleepimage` 在这里也无关——休眠同样从来没成功过。

务实的做法是关掉睡眠，把合盖当熄屏用：

```bash
sudo pmset -a sleep 0
```

## AirDrop、Handoff、Continuity

三个都依赖 AWDL，那个点对点 Wi-Fi 接口。AirportItlwm 没有实现它，EFI 设置或 kext 都补不回来。

**不是蓝牙的问题。** 这句话值得直说，因为症状（近处设备之间传不了文件）读起来像蓝牙问题，会把人带偏。

## Secure Boot

两条路都试过，都被这台机器的固件堵死。

**自签 db。** 生成密钥、签 `BOOTx64.efi`、`OpenCore.efi` 和全部驱动、登记证书。这台机器的 BIOS 只提供一个破坏性的 setup mode，会清掉已有密钥，没有追加入口。要登记就得手工把微软的 PK/KEK/db 重新装一遍，把 Windows 的引导链也押上去。

**shim。** 先启微软签名的 `shim`，由它链到签过名的 OpenCore，再把证书登记进 MOK。链条本身是通的——shim 能加载、Bootstrap 能跑、OpenCore 能起来。失败的是 `MokList`：登记 `db.cer` 之后，通过 `IODeviceTree` 读回，所有 `Mok*` 变量依然不存在。BIOS 显示 `Device Guard` 处于活动状态，拒绝写入那个存储。

所以 Secure Boot 保持关闭，U 盘上的 EFI 本身是未签名的。**打开 Secure Boot 会让这台机器起不来。**

一个可以留着的诊断细节：`nvram -p` 只列 Apple 自己的变量，`Mok*` 看起来不存在也可能只是被过滤了。要读改用 `ioreg -p IODeviceTree -n options -r -l -w0`。

## 硬件视频解码

这块 GPU 在 macOS 下没有 NVDEC，iGPU 没有驱动，所以路径里没有解码引擎。

VideoToolbox 有一条走 Metal 的软件路线，HEVC 比 ffmpeg 的 CPU 解码快 2.5–4 倍，但那是软件解码，H.264 一点好处都拿不到。数据和测法见 [gpu.md](gpu.md)。

## 同样坏的

- **DRM 播放** —— Apple TV、Netflix。驱动里没有 HDCP 路径，不值得查。
- **Wi-Fi 6** —— 网卡支持 802.11ax，链路停在 ac / 80 MHz。见 [wireless.md](wireless.md)。
- **部分 BLE 外设** —— 能配对，随后掉线。固件层的，没有配置解法。
- **键盘 RGB 分区** —— 控制器是 USB HID 设备，macOS 没有驱动。背光能用，颜色不能。
