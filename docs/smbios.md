# 机型与序列号

## 当前身份

| 字段 | 值 |
|---|---|
| `SystemProductName` | `MacBookPro16,4` |
| Board ID | `Mac-A61BADE1FDAD7B05` |
| CPUID 伪装 | family 6 / model `0xA5` / stepping 5 |

`MacBookPro16,4` 是 2019 年 16 寸、配 Radeon Pro 5600M 的那台——苹果最后一款 Intel 大屏机，8 核，带独显。对一台内屏必须由 dGPU 驱动、又要跑 macOS 26 的笔记本来说，这是最接近的匹配。

`Kernel → Emulate → Cpuid1Data = 55 06 0A 00` 报的是 model `0xA5`（Raptor Lake），而不是真实的 `0xB7`（Raptor Lake Refresh）。`0xA5` 的电源管理路径被验证得更充分，这台机器上一直稳定。

## 序列号决定「关于本机」显示什么

这是花掉一个下午的那个坑。

「关于本机」的机型名**不读 SMBIOS**。它取序列号的**末四位**，去问 Apple：

```bash
curl "https://support-sp.apple.com/sp/product?cc=ML7H&lang=en"
```

返回什么，界面就显示什么。所以一个末四位属于别的机型的序列号，会造出一台内部自洽、显示错误的机器：

| 字段 | 值 | 指向 |
|---|---|---|
| SMBIOS 产品名 | `MacBookPro16,4` | 16 寸 2019 |
| Board ID | `Mac-A61BADE1FDAD7B05` | 16 寸 2019 |
| 序列号（旧） | 末四位 `ML7H` | **13 寸 2020，四个雷雳口** |

除了序列号，全对；显示出来是错的。那个旧号还是一台真实机器的号，这本身是另一个问题——iServices 是按序列号认设备的。

## 修法

按目标机型生成，然后查配置码，别只信工具输出：

```bash
./macserial -m MacBookPro16,4 -n 10
```

`MacBookPro16,4` 的配置码是 `MD6T`。末四位为 `MD6T` 的候选会解析成「MacBook Pro (16-inch, 2019)」：

```bash
curl "https://support-sp.apple.com/sp/product?cc=MD6T&lang=en"
```

三个字段要一起换：

- `SystemSerialNumber`
- `MLB` —— 必须和序列号同代
- `SystemUUID` —— 重新生成，新的 v4 就行

`ROM` 没动。理想值应该是 `en0` 的 MAC，但改它又是一次身份变动，而当时服务是好的。

## 改之前

先退出 Apple ID，包括「查找我的 Mac」。带着登录状态改序列号，账号上会留一条幽灵设备记录，清理起来很烦。

之后 iServices 可能会要验证。如果直接拒绝，等 24 小时，别反复试——重复尝试会抬高风控等级。

`~/Library/Preferences/ByHost/` 下按旧 `IOPlatformUUID` 命名的条目会失效，显示器和声音偏好可能重置一次。LaunchAgent 不受影响，它是按 label 存的。

## 读 SMBIOS 表

`ioreg` 不直接暴露那些字符串，表是二进制块。要读得自己解析：

```bash
ioreg -c AppleSMBIOS -r -l -w0 | grep '"SMBIOS" = <'
```

把十六进制解出来，按结构走：Type 1 是产品名和序列号，Type 4 是处理器字符串。CPU 名显示 `Unknown` 那次就是这么查出来的——SMBIOS 里字符串一直是对的，只是 macOS 没用它。
