# 3. 调试方法

问题是怎么找到的，整理成可以用在其它相机或 App 版本上的形式（macOS）。

## 工具

```bash
brew install openjdk apktool jadx
brew install --cask android-commandlinetools android-platform-tools
sdkmanager "build-tools;35.0.0"
export JAVA_HOME=/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home   # apksigner 需要真正的 JDK
```

- 用 **jadx** 读代码，改 `apktool d -r` 输出的 **smali**。类名基本没混淆，直接搜 `UsbSupportedCameraManager`、`switchFunctionMode` 等即可。
- 用 **`-r`**（不解码资源），资源文件保持原样。界面改动放在代码里（例如 `setVisibility`）。
- 在不区分大小写的 APFS 上，apktool 会给重名类改名。按 `.class` 行识别文件，不要按路径。

## 先拿证据

1. **正式版没有日志。** `common/log/AdbLog` 的方法体不输出任何内容，也没有开关，所以调试版注入了追踪代码（[D1、D2](02-patches.md#仅调试版)）。
2. **用无线 adb**，手机 USB 口插着相机。开发者选项 → 无线调试 → 使用配对码配对。
3. **插相机之前开始抓日志**，搜索时用 `grep -a`（logcat 里有特殊字节，不加 `-a` 会被当成二进制文件）：
   ```bash
   adb logcat -c && adb logcat -v time > run.log
   grep -a ZVPATCH run.log
   ```
4. **ColorOS/OxygenOS 会丢日志**：单个 App 每秒超过 300 行的部分被丢弃（`LOGS OVER PROC QUOTA(300) … DROPPED`），缺某一行不能说明没发生。

## 常用日志

| 关键词 | 含义 |
|---|---|
| `INIT OK mode=` / `INIT FAILED code=` | 会话初始化结果 |
| `PTP FAIL op=` | 找第一条；之后成片的 `GeneralError` 都是它的后果 |
| `SWITCH function mode` / `TERMINATE` / `DISCONNECT` | 会话被关闭，下面是调用栈 |
| `SessionAlreadyOpen` | 相机还留着上次的会话，App 会关掉后重试，无害 |
| `UsbPermissionActivity` | 弹出 USB 授权框（可能是别的 App 申请的） |
| `FATAL EXCEPTION` | 崩溃 |

## 别的 App 抢占 USB

相机连着时，别的 App 可能申请访问它（ColorOS 相册打开时就会）。允许后实时画面停止，要重新插拔数据线才能恢复；点取消不受影响。怀疑补丁有问题之前，先排除这一点。

## 方法

- 把相机 DeviceInfo 里的命令列表和 App 发出的每条命令对比，包括会话打开之后才发的。
- 转圈和 "Could not connect" 只是现象，要找第一条失败的命令，或者关闭会话的调用栈。
- 注意界面恢复显示、切换标签页时做了什么，会话模式常常在这里被改掉。
- 一次只改一处。装机前用 `dexdump -d` 确认改动进了 DEX，用 `apksigner verify --print-certs` 确认所有 APK 证书一致。
- 没搞清失败原因时，不要改错误处理去"容错"。
