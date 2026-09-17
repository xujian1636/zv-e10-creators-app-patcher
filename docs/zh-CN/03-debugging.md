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

1. **正式版没有日志。** `common/log/AdbLog` 的方法体不输出任何内容，也没有开关，所以调试版注入了追踪代码（[D1–D3](02-patches.md#仅调试版)）。
2. **按测试内容选择 adb 连接方式。** 测 USB 时手机的 USB 口插着相机，要用无线 adb（开发者选项 → 无线调试 → 使用配对码配对）；测 Wi-Fi 时正好相反，把数据线插在电脑上，手机切到相机热点、离开家里网络时日志也不会断。
3. **连相机之前开始抓日志**，抓完整缓冲区，搜索时用 `grep -a`（logcat 里有特殊字节，不加 `-a` 会被当成二进制文件）：
   ```bash
   adb logcat -c && adb logcat -v time > run.log
   grep -a ZVPATCH run.log
   ```
   不要用 `adb logcat -s ZVPATCH` 过滤：过滤后的流在高负载下会丢行，本来正常的过程会看起来像是中途卡住。
4. **ColorOS/OxygenOS 会丢日志**：单个 App 每秒超过 300 行的部分被丢弃（`LOGS OVER PROC QUOTA(300) … DROPPED`），缺某一行不能说明没发生。
5. **没有调试版也能判断发生了什么**：`adb shell dumpsys usagestats` 能看出哪个界面在什么时刻位于前台，`ls -lt /sdcard/DCIM/CA_IMAGES` 能看出收到了什么文件，系统日志里有 Wi-Fi 连接（`connectToNetwork "DIRECT-…"`）、USB 插拔和崩溃记录。

## 常用日志

| 关键词 | 含义 |
|---|---|
| `INIT OK mode=` / `INIT FAILED code=` | 会话初始化结果 |
| `PTP FAIL op=` | 找第一条；之后成片的 `GeneralError` 都是它的后果 |
| `SWITCH function mode` / `TERMINATE` / `DISCONNECT` | 会话被关闭，下面是调用栈 |
| `SessionAlreadyOpen` | 相机还留着上次的会话，App 会关掉后重试，无害 |
| `UsbPermissionActivity` | 弹出 USB 授权框（可能是别的 App 申请的） |
| `DID name=… mediaServer=` | 响应的是哪套 Wi-Fi 服务：关闭为电脑遥控，开启为智能手机连接 |
| `STORAGE ids=` | `[]` 没有可传输的内容，`[VIRTUAL_MEDIA_1]` 相机上选中的影像，`[STORAGE_MEDIA_1]` 存储卡 |
| `LV setLiveViewStreamCallback` / `LV frames=` | 实时画面已启动，并且有帧到达 |
| `FATAL EXCEPTION` | 崩溃 |

## Wi-Fi 相关

- 测传输功能时相机不能停在电脑遥控模式：电脑遥控开着时，相机会拒绝*发送到智能手机*，屏幕提示*"该操作或设定无法如下进行"*。
- 两套 Wi-Fi 服务用的热点不同（`DIRECT-…` 的名字不一样），保存目的地等设置也各管各的，所以同一张遥控拍摄的照片，在一种模式下只回传 JPEG，在另一种下是 RAW+JPEG。
- 遥控模式的会话收不到影像。退出远程拍摄时 App 会把会话改回传输模式，这个过程约三秒：关闭、重开、重新读取存储和对象属性。

## 别的 App 抢占 USB

相机连着时，别的 App 可能申请访问它（ColorOS 相册打开时就会）。允许后实时画面停止，要重新插拔数据线才能恢复；点取消不受影响。怀疑补丁有问题之前，先排除这一点。

## 方法

- 把相机 DeviceInfo 里的命令列表和 App 发出的每条命令对比，包括会话打开之后才发的。
- 转圈和 "Could not connect" 只是现象，要找第一条失败的命令，或者关闭会话的调用栈。
- 注意界面恢复显示、切换标签页时做了什么，会话模式常常在这里被改掉。
- 一次只改一处。装机前用 `dexdump -d` 确认改动进了 DEX，用 `apksigner verify --print-certs` 确认所有 APK 证书一致。
- 没搞清失败原因时，不要改错误处理去"容错"。
