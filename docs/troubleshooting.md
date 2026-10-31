# 排错

按出现频率排序。

## 装完驱动后黑屏

1. 关机，进 picker
2. **按空格**显示辅助条目
3. 选 `1401: Start once without NVIDIA driver`
4. 进桌面后用 `1401.app` 重装驱动，重启

如果条目不见了，多半是 `1401.app` 把 `Misc → Tools` 那两条重置了——见 [gpu.md](gpu.md)。`1401: Remove NVIDIA driver` 是更重的手段：设 NVRAM 标志、系统起来后卸载、自己重启。

另外驱动也会自己回滚。新装失败两次，`lastgood.sh` 会从 `/Library/NullMoth/backup-<时间戳>/` 恢复上一版。在下结论之前先看 `lastgood.log`。

## 能进桌面，但很慢、分辨率很低

驱动没加载。

```bash
kmutil showloaded | grep -i nullmoth      # 期望 4 行
system_profiler SPDisplaysDataType | grep -i metal
```

四个 kext 加 `Metal 3`，说明它加载着，问题在别处。

## 完全没画面，但机器在跑

风扇转、键盘背光亮、屏幕上什么都没有。这是驱动加载了但没能把显示带起来。用 `Start once without NVIDIA driver` 引导。

## App 用不了麦克风或摄像头

先看权限列表：

- 列表里有这个 App → 关掉再打开
- 列表是空的 → [permissions.md](permissions.md)，这需要写数据库，开关解决不了

区分硬件和权限层，一条命令就够：

```bash
ffmpeg -f avfoundation -i ":0" -t 1 -y /tmp/mic.wav
```

能录出来，音频栈就是好的，问题在 TCC。

## Wi-Fi 能连但很慢

看协商模式，不是看信号强度：

```bash
system_profiler SPAirPortDataType | grep -E "PHY Mode|Channel"
```

就算加了 `itlwm_he=1`、信号也很强，这台机器现在协商的还是 `802.11ac` / 80 MHz。见 [wireless.md](wireless.md)。

## 触控板没反应

```bash
ioreg -c VoodooI2CHIDDevice | grep -c "class VoodooI2CHIDDevice"
```

`0` 表示栈没摸到设备。通常是 `-vi2c-force-polling` 没加，或者 `SSDT-GPI0.aml` 没加载。

## 更新永远是完整安装包

`revpatch=sbvmm` 丢了，或者被别的改动覆盖了。见 [updates.md](updates.md)。

## `Processor Name: Unknown`

`revpatch` 里的 `cpuname` 没生效。见 [power.md](power.md)——几乎都是 `boot-args` 盖掉了 NVRAM，不是 SMBIOS 坏了。

## picker 里多出不该出现的条目

`1401.app` 会重写 `config.plist`，把 `HideAuxiliary` 和工具条目的 `Auxiliary` 重置。在那个 App 里改过任何设置之后，两处都要重设一遍。

## 读日志

```bash
# 本次启动以来的内核错误
log show --last 5m --predicate 'process == "kernel" AND messageType == error' --style compact

# tccd 的判断
log show --last 1h --predicate 'process == "tccd"' --style compact | grep -i policy

# 驱动加载状态
kmutil showloaded | grep -v com.apple

# OpenCore 日志（如果开了文件日志）
cat /Volumes/1401/oc.log 2>/dev/null
```

`Misc → Debug → Target` 控制 OpenCore 的日志。`3` 只写控制台；加上 `0x40` 会往 EFI 分区写文件。那个文件是「macOS 还没起来就失败了」时唯一的记录，动任何有风险的东西之前先打开。

一个 `nvram` 的坑：它只列 Apple 自己的变量。第三方 GUID 下的（Mok、RestrictEvents）要通过设备树读：

```bash
ioreg -p IODeviceTree -n options -r -l -w0 | tr ',' '\n' | grep -i <名字>
```
