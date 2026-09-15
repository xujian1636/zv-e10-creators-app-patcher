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
   debug build injects tracing ([D1, D2](02-patches.md#debug-build-only)).
2. **Use wireless adb** — the phone's USB port is taken by the camera. Developer options → Wireless debugging →
   Pair with pairing code.
3. **Start logging before plugging in the camera**, and search with `grep -a` (logcat contains stray bytes that make
   `grep` treat the file as binary):
   ```bash
   adb logcat -c && adb logcat -v time > run.log
   grep -a ZVPATCH run.log
   ```
4. **ColorOS/OxygenOS drop logs** above 300 lines per second per app (`LOGS OVER PROC QUOTA(300) ... DROPPED`); a
   missing line does not prove something did not happen.

## Useful log lines

| Pattern | Meaning |
|---|---|
| `INIT OK mode=` / `INIT FAILED code=` | Session initialisation result |
| `PTP FAIL op=` | Find the first one; the `GeneralError`s that follow are its consequence |
| `SWITCH function mode` / `TERMINATE` / `DISCONNECT` | Session closed; the stack trace follows |
| `SessionAlreadyOpen` | The camera still holds the previous session; the app closes it and retries (harmless) |
| `UsbPermissionActivity` | USB permission dialog shown (possibly for another app) |
| `FATAL EXCEPTION` | Crash |

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
