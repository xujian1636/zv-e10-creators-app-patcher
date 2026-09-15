# 4. Status and limitations

Tested with app 3.5.0 (region: Hong Kong), ZV-E10 firmware 2.03, Android 16 (OnePlus, ColorOS).

## Works

- USB connection, no Bluetooth pairing needed
- Live view
- Photo capture (hold the shutter button); photos are copied to `DCIM/CA_IMAGES/` on the phone
- Movie recording start/stop
- Exposure mode, exposure compensation, ISO
- Leaving Remote Shooting or switching tabs and coming back

## Not available

| Feature | Reason |
|---|---|
| Wi-Fi connection | Not supported (tested); cause not analysed |
| Import | The camera does not expose its card in PC Remote mode, see [P6](02-patches.md#p6--hide-import) |
| Tapping the shutter button | The app sends S1/S2 down and up within ~65 ms; the camera acknowledges all four but takes no photo. Holding the button works. The cause was not isolated |
| Reading the service log over USB | Skipped by P4 |

Not investigated: Bluetooth, cloud features, camera settings backup, firmware update.

## Notes

- **Tap Cancel when another app asks for access to the camera**; otherwise live view stops until the cable is
  reconnected.
- **Choose Hong Kong as the region on first launch.** With mainland China the app requires a Creators' Cloud
  sign-in before connecting a camera (`CNApi.isNeedSignInForCameraConnect()`).
- While showing the photo just taken, the app polls `GetObjectInfo(0xFFFFC001)` heavily (~1100 times in 10 s);
  no functional impact was seen.

## Other cameras and app versions

- **New app version:** relocate the patches by class name: `UsbSupportedCameraManager`, `UsbInsertedState`,
  `FunctionModeController.onResume`, `PtpUsbClient.getDeviceLog`, `PtpUsbCamera.initialize` / `switchFunctionMode`,
  `UsbCameraConnectionController`. The patcher refuses other base APKs unless `ZVPATCH_SKIP_HASH=1`, and aborts when
  an anchor does not match.
- **Other cameras:** start from P1 and use the debug build to compare the DeviceInfo operation list with the trace;
  find the first `PTP FAIL`.
- **Cameras that support contents transfer in PC Remote mode** would lose Import with P5 and P6.
