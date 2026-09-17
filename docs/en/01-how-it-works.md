# 1. How the app talks to the camera

Creators' App keeps most class names unobfuscated. The package prefix `jp.co.sony.ips.portalapp` is omitted below.

## Protocol

Over USB and over Wi-Fi the app speaks the same language: **PTP** plus Sony's vendor extensions (named `SDIO_*` in
the app). Only the transport and the discovery differ.

| Operation | Code | Purpose |
|---|---|---|
| `SDIO_Connect` | `0x9201` | Three-phase remote handshake |
| `SDIO_GetExtDeviceInfo` | `0x9202` | Protocol version (300 on the ZV-E10), supported properties and controls |
| `SDIO_ControlDevice` | `0x9207` | Button presses: S1 `0xD2C1`, S2 `0xD2C2`, record `0xD2C8` |
| `SDIO_GetAllExtDevicePropInfo` | `0x9209` | Read property values |
| `SDIO_OpenSession` | `0x9210` | Open a session in a function mode (params `1, mode`) |
| `SDIO_SetContentsTransferMode` | `0x9212` | Turn contents transfer on or off, and say who selects the images |
| `SDIO_GetDeviceLog` | `0x9213` | Read the service log — **not implemented by the ZV-E10** |
| `SDIO_GetPartialLargeObject` | `0x9219` | Download a file in chunks |
| `GetStorageIDs` / `GetObjectHandles` / `GetObjectPropList` | `0x1004` / `0x1007` / `0x9805` | List storages, files, file metadata |
| `GetObject` / `GetObjectInfo` / `GetPartialObject` | `0x1009` / `0x1008` / `0x101B` | Handle `0xFFFFC002` is the live-view frame over USB, `0xFFFFC001` the photo just taken |

## Transports

**USB.** The camera exposes a single PTP interface (device class 6, vendor ID `0x054C`). Live view is a repeated
`GetObject(0xFFFFC002)`.

**Wi-Fi.** The app searches the network with SSDP, downloads the camera's `dd.xml` and the capability file it
points to (`DigitalImagingDesc.xml`, "DID" below), then speaks PTP over TCP. Live view is **not** PTP here: `dd.xml`
carries an HTTP stream URL (`http://<camera>:60152/liveviewstream?…`) that the app pulls in a separate thread.

## The camera's two Wi-Fi services

The ZV-E10 publishes the same `dd.xml` in both of its Wi-Fi modes; the DID differs, and that single flag decides
what the app can do:

| | PC Remote Function | Smartphone Connect / Send to Smartphone |
|---|---|---|
| Connection method | Wi-Fi Direct, or the router the camera joined (access point) | Wi-Fi Direct only, with a different SSID than PC Remote |
| DID `X_ServerVersion` | 3.00 | 3.00 |
| DID media server | **disabled** | **enabled** |
| Session open | plain `OpenSession` — the app cannot request a mode | `SDIO_OpenSession` with a mode, like USB |
| `GetStorageIDs` | `StoreNotAvailable` | storage list, see below |

`ptpip/initialization/Initializer.createOpenSessionState()` makes that choice: without the media-server flag it
cannot ask for contents transfer at all, so a PC Remote session is remote control only.

## Session modes and storages

`EnumFunctionMode`: `REMOTE_CONTROL_MODE` (0) for shooting, `CONTENTS_TRANSFER_MODE` (1) for transfer. Switching
mode closes the session and reopens it (`PtpUsbCamera` / `PtpIpCamera.switchFunctionMode()`).

Under Smartphone Connect a contents-transfer session exposes one of two storages:

- `VIRTUAL_MEDIA_1` (`0x00F10001`) — the images you selected on the camera. `ContentsPushController` sees the
  storage appear and copies them to the phone by itself (`SDIO_SetContentsTransferMode(1, 1, 0)` →
  `GetObjectHandles` → `SDIO_GetPartialLargeObject`).
- `STORAGE_MEDIA_1` (`0x00010001`) — the memory card, exposed after the Import screen sends
  `SDIO_SetContentsTransferMode(2, 1, 0)`, meaning "the phone selects". The camera then shows *Operating from the
  smartphone…* and the app lists the card with `GetObjectHandles` and `GetObjectPropList`.

In PC Remote mode neither appears: `GetStorageIDs` answers `StoreNotAvailable` over USB and over Wi-Fi.

## Connection flow

```
USB   usb/UsbPermissionRequester → UsbPermissionReceiver → UsbSupportedCameraManager (model check)
      usb/statemachine/UsbInsertedState → CameraConnector.connectPtpUsb(...)
Wi-Fi common/device/SsdpUtil (M-SEARCH, dd.xml) → DeviceDescription.getControlModel() (DID gate)
      toppage/devicetab/wificonnect/SearchedDeviceListController → CameraManagerUtil.addCamera(...)
both  camera/Ptp{Usb,Ip}Camera → ptpip/Ptp{Usb,Ip}Client → ptpip/Ptp{Usb,Ip}Manager
      ptpip/initialization/Initializer  GetDeviceInfo → (SDIO_)OpenSession → SDIO_Connect(1) → (2)
                                        → SDIO_GetExtDeviceInfo(300) → SDIO_Connect(3)
```

- The USB supported-camera list comes from `assets/camera_guide.json` and is replaced by a list downloaded from
  Sony's server once the cache is older than 24 hours.
- After the session opens, the camera object asks for the service log (`getDeviceLog()`).
- `DeviceInfo.isUxpSupported()` (DID version ≥ 3.01) guards the transfer paths: `ContentsPushController` and the
  Import screen.

## Cameras tab

- `toppage/devicetab/controller/UsbCameraConnectionController` — the **Remote Shooting** and **Import** buttons on
  the USB card; the Wi-Fi card greys Import out when the DID reports no media server.
- `toppage/devicetab/controller/FunctionModeController.onResume()` — every time the tab appears, a session that is
  not in contents-transfer mode is reopened as `CONTENTS_TRANSFER_PUSH`, so the camera can send images.
- `toppage/{Home,Library,Community}TabFragment.onResume()` — these tabs disconnect the camera; the Cameras tab
  reconnects it.

## Remote shooting

- **Photo:** finger down sends S1 down; finger up sends S2 down, S2 up, S1 up. The app then reads `0xFFFFC001` with
  `GetObjectInfo` and `GetPartialObject` and saves it to `DCIM/CA_IMAGES/` on the phone.
- **Movie:** `SDIO_ControlDevice(0xD2C8)`. The file stays on the card.
- **Live view:** started from `BaseCamera.startLiveView()`, which the app calls when the camera reports a change of
  its live-view status, or from the connection paths used for registered cameras.
