# 1. How the app talks to the camera

Creators' App keeps most class names unobfuscated. The package prefix `jp.co.sony.ips.portalapp` is omitted below.

## Protocol

In PC Remote mode the camera exposes a single **PTP** interface over USB (device class 6). Commands are PTP plus
Sony's vendor extensions (named `SDIO_*` in the app).

| Operation | Code | Purpose |
|---|---|---|
| `SDIO_Connect` | `0x9201` | Three-phase remote handshake |
| `SDIO_GetExtDeviceInfo` | `0x9202` | Protocol version (300 on the ZV-E10), supported properties and controls |
| `SDIO_ControlDevice` | `0x9207` | Button presses: S1 `0xD2C1`, S2 `0xD2C2`, record `0xD2C8` |
| `SDIO_GetAllExtDevicePropInfo` | `0x9209` | Read property values |
| `SDIO_OpenSession` | `0x9210` | Open a session in a function mode (params `1, mode`) |
| `SDIO_SetContentsTransferMode` | `0x9212` | Turn contents transfer on or off |
| `SDIO_GetDeviceLog` | `0x9213` | Read the service log — **not implemented by the ZV-E10** |
| `GetStorageIDs` | `0x1004` | List memory cards |
| `GetObjectPropsSupported` | `0x9801` | MTP object properties |
| `GetObject` / `GetObjectInfo` / `GetPartialObject` | `0x1009` / `0x1008` / `0x101B` | Handle `0xFFFFC002` is the live-view frame, `0xFFFFC001` the photo just taken |

In PC Remote mode the ZV-E10 (firmware 2.03) answers `GetStorageIDs` with `StoreNotAvailable` and
`GetObjectPropsSupported` with `OperationNotSupported`.

## Session modes

The mode parameter of `SDIO_OpenSession` comes from `EnumFunctionMode`: `REMOTE_CONTROL_MODE` (0) for remote
shooting, `CONTENTS_TRANSFER_MODE` (1) for import. Over USB, switching mode means closing the session and reopening it
in the new mode (`PtpUsbCamera.switchFunctionMode()`).

## USB connection flow

```
usb/UsbPermissionRequester         request USB permission (vendor ID 0x054C)
usb/UsbPermissionReceiver          permission granted (mainland China region: sign-in check)
usb/UsbSupportedCameraManager      read DeviceInfo, check the model against the supported camera list
usb/statemachine/UsbInsertedState  CameraConnector.connectPtpUsb(CONTENTS_TRANSFER_PULL)
camera/PtpUsbCamera → ptpip/PtpUsbClient → ptpip/PtpUsbManager
ptpip/initialization/Initializer   GetDeviceInfo → SDIO_OpenSession → SDIO_Connect(1) → (2) → SDIO_GetExtDeviceInfo(300) → SDIO_Connect(3)
```

- The supported camera list comes from `assets/camera_guide.json` and is replaced by a list downloaded from Sony's
  server once the cache is older than 24 hours.
- In contents transfer mode, initialisation also reads storage and object properties; an object-property failure
  ends the session without any message.
- After the session opens, `PtpUsbCamera` sends `SDIO_GetDeviceLog` to store the camera's service log.

## Cameras tab

- `toppage/devicetab/controller/UsbCameraConnectionController`: the **Remote Shooting** and **Import** buttons on the
  USB camera card.
- `toppage/devicetab/controller/FunctionModeController.onResume()`: every time the tab is shown, switches any session
  not in contents transfer mode to `CONTENTS_TRANSFER_PUSH`.
- `toppage/HomeTabFragment.onResume()`: the Home tab disconnects the camera; the Cameras tab reconnects it.

## Remote shooting and import

- **Live view:** repeated `GetObject(0xFFFFC002)`.
- **Photo:** finger down sends S1 down; finger up sends S2 down, S2 up, S1 up. The app then reads `0xFFFFC001` with
  `GetObjectInfo` and `GetPartialObject` and saves it to `DCIM/CA_IMAGES/` on the phone.
- **Movie:** `SDIO_ControlDevice(0xD2C8)`.
- **Import:** needs a successfully initialised contents transfer session. The import screen sends
  `SDIO_SetContentsTransferMode(2, 1, 0)` and waits up to 5 s for Contents Transfer Enable Status (`0xD295`) to
  become enabled.
