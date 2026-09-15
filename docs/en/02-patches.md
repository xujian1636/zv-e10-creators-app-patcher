# 2. Problems and patches

All patches target app 3.5.0 decoded with `apktool d -r`. The implementation is
[`patcher/zvpatch.py`](../../patcher/zvpatch.py); each patch finds its target by class descriptor and aborts if its
anchor does not match exactly once. Logs come from the debug build ([D1, D2](#debug-build-only)), shortened.

## P1 — Not in the supported camera list

- **Cause:** `usb/UsbSupportedCameraManager` does not find `ZV-E10` in the supported camera list →
  `NOT_SUPPORTED_CAMERA_FAIL`. Both the bundled and the server-provided list contain the ZV-E10 II (`ZV-E10M2`), not
  the original model.
- **Change:** `…$set$deviceInfoUpdaterListener$1.onDeviceInfoChanged()` adds an entry with `modelName = "ZV-E10"` to
  its copy of the list. Server updates to the list do not undo this.

## P2 — Endless spinner after plugging in

- **Cause:** `usb/statemachine/UsbInsertedState.onEnter()` auto-connects with `CONTENTS_TRANSFER_PULL`; the camera
  refuses the storage requests and the session ends silently:
  ```
  PTP req  op=SDIO_OpenSession params=[1, 1]
  PTP FAIL op=GetStorageIDs code=StoreNotAvailable
  PTP FAIL op=GetObjectPropsSupported code=OperationNotSupported
  ```
- **Change:** the three uses in `UsbInsertedState` and `UsbInsertedState$onEnter$1$1` become `REMOTE_CONTROL`.

## P3 — "Could not connect to your camera via USB."

- **Cause:** when the Cameras tab is shown, `FunctionModeController.onResume()` switches the freshly opened remote
  session to contents transfer, i.e. closes it and reopens it in the mode the camera refuses:
  ```
  SWITCH function mode to=CONTENTS_TRANSFER_MODE
      at camera.CameraConnector.switchFunctionMode
      at toppage.devicetab.controller.FunctionModeController.onResume
  ```
- **Change:** skip that switch (the first branch becomes a `goto`); the rest of `onResume()` is unchanged.

## P4 — "Could not perform.", spinning live view

- **Cause:** after the session opens, `PtpUsbCamera` calls `ptpip/PtpUsbClient.getDeviceLog()` to read the service
  log. The ZV-E10 does not list `SDIO_GetDeviceLog`; after rejecting it, every command on the session fails:
  ```
  PTP FAIL op=SDIO_GetDeviceLog code=InvalidParameter
  PTP FAIL op=SDIO_GetAllExtDevicePropInfo code=GeneralError
  PTP FAIL op=SDIO_ControlDevice code=GeneralError
  ```
- **Change:** `getDeviceLog()` returns immediately. The service log is no longer read over USB.

## P5 — Connection fails again after visiting Home

- **Cause:** the Home tab disconnects the camera; when the Cameras tab reconnects, the new camera object opens the
  session in contents transfer mode again and initialisation fails (`SDIO_OpenSession params=[1, 1]` →
  `INIT FAILED code=GeneralError`). P2 and P3 patch callers and do not cover this path.
- **Change:** handle it in `camera/PtpUsbCamera`: `initialize()` always uses `REMOTE_CONTROL_MODE`, and
  `switchFunctionMode()` returns `false` for any other mode. P2 and P3 stay; they also keep the app's current function
  set to remote control.

## P6 — Hide Import

- **Cause:** in PC Remote mode the ZV-E10 accepts `SDIO_SetContentsTransferMode(2, 1, 0)`, but `GetStorageIDs` still
  returns `StoreNotAvailable` and Contents Transfer Enable Status does not change, so Import can only end in
  *"Failed to access storage on the camera."* Verified by letting the contents transfer session ignore the
  object-property failure and opening the import screen.
- **Change:** when `toppage/devicetab/controller/UsbCameraConnectionController` binds the card, Import (`0x7f0a0494`)
  is set to `View.GONE`; Remote Shooting fills the row.

## Debug build only

- **D1:** adds `zv/ZvLog` and logs (tag `ZVPATCH`) at request, success and failure in the USB `TransactionExecutor`,
  plus transaction timeouts and the initialisation result. Live-view polling logs only failures other than
  `AccessDenied`.
- **D2:** prints a stack trace in `BasePtpManager.terminatePtpCommunication()`, `PtpUsbClient.switchFunctionMode()`
  and `PtpUsbCamera.disconnect()` to show who closed the session (this is how P3 and P5 were found).

## Signing

All APKs in the XAPK (base and splits) must be signed with the same key using v2/v3 signatures (`zipalign -p 4`
first, then `apksigner`). A build signed with your key cannot be installed over one signed with another key.
