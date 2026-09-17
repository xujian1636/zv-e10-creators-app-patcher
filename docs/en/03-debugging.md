# 3. Debugging workflow

How the problems were found, in a form that can be reused for other cameras or app versions (macOS).

## Tools

```bash
brew install openjdk apktool jadx
brew install --cask android-commandlinetools android-platform-tools
sdkmanager "build-tools;35.0.0"
export JAVA_HOME=/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home   # apksigner needs a real JDK
```

- Read code with **jadx**; patch the **smali** from `apktool d -r`. Class names are mostly intact, so searching for
  `UsbSupportedCameraManager`, `switchFunctionMode` and the like works.
- Use **`-r`** (do not decode resources) so resources stay byte-identical. Make UI changes in code
  (e.g. `setVisibility`).
- On case-insensitive APFS apktool renames colliding classes. Identify files by their `.class` line, not by path.

## Get evidence first

1. **The release build has no logs.** The `common/log/AdbLog` methods print nothing and there is no switch, so the
   debug build injects tracing ([D1–D3](02-patches.md#debug-build-only)).
2. **Pick the right adb transport.** For USB tests the phone's USB port is taken by the camera, so use wireless adb
   (Developer options → Wireless debugging → Pair with pairing code). For Wi-Fi tests do the opposite: keep the adb
   cable plugged in, so the log keeps flowing while the phone joins the camera's hotspot and leaves your network.
3. **Start logging before connecting the camera**, take the whole buffer, and search with `grep -a` (logcat contains
   stray bytes that make `grep` treat the file as binary):
   ```bash
   adb logcat -c && adb logcat -v time > run.log
   grep -a ZVPATCH run.log
   ```
   Do not filter with `adb logcat -s ZVPATCH`: the filtered stream drops lines under load and makes a working
   sequence look truncated.
4. **ColorOS/OxygenOS drop logs** above 300 lines per second per app (`LOGS OVER PROC QUOTA(300) ... DROPPED`); a
   missing line does not prove something did not happen.
5. **Without the debug build** you can still tell what happened: `adb shell dumpsys usagestats` lists which screen
   was in the foreground and when, `ls -lt /sdcard/DCIM/CA_IMAGES` shows what arrived, and the system log records
   Wi-Fi joins (`connectToNetwork "DIRECT-…"`), USB attach/detach and crashes.

## Useful log lines

| Pattern | Meaning |
|---|---|
| `INIT OK mode=` / `INIT FAILED code=` | Session initialisation result |
| `PTP FAIL op=` | Find the first one; the `GeneralError`s that follow are its consequence |
| `SWITCH function mode` / `TERMINATE` / `DISCONNECT` | Session closed; the stack trace follows |
| `SessionAlreadyOpen` | The camera still holds the previous session; the app closes it and retries (harmless) |
| `UsbPermissionActivity` | USB permission dialog shown (possibly for another app) |
| `DID name=… mediaServer=` | Which Wi-Fi service answered: PC Remote (disabled) or Smartphone Connect (enabled) |
| `STORAGE ids=` | `[]` nothing to transfer, `[VIRTUAL_MEDIA_1]` images selected on the camera, `[STORAGE_MEDIA_1]` the card |
| `LV setLiveViewStreamCallback` / `LV frames=` | Live view was started, and frames are arriving |
| `FATAL EXCEPTION` | Crash |

## Wi-Fi specifics

- The camera must not be in PC Remote mode when you test transfers: it refuses *Send to Smartphone* while PC Remote
  is on, with *"The operation or setting cannot be performed"* on its screen.
- The two Wi-Fi services use different hotspots (`DIRECT-…` names differ), and each has its own save-destination
  settings, so a remote shot can arrive as JPEG only in one mode and RAW+JPEG in the other.
- A session in remote-control mode cannot receive images. Leaving Remote Shooting reopens it for transfer, which
  takes about three seconds: close, reopen, re-read storage and object properties.

## Other apps taking the USB device

While the camera is connected, other apps may ask for access to it (the ColorOS Gallery does when opened). If allowed,
live view stops until the cable is reconnected; Cancel leaves the session intact. Rule this out before suspecting
the patches.

## Method

- Compare the camera's DeviceInfo operation list with every command the app sends, including those sent after the
  session opens.
- A spinner or "Could not connect" is a symptom; find the first failing command or the stack trace that closed the
  session.
- Watch what screens do on resume and tab switches; session modes are often changed there.
- Change one thing per build. Before installing, check the change with `dexdump -d` and that all APKs share one
  certificate with `apksigner verify --print-certs`.
- Do not patch error handling to tolerate failures you have not identified.
