# 授权弹窗不出现时怎么办

这是这台机器上唯一没有上游解法的问题，也是最花时间才搞清楚的一个。诊断过程完整写在这里，因为症状指向的方向是错的。

## 症状

系统设置 → 隐私与安全性 → 麦克风是**空的**。不是"全部关闭"，是列表里一条记录都没有。应用申请麦克风，没有任何弹窗就被拒绝，然后提示用户去设置里打开，而设置里没有东西可打开。

摄像头表现相同，只是没那么稳定。

## 日志里到底是什么

```
tccd: Policy disallows prompt for Sub:{com.tencent.qq}
tccd: access to kTCCServiceMicrophone denied
```

`Policy disallows prompt` 的意思是：tccd 判定这次请求不允许展示给用户，于是不询问直接返回拒绝。没弹窗就没有记录，没记录列表就是空的，列表空就没有开关可打开。这个环自己闭合了。

三个看着像原因、其实不是的：

- **应用的签名。** `codesign --verify --deep --strict` 通过，Gatekeeper 报 `accepted`，designated requirement 和存的对得上。
- **`csreq` 数据。** 用 `csreq -r file -t` 解出来就是应用的 requirement —— `identifier "com.tencent.qq"`，团队 ID 一致。
- **`Policy disallows prompt` 这条消息本身。** `UserEventAgent`、`routined`、`managedcorespotlightd` 也会出。后台守护进程本来就不该弹窗，这条是常态；异常的是**前台应用**出现它。

## 可能的原因

AMFI 是关的（`amfi=0x80`），因为 NVIDIA 的 kext 未签名。tccd 判断"能否弹窗"要经过代码签名验证，AMFI 关掉之后那条路是残缺的，保守的结果就是拒绝。

这是推断，不是证明。能确认的是：这台机器上重新打开 AMFI 不是选项——没有它驱动不加载，没有驱动就没有画面。权限是更便宜的那个损失。

有一个假设被试过并否掉了：移除 `revpatch=sbvmm` 重启，行为没有变化。VMM 那条不是原因。

## 真正关键的地方：有两个数据库

```
/Library/Application Support/com.apple.TCC/TCC.db          系统
~/Library/Application Support/com.apple.TCC/TCC.db         用户
```

**应用级授权存在用户库里。**

往系统库里写一条 `kTCCServiceMicrophone`，记录看起来正确、读回来也正确，但什么都不会发生——应用照样被拒，设置面板照样是空的，因为这两处读的都是用户库。

给出决定性线索的是：用户库里本来就有一条 `com.tencent.qq` 的 `kTCCServiceAudioCapture`，日期在升级之前，而系统库里没有任何麦克风记录。

## 有效的记录长什么样

| 列 | 值 | 说明 |
|---|---|---|
| `service` | `kTCCServiceMicrophone` | 也可以是 `kTCCServiceCamera`、`kTCCServiceScreenCapture` |
| `client` | bundle id | |
| `auth_value` | `2` | 允许 |
| `auth_reason` | **`1`** | 用户设置。`4`（系统设置）处理路径不同，实测无效 |
| `csreq` | 应用的 designated requirement | 同一应用已有记录的话直接复用 |
| `last_modified` | 当前时间 | |

**两个库都要写**，写之前先停 `tccd`：

```bash
sudo killall tccd
# 往系统库写，再往用户库写同样的行
sudo killall tccd
```

起作用的是用户库；系统库那份让 `tccgrant list` 看到的状态是完整的。

## tccgrant

这里用的脚本把上面这些包好了。`csreq` 分三级取：复用同一 bundle id 的现有记录、用 `codesign -d -r-` 交给 `csreq` 现算、实在没有再留空。写之前备份两个库，写的时候放在一个事务里。

```bash
tccgrant list                              # 两个库全看
tccgrant check mic com.tencent.qq          # 单应用单服务
tccgrant mic com.tencent.qq                # 授予
tccgrant mic,camera /Applications/QQ.app   # 多服务，也接受路径
tccgrant revoke mic com.tencent.qq         # 撤销
```

服务名：`mic`、`camera`、`screen`。

授完之后必须**完全退出**应用（⌘Q，不是关窗口）再打开。tccd 在进程启动时读授权。

### 一个值得记的 shell 坑

第一版跑出来屏幕上显示 `授予 ��`。原因是 bash 里 `"$client："` 这种写法——变量后面紧跟全角字符，shell 会把那个字符的首字节吞进变量名，结果变量变空、字符残缺。要写 `"${client}："`。

## 还没解决的

每装一个需要麦克风的新应用，都会撞同一堵墙。除了重新打开 AMFI（代价是显卡驱动），没有别的办法让弹窗出现。

所以流程是：装应用、试一次、看它失败、`tccgrant mic <bundle-id>`、重启应用。

摄像头和屏幕录制情况好一些——部分授权是通过真实弹窗建立的，说明限制是按服务施加的，不是全局。先试正常路径，列表确实空了再上脚本。
