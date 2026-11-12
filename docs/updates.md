# 增量更新

macOS 只有在能验证安全启动链时，才会提供增量包（几百 MB 到 3 GB 左右），否则给的是完整安装包（10–15 GB）。在不支持的板子上这个校验会失败，所以要让系统自称一个 VMM 的安全启动模型。

## 需要的三样

缺一不可：

| 项 | 这台机器的值 |
|---|---|
| `Misc → Security → SecureBootModel` | `Disabled` |
| `RestrictEvents.kext` | 1.1.7，已启用 |
| `boot-args` | `revpatch=sbvmm` |

`sbvmm` 在上游的描述是：*forces VMM SB model, allowing OTA updates for unsupported models*。

## 为什么容易丢

`revpatch` 是一个 token。这份配置在 `NVRAM → Add` 的 RestrictEvents GUID 下还有一份副本，里面已经带着 `cpuname`：

```
boot-args   revpatch = sbvmm              ← 生效
NVRAM       revpatch = sbvmm,cpuname      ← 被忽略
```

为了修更新往 `boot-args` 里加 `sbvmm`，等于整体替换掉那个带 `cpuname` 的值，CPU 名字作为副作用消失。当时这两件事看起来毫无关系。详见 [power.md](power.md)。

动 `revpatch` 时，一次把完整列表写全：

```
revpatch=sbvmm,cpuname
```

## 怎么确认生效

唯一的可靠测试就是更新本身：

- 几百 MB 到 3 GB 左右 → 增量包，生效了
- 10–15 GB → 完整包，校验仍然失败

如果还是完整包，去读 `install.log` 和下载前后的 `softwareupdated` 记录，找具体的拒绝原因，别猜。

## 装了 AirDrop 之后多一条

`BTLEAdvFixup` 会改写 `bluetoothd` 的代码页。更新程序校验这个文件时读到的是打过补丁的内容，判定文件被改动，增量更新同样失败——和 `revpatch` 那条是两回事，需要各自处理。

每次更新前：

1. 从 `boot-args` 去掉 `-btleadv`（或把 `BTLEAdvFixup` 条目设成 `Enabled=false`）
2. 重启，正常进系统
3. 再做更新
4. 更新完先确认新版本仍在驱动支持列表内，再把 `-btleadv` 加回去

## 更新之后会坏什么

- **显卡驱动。** 内核集合被重建。用 `kmutil showloaded | grep -i nullmoth` 看，少于四行就用 `1401.app` 重装。
- **`/Library/Extensions` 里的东西。** EFI 之外安装的都在风险里。
- **`hidutil` 映射**，如果你用的是它而不是系统设置。
- **LaunchAgent 不受影响。** 亮度那个照常。

## 这不是 Secure Boot

`revpatch=sbvmm` 只是让系统**自称**一个 VM 安全启动模型。它不开启固件 Secure Boot，也不会让启动盘变成已签名。固件层面的 Secure Boot 必须保持关闭 —— 见 [limits.md](limits.md)。
