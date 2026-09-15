# ZV-E10 × Creators' App

给索尼 **Creators' App** 安卓版（3.5.0）打补丁，让它能通过 **USB** 遥控 **ZV-E10** 拍摄：实时画面、拍照并自动传到手机、录像、调曝光。附带说明 App 为什么用不了这台相机，以及改了什么。

[English](README.md)

> 非官方项目，与索尼无关。仓库不含索尼的代码或二进制文件：原版 App 由你自己提供，补丁工具在本地修改，风险自负。

## 兼容性

Creators' App 的支持机型列表里有 ZV-E10 II，没有初代 ZV-E10，下面的功能都是本项目新适配的。打补丁后实测：

| 功能 | 状态 |
|---|---|
| USB 连接 | ✅ |
| Wi-Fi 连接 | ❌ |
| 实时画面 | ✅ |
| 拍照 | ✅ 需按住快门键；照片自动传到手机 |
| 录像 | ✅ |
| 曝光模式、曝光补偿、ISO | ✅ |
| 导入（从存储卡拷贝） | ❌ 相机不开放存储卡，按钮已隐藏（[原因](docs/zh-CN/04-status.md)） |

测试环境：App 3.5.0，ZV-E10 固件 2.03，Android 16。相机设置：*电脑遥控：开*，连接方式 *USB*。

## 原版 App 为什么用不了

1. App 按**支持机型列表**检查型号，直接拒绝 ZV-E10。
2. 插上相机后，App 用**传输内容**模式打开会话。电脑遥控模式下 ZV-E10 不支持，界面一直转圈。
3. 相机页每次显示，都会把会话**切到传输内容模式**，远程拍摄提示 *"Could not connect to your camera via USB."*。
4. 会话打开后 App 发送 **`SDIO_GetDeviceLog`**，ZV-E10 没有实现这条命令，报错后整个会话失效：实时画面转圈，快门提示 *"Could not perform."*。
5. 去过**首页**后，App 重连时又用传输内容模式。

补丁 P1–P5 依次解决以上问题，P6 隐藏用不了的导入按钮，详见[文档](#文档)。

## 文档

1. [App 如何与相机通信](docs/zh-CN/01-how-it-works.md)
2. [问题与补丁](docs/zh-CN/02-patches.md)
3. [调试方法](docs/zh-CN/03-debugging.md)：注入 PTP 追踪、无线 adb、别的 App 抢占 USB
4. [现状与限制](docs/zh-CN/04-status.md)

## 打补丁（macOS）

```bash
brew install openjdk apktool
brew install --cask android-commandlinetools
sdkmanager "build-tools;35.0.0"

./patcher/patch.sh "Creators' App_3.5.0.xapk" out/
```

输入的 XAPK 中，base APK 的 SHA-256 必须是 `b0716bf5b02b7c723d54d670d97c1c6c6daf0519a5ded5330fc4d9f3ff443f09`。输出：

- `CreatorsApp_3.5.0_ZV-E10_release.xapk`：打好补丁的 App
- `CreatorsApp_3.5.0_ZV-E10_debug.xapk`：同上，额外带 PTP 追踪日志，用于反馈问题
- `zvpatch-signing.jks` / `.pass`：你的签名密钥，请保存好，以后才能覆盖安装新版本

先卸载原版 App（签名不同），再用 XAPK 安装器安装。

## 使用

1. 首次启动 App 时，地区选**中国香港**（选中国大陆的话，要先登录 Creators' Cloud 账号才能连接相机）。
2. 相机设为 *电脑遥控：开*，连接方式 *USB*，接上数据线。
3. 允许 Creators' App 访问 USB 设备，**相机**页出现 ZV-E10 和 **Remote Shooting** 按钮。
4. 拍照时**按住**快门键，对上焦再松开。
5. 相机连着时，别的 App（例如相册）申请访问 ZV-E10，点**取消**。

**反馈问题**：安装调试版，复现时运行 `adb logcat -v time > zvpatch.log`，附上 `grep -a ZVPATCH zvpatch.log` 的结果。

## 免责声明

Sony、ZV-E10、Creators' App 是索尼集团公司的商标，仅用于说明兼容性。补丁只修改机型检查、USB 会话处理和一个按钮，不涉及账号、登录、订阅或网络服务。按原样提供，不作任何担保。

## 许可证

[MIT](LICENSE)
