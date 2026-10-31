# 键盘与触控板

## 键盘

两块键盘，处理方式不同。

| 设备 | 总线 | 驱动 |
|---|---|---|
| 内置 | PS/2 | [`VoodooPS2Controller`](https://github.com/acidanthera/VoodooPS2) |
| 外接（MCHOSE A7 V2 Pro+） | USB HID | 系统原生，不需要 kext |

外接是 Windows 布局，物理位置上 `Ctrl / Win / Alt` 对应 macOS 的 `Ctrl / Option / Command`。

### 修饰键

**不需要任何映射。** 两块键盘发出来的修饰键位置本来就是 macOS 期望的，`hidutil property --get UserKeyMapping` 返回 `null` —— 这是**正确状态**，不是缺配置。

交换 Option 和 Command 是拿到 Windows 键盘后最常做的第一件事，在这台机器上那么做反而是错的。外接键盘的 `Win` 键就是 Command，`Alt` 就是 Option，不用改。

真遇到需要交换的键盘，**系统设置 → 键盘 → 键盘快捷键 → 修饰键** 里有按键盘区分的面板。需要的时候用它比 `hidutil` 好：能跨系统更新存活，也不需要登录项。

### 功能键

F 区发的是标准功能键，亮度和音量在 Fn 层。这个机型的原厂行为就是这样，与 Windows 下一致。

## 触控板

能用。**这不是系统更新带来的，是把 config 里一条失效引用改掉之后才通的。**

`VoodooInput` 是触控板输入协议那一层。**VoodooI2C 2.9.1 的包里只有 `VoodooGPIO` 和 `VoodooI2CServices`，没有 `VoodooInput`** —— 它依赖 VoodooPS2 提供的那份。

旧配置里有两处 `VoodooInput` 条目：

- 指向 `VoodooI2C.kext/Contents/PlugIns/VoodooInput.kext` 的，**启用**，但那个路径下没有文件
- 指向 `VoodooPS2Controller.kext/Contents/PlugIns/VoodooInput.kext` 的，**禁用**

结果就是这一层根本没加载。kext 全都起来了、设备也在，触控板一点反应都没有——旧笔记那句「栈都在，从没响应过」说的就是这个。

现在的组合：

| 组件 | 版本 | 状态 |
|---|---|---|
| [`VoodooI2C.kext`](https://github.com/VoodooI2C/VoodooI2C) | 2.9.1 | 启用 |
| `VoodooI2CHID.kext` | 1.0 | 启用 |
| `VoodooGPIO.kext` | 1.1 | 启用 |
| `VoodooI2CServices.kext` | 1.0 | 启用 |
| `VoodooPS2Controller` 的 `VoodooInput.kext` | 1.1.7 | **启用**（关键） |
| `VoodooPS2Mouse.kext` / `VoodooPS2Trackpad.kext` | 2.3.8 | **禁用** |
| 启动参数 | | `-vi2c-force-polling` |

改动是：删掉那条指向不存在文件的条目，把 VoodooPS2 的 `VoodooInput` 打开，PS2 的鼠标和触控板插件关掉。

设备是 GPIO 上的 I2C HID，`INTC1085`。`ioreg` 里能看到一个 `VoodooI2CHIDDevice` 实例。

`-vi2c-force-polling` 在旧配置里就有，一直都在。它让驱动轮询而不是等 GPIO 中断——去掉之后 kext 照样加载，设备永远不报输入。它是必要条件，不是充分条件：旧配置有它，触控板照样不动。

### 检查

```bash
ioreg -c VoodooI2CHIDDevice | grep -c "class VoodooI2CHIDDevice"
```

`1` 表示设备附着上了。`0` 表示栈没摸到硬件——先确认 `SSDT-GPI0.aml` 有加载、ACPI 里的 `GPI0 _STA → XSTA` 改名是启用的，因为 GPIO 控制器得交给 kext 而不是固件。

## RGB 灯

四区 RGB 是独立的 USB HID 设备（`048d:c995`），macOS 没有驱动。键盘背光本身能用，因为那是固件控制的，不需要操作系统知道。
