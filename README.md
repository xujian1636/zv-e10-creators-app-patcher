# ZV-E10 × Creators' App

Patches for Sony's **Creators' App** for Android (3.5.0) so it can remote-control a **ZV-E10 over USB**: live view,
photo capture with automatic transfer to the phone, movie recording, exposure controls. Includes a write-up of why
the app cannot use the camera and what was changed.

[中文说明](README.zh-CN.md)

> Unofficial, not affiliated with Sony. No Sony code or binaries are included: you supply the original app and
> the patcher modifies it locally, at your own risk.

## Compatibility

Creators' App's supported camera list includes the ZV-E10 II but not the original ZV-E10; every feature below was
adapted by this project. Tested with the patched app:

| Feature | Status |
|---|---|
| USB connection | ✅ |
| Wi-Fi connection | ❌ |
| Live view | ✅ |
| Photo capture | ✅ hold the shutter button; photos are copied to the phone |
| Movie recording | ✅ |
| Exposure mode, exposure compensation, ISO | ✅ |
| Import from the memory card | ❌ the camera does not expose its card; the button is hidden ([why](docs/en/04-status.md)) |

Tested: app 3.5.0, ZV-E10 firmware 2.03, Android 16. Camera setting: *PC Remote: On*, connection method *USB*.

## Why the unpatched app fails

1. The app checks the model against its **supported camera list** and rejects the ZV-E10.
2. On plug-in the app opens a **contents transfer** session, which the ZV-E10 does not support in PC Remote mode →
   endless spinner.
3. Whenever the Cameras tab is shown it **switches the session to contents transfer** → Remote Shooting says
   *"Could not connect to your camera via USB."*
4. After the session opens the app sends **`SDIO_GetDeviceLog`**, which the ZV-E10 does not implement; the error
   breaks the whole session → spinning live view, *"Could not perform."*
5. After a visit to the **Home** tab the app reconnects in contents transfer mode again.

Patches P1–P5 fix these in order; P6 hides the Import button, which cannot work. Details: [docs](#documentation).

## Documentation

1. [How the app talks to the camera](docs/en/01-how-it-works.md)
2. [Problems and patches](docs/en/02-patches.md)
3. [Debugging workflow](docs/en/03-debugging.md) — injected PTP tracing, wireless adb, other apps taking the USB device
4. [Status and limitations](docs/en/04-status.md)

## Patching (macOS)

```bash
brew install openjdk apktool
brew install --cask android-commandlinetools
sdkmanager "build-tools;35.0.0"

./patcher/patch.sh "Creators' App_3.5.0.xapk" out/
```

The input XAPK must contain a base APK with SHA-256
`b0716bf5b02b7c723d54d670d97c1c6c6daf0519a5ded5330fc4d9f3ff443f09`. Output:

- `CreatorsApp_3.5.0_ZV-E10_release.xapk` — the patched app
- `CreatorsApp_3.5.0_ZV-E10_debug.xapk` — same, with PTP tracing for bug reports
- `zvpatch-signing.jks` / `.pass` — your signing key; keep it to install future builds as updates

Uninstall the original app first (different signature), then install the XAPK with an XAPK installer.

## Use

1. On first launch, choose **Hong Kong** as the region (with mainland China the app requires a Creators' Cloud
   sign-in before it connects to a camera).
2. Set the camera to *PC Remote: On*, connection method *USB*, and connect the cable.
3. Allow USB access for Creators' App. The **Cameras** tab shows the ZV-E10 with a **Remote Shooting** button.
4. To take a photo, **hold** the shutter button until the camera has focused, then release.
5. While the camera is connected, tap **Cancel** if another app (for example the Gallery) asks for access to it.

**Bug reports:** use the debug build, run `adb logcat -v time > zvpatch.log` while reproducing, and attach the
output of `grep -a ZVPATCH zvpatch.log`.

## Disclaimer

Sony, ZV-E10 and Creators' App are trademarks of Sony Group Corporation, used only to describe compatibility.
The patches change the camera model check, USB session handling and one button only; they do not touch accounts,
sign-in, subscriptions or network services. Provided as-is, without warranty.

## License

[MIT](LICENSE)
