# 1. App 如何与相机通信

Creators' App 的类名基本没有混淆，下文省略包名前缀 `jp.co.sony.ips.portalapp`。

## 协议

电脑遥控模式下，相机通过 USB 只暴露一个 **PTP** 接口（USB 设备类 6），命令是 PTP 加索尼的厂商扩展（App 里以 `SDIO_*` 命名）。

| 命令 | 代码 | 用途 |
|---|---|---|
| `SDIO_Connect` | `0x9201` | 三阶段遥控握手 |
| `SDIO_GetExtDeviceInfo` | `0x9202` | 协议版本（ZV-E10 为 300），以及支持的属性和控制列表 |
| `SDIO_ControlDevice` | `0x9207` | 模拟按键：S1 `0xD2C1`、S2 `0xD2C2`、录制 `0xD2C8` |
| `SDIO_GetAllExtDevicePropInfo` | `0x9209` | 读取属性值 |
| `SDIO_OpenSession` | `0x9210` | 按功能模式打开会话（参数 `1, 模式`） |
| `SDIO_SetContentsTransferMode` | `0x9212` | 开启或关闭内容传输 |
| `SDIO_GetDeviceLog` | `0x9213` | 读取维修日志，**ZV-E10 未实现** |
| `GetStorageIDs` | `0x1004` | 列出存储卡 |
| `GetObjectPropsSupported` | `0x9801` | MTP 对象属性 |
| `GetObject` / `GetObjectInfo` / `GetPartialObject` | `0x1009` / `0x1008` / `0x101B` | 句柄 `0xFFFFC002` 是实时画面帧，`0xFFFFC001` 是刚拍的照片 |

电脑遥控模式下，ZV-E10（固件 2.03）对 `GetStorageIDs` 回复 `StoreNotAvailable`，对 `GetObjectPropsSupported` 回复 `OperationNotSupported`。

## 会话模式

`SDIO_OpenSession` 的模式参数来自 `EnumFunctionMode`：`REMOTE_CONTROL_MODE`（0）用于远程拍摄，`CONTENTS_TRANSFER_MODE`（1）用于导入。USB 上切换模式，就是关闭会话、换模式重新打开（`PtpUsbCamera.switchFunctionMode()`）。

## USB 连接流程

```
usb/UsbPermissionRequester         申请 USB 权限（厂商 ID 0x054C）
usb/UsbPermissionReceiver          获得权限（中国大陆地区：检查登录）
usb/UsbSupportedCameraManager      读取 DeviceInfo，按支持机型列表检查型号
usb/statemachine/UsbInsertedState  CameraConnector.connectPtpUsb(CONTENTS_TRANSFER_PULL)
camera/PtpUsbCamera → ptpip/PtpUsbClient → ptpip/PtpUsbManager
ptpip/initialization/Initializer   GetDeviceInfo → SDIO_OpenSession → SDIO_Connect(1) → (2) → SDIO_GetExtDeviceInfo(300) → SDIO_Connect(3)
```

- 支持机型列表来自 `assets/camera_guide.json`，缓存超过 24 小时后换成服务器下载的列表。
- 传输内容模式下，初始化后还会读存储卡和对象属性；读对象属性失败会结束会话，界面不提示。
- 会话打开后，`PtpUsbCamera` 会发送 `SDIO_GetDeviceLog` 读取维修日志，供售后使用。

## 相机页

- `toppage/devicetab/controller/UsbCameraConnectionController`：USB 相机卡片上的 **Remote Shooting** 和 **Import** 按钮。
- `toppage/devicetab/controller/FunctionModeController.onResume()`：相机页每次显示，都把不在传输内容模式的会话切到 `CONTENTS_TRANSFER_PUSH`。
- `toppage/HomeTabFragment.onResume()`：首页会断开相机，回到相机页时重新连接。

## 远程拍摄与导入

- **实时画面**：不断发送 `GetObject(0xFFFFC002)`。
- **拍照**：手指按下发送 S1 按下；抬起时依次发送 S2 按下、S2 松开、S1 松开。随后用 `GetObjectInfo` 和 `GetPartialObject` 读取 `0xFFFFC001`，存到手机 `DCIM/CA_IMAGES/`。
- **录像**：`SDIO_ControlDevice(0xD2C8)`。
- **导入**：需要初始化成功的传输内容会话。进入导入界面后发送 `SDIO_SetContentsTransferMode(2, 1, 0)`，最多等 5 秒，直到 Contents Transfer Enable Status（`0xD295`）变为已启用。
