# 4. 现状与限制

测试环境：App 3.5.0（地区选中国香港），ZV-E10 固件 2.03，Android 16（一加，ColorOS）。

## 可用

- USB 连接，不需要蓝牙配对
- 实时画面
- 拍照（按住快门），照片自动传到手机 `DCIM/CA_IMAGES/`
- 录像开始/停止
- 曝光模式、曝光补偿、ISO
- 退出远程拍摄、切换标签页后再进入

## 不可用

| 功能 | 原因 |
|---|---|
| Wi-Fi 连接 | 实测不支持，原因未分析 |
| 导入 | 电脑遥控模式下相机不开放存储卡，见 [P6](02-patches.md#p6隐藏-import) |
| 轻点快门拍照 | App 在约 65 毫秒内发完 S1、S2 的按下和松开，相机都回复成功但不拍照；按住快门可以拍。原因没有单独验证 |
| USB 连接时读取维修日志 | 被 P4 跳过 |

未研究：蓝牙、云端功能、相机设置备份、固件升级。

## 注意

- **别的 App 申请访问相机时点取消**，否则实时画面停止，要重新插拔数据线。
- **首次启动时地区选中国香港。** 选中国大陆的话，连接相机前需要登录 Creators' Cloud 账号（`CNApi.isNeedSignInForCameraConnect()`）。
- 显示刚拍的照片时，App 高频查询 `GetObjectInfo(0xFFFFC001)`（实测 10 秒约 1100 次），没有发现功能影响。

## 其它相机和 App 版本

- **新 App 版本**：按类名重新定位 `UsbSupportedCameraManager`、`UsbInsertedState`、`FunctionModeController.onResume`、`PtpUsbClient.getDeviceLog`、`PtpUsbCamera.initialize` / `switchFunctionMode`、`UsbCameraConnectionController`。补丁工具默认拒绝其它 base APK（`ZVPATCH_SKIP_HASH=1` 可跳过），锚点对不上时会中止。
- **其它相机**：从 P1 开始，用调试版对比 DeviceInfo 命令列表和追踪日志，找第一条 `PTP FAIL`。
- **电脑遥控模式下支持内容传输的相机**：P5、P6 会让它失去导入功能。
