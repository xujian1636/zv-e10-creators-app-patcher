# 1. App 如何与相机通信

Creators' App 的类名基本没有混淆，下文省略包名前缀 `jp.co.sony.ips.portalapp`。

## 协议

USB 和 Wi-Fi 下，App 说的是同一种语言：**PTP** 加索尼的厂商扩展（App 里以 `SDIO_*` 命名）。区别只在传输方式和发现方式。

| 命令 | 代码 | 用途 |
|---|---|---|
| `SDIO_Connect` | `0x9201` | 三阶段遥控握手 |
| `SDIO_GetExtDeviceInfo` | `0x9202` | 协议版本（ZV-E10 为 300），以及支持的属性和控制列表 |
| `SDIO_ControlDevice` | `0x9207` | 模拟按键：S1 `0xD2C1`、S2 `0xD2C2`、录制 `0xD2C8` |
| `SDIO_GetAllExtDevicePropInfo` | `0x9209` | 读取属性值 |
| `SDIO_OpenSession` | `0x9210` | 按功能模式打开会话（参数 `1, 模式`） |
| `SDIO_SetContentsTransferMode` | `0x9212` | 开关内容传输，并指定由谁挑选影像 |
| `SDIO_GetDeviceLog` | `0x9213` | 读取维修日志，**ZV-E10 未实现** |
| `SDIO_GetPartialLargeObject` | `0x9219` | 分块下载文件 |
| `GetStorageIDs` / `GetObjectHandles` / `GetObjectPropList` | `0x1004` / `0x1007` / `0x9805` | 列出存储、文件、文件信息 |
| `GetObject` / `GetObjectInfo` / `GetPartialObject` | `0x1009` / `0x1008` / `0x101B` | USB 下句柄 `0xFFFFC002` 是实时画面帧，`0xFFFFC001` 是刚拍的照片 |

## 传输方式

**USB**：相机只暴露一个 PTP 接口（USB 设备类 6，厂商 ID `0x054C`）。实时画面靠反复发送 `GetObject(0xFFFFC002)`。

**Wi-Fi**：App 先用 SSDP 在网络里搜索，下载相机的 `dd.xml` 及它指向的能力文件（`DigitalImagingDesc.xml`，下称 DID），然后把 PTP 指令跑在 TCP 上。实时画面**不走 PTP**：`dd.xml` 里带一个 HTTP 地址（`http://<相机>:60152/liveviewstream?…`），由单独的线程拉取。

## 相机的两套 Wi-Fi 服务

两种模式下相机发布的 `dd.xml` 结构相同，差别在 DID，而这一个标志决定了 App 能做什么：

| | 电脑遥控功能 | 智能手机连接 / 发送到智能手机 |
|---|---|---|
| 连接方式 | Wi-Fi Direct，或相机接入的路由器（接入点） | 仅 Wi-Fi Direct，SSID 与电脑遥控的不同 |
| DID 协议版本 | 3.00 | 3.00 |
| DID 媒体服务 | **关闭** | **开启** |
| 打开会话的方式 | 普通 `OpenSession`，无法指定模式 | `SDIO_OpenSession` 带模式参数，和 USB 一样 |
| `GetStorageIDs` | `StoreNotAvailable` | 返回存储列表，见下 |

这个分支在 `ptpip/initialization/Initializer.createOpenSessionState()`：没有媒体服务标志时，App 根本无法请求传输模式，所以电脑遥控的会话只能用于遥控。

## 会话模式与存储

`EnumFunctionMode`：`REMOTE_CONTROL_MODE`（0）用于拍摄，`CONTENTS_TRANSFER_MODE`（1）用于传输。切换模式就是关闭会话再重开（`PtpUsbCamera` / `PtpIpCamera.switchFunctionMode()`）。

智能手机连接下，传输会话会暴露两种存储之一：

- `VIRTUAL_MEDIA_1`（`0x00F10001`）：你在相机上选中的影像。`ContentsPushController` 发现这个存储出现后自动下载（`SDIO_SetContentsTransferMode(1, 1, 0)` → `GetObjectHandles` → `SDIO_GetPartialLargeObject`）。
- `STORAGE_MEDIA_1`（`0x00010001`）：存储卡本身。导入界面发送 `SDIO_SetContentsTransferMode(2, 1, 0)`（含义是"由手机挑选"）之后才会出现，相机随即显示*在智能手机上操作…*，App 用 `GetObjectHandles` 和 `GetObjectPropList` 列出整张卡。

电脑遥控模式下两者都没有：`GetStorageIDs` 在 USB 和 Wi-Fi 下都回复 `StoreNotAvailable`。

## 连接流程

```
USB    usb/UsbPermissionRequester → UsbPermissionReceiver → UsbSupportedCameraManager（机型检查）
       usb/statemachine/UsbInsertedState → CameraConnector.connectPtpUsb(...)
Wi-Fi  common/device/SsdpUtil（M-SEARCH、dd.xml）→ DeviceDescription.getControlModel()（DID 门槛）
       toppage/devicetab/wificonnect/SearchedDeviceListController → CameraManagerUtil.addCamera(...)
共同   camera/Ptp{Usb,Ip}Camera → ptpip/Ptp{Usb,Ip}Client → ptpip/Ptp{Usb,Ip}Manager
       ptpip/initialization/Initializer  GetDeviceInfo →（SDIO_）OpenSession → SDIO_Connect(1) → (2)
                                         → SDIO_GetExtDeviceInfo(300) → SDIO_Connect(3)
```

- USB 的支持机型列表来自 `assets/camera_guide.json`，缓存超过 24 小时后换成服务器下载的列表。
- 会话打开后，相机对象会索取维修日志（`getDeviceLog()`）。
- `DeviceInfo.isUxpSupported()`（DID 版本 ≥ 3.01）是传输功能的总开关，`ContentsPushController` 和导入界面都受它控制。

## 相机页

- `toppage/devicetab/controller/UsbCameraConnectionController`：USB 卡片上的 **Remote Shooting** 和 **Import** 按钮；Wi-Fi 卡片在 DID 声明没有媒体服务时会把 Import 置灰。
- `toppage/devicetab/controller/FunctionModeController.onResume()`：相机页每次显示，都会把不在传输模式的会话改成 `CONTENTS_TRANSFER_PUSH`，好让相机能发送影像。
- `toppage/{Home,Library,Community}TabFragment.onResume()`：这几个页面会断开相机，回到相机页时重新连接。

## 远程拍摄

- **拍照**：手指按下发送 S1 按下；抬起时依次发送 S2 按下、S2 松开、S1 松开。随后用 `GetObjectInfo` 和 `GetPartialObject` 读取 `0xFFFFC001`，存到手机 `DCIM/CA_IMAGES/`。
- **录像**：`SDIO_ControlDevice(0xD2C8)`，文件留在存储卡上。
- **实时画面**：由 `BaseCamera.startLiveView()` 启动，触发时机是相机上报实时画面状态发生变化，或走已注册相机的连接流程。
