# 2. 问题与补丁

所有补丁都针对用 `apktool d -r` 解码的 App 3.5.0，实现见 [`patcher/zvpatch.py`](../../patcher/zvpatch.py)。每个补丁按类描述符定位目标；锚点匹配不是恰好一次时直接中止。日志来自调试版（[D1、D2](#仅调试版)），做了缩短。

## P1：不在支持机型列表里

- **原因**：`usb/UsbSupportedCameraManager` 在支持机型列表里找不到 `ZV-E10`，结果为 `NOT_SUPPORTED_CAMERA_FAIL`。安装包自带的列表和服务器下发的列表里有 ZV-E10 II（`ZV-E10M2`），没有初代。
- **改动**：`…$set$deviceInfoUpdaterListener$1.onDeviceInfoChanged()` 复制列表后，往副本里加一项 `modelName = "ZV-E10"`。服务器更新列表不影响补丁。

## P2：插上相机后一直转圈

- **原因**：`usb/statemachine/UsbInsertedState.onEnter()` 用 `CONTENTS_TRANSFER_PULL` 自动连接，相机拒绝读存储卡，会话悄悄结束：
  ```
  PTP req  op=SDIO_OpenSession params=[1, 1]
  PTP FAIL op=GetStorageIDs code=StoreNotAvailable
  PTP FAIL op=GetObjectPropsSupported code=OperationNotSupported
  ```
- **改动**：`UsbInsertedState` 和 `UsbInsertedState$onEnter$1$1` 里的三处改为 `REMOTE_CONTROL`。

## P3："Could not connect to your camera via USB."

- **原因**：相机页显示时，`FunctionModeController.onResume()` 把刚打开的远程拍摄会话切到传输内容模式，也就是关掉后用相机拒绝的模式重开：
  ```
  SWITCH function mode to=CONTENTS_TRANSFER_MODE
      at camera.CameraConnector.switchFunctionMode
      at toppage.devicetab.controller.FunctionModeController.onResume
  ```
- **改动**：跳过这次切换（第一个分支改为 `goto`），`onResume()` 其余部分不变。

## P4："Could not perform."，实时画面转圈

- **原因**：会话打开后，`PtpUsbCamera` 调用 `ptpip/PtpUsbClient.getDeviceLog()` 读取维修日志。ZV-E10 的命令列表里没有 `SDIO_GetDeviceLog`，拒绝后会话上的每条命令都失败：
  ```
  PTP FAIL op=SDIO_GetDeviceLog code=InvalidParameter
  PTP FAIL op=SDIO_GetAllExtDevicePropInfo code=GeneralError
  PTP FAIL op=SDIO_ControlDevice code=GeneralError
  ```
- **改动**：`getDeviceLog()` 直接返回。USB 连接时不再读取维修日志。

## P5：去过首页后又连不上

- **原因**：首页断开相机后，回到相机页时新建的相机对象又用传输内容模式打开会话，初始化失败（`SDIO_OpenSession params=[1, 1]` → `INIT FAILED code=GeneralError`）。P2、P3 改的是调用方，覆盖不到这条路径。
- **改动**：在 `camera/PtpUsbCamera` 里统一处理：`initialize()` 一律用 `REMOTE_CONTROL_MODE`，`switchFunctionMode()` 遇到其它模式直接返回 `false`。P2、P3 仍保留，它们还让 App 记录的当前功能保持为远程拍摄。

## P6：隐藏 Import

- **原因**：电脑遥控模式下，ZV-E10 接受 `SDIO_SetContentsTransferMode(2, 1, 0)`，但 `GetStorageIDs` 仍回复 `StoreNotAvailable`，Contents Transfer Enable Status 也不变，导入只会提示 *"Failed to access storage on the camera."*。验证方法是让传输内容会话忽略读对象属性的失败，再进入导入界面。
- **改动**：`toppage/devicetab/controller/UsbCameraConnectionController` 绑定按钮时，把 Import（`0x7f0a0494`）设为 `View.GONE`，Remote Shooting 占满整行。

## 仅调试版

- **D1**：新增 `zv/ZvLog`，在 USB `TransactionExecutor` 的发送、成功、失败处，以及传输超时和初始化结果处记录日志（标签 `ZVPATCH`）。实时画面轮询只记录 `AccessDenied` 以外的失败。
- **D2**：在 `BasePtpManager.terminatePtpCommunication()`、`PtpUsbClient.switchFunctionMode()`、`PtpUsbCamera.disconnect()` 打印调用栈，用来找出是谁关闭了会话（P3、P5 就是这样找到的）。

## 签名

XAPK 里的所有 APK（base 和 split）必须用同一把密钥签名，且需要 v2/v3 签名（先 `zipalign -p 4`，再用 `apksigner`）。用你的密钥签名的包，不能覆盖安装在其它密钥签名的版本上。
