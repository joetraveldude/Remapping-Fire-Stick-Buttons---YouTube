# Remapping the YouTube Button to YouTube TV

> **Status:** Working proof of concept on the tested Fire TV Stick 4K. The physical YouTube shortcut successfully launches YouTube TV using a background shell process. **Automatic restart after reboot is not configured or verified.** Operation after `adb disconnect` has not yet been verified.

## Overview

This guide documents a free, computer-assisted method for making the dedicated **YouTube** button on an Amazon Fire TV remote launch **YouTube TV**.

It uses Android Debug Bridge (ADB) from Windows to start a shell process on the Fire TV. That process watches Android logs for the physical YouTube-button event and launches YouTube TV.

**Important:** This is a *log-triggered launcher*, not a true system-level button reassignment. Amazon may still process the original YouTube shortcut first. The solution is experimental and may stop working after a restart, update, or process termination.

## Tested setup

| Item | Tested value |
| --- | --- |
| Streaming device | Amazon Fire TV Stick 4K |
| Operating system | Fire OS 6.7.1.1 |
| Remote | Alexa Voice Remote with Prime Video, Netflix, Disney+, and YouTube shortcut buttons |
| Target button | Red YouTube button, bottom right |
| Physical scan code | `747` (hex `02eb`) |
| Android key name | `KEYCODE_APP_4` |
| Android keycode | `294` |
| YouTube TV package | `com.amazon.firetv.youtube.tv` |
| Computer | Windows, Command Prompt |
| ADB | Android SDK Platform Tools |
| Tested connection | ADB over local network, TCP port `5555` |

The example ADB executable was run from:

```text
C:\Users\tst-1691\Downloads\platform-tools-latest-windows\platform-tools
```

Your folder and Fire TV IP address may differ.

## Prerequisites

1. Install [Android SDK Platform Tools](https://developer.android.com/tools/releases/platform-tools) on Windows.
2. Enable ADB debugging on your Fire TV in its developer settings.
3. Ensure the Fire TV and Windows computer can communicate over your local network.
4. Install **YouTube TV** on the Fire TV and verify it launches normally.
5. Open **Command Prompt** in the folder containing `adb.exe`.

> **Security:** ADB debugging grants powerful device access. Use it only on a trusted network. Consider disabling network ADB debugging when you no longer need it, recognizing that this may affect the background process.

## FAQ

### 1. How do I verify the computer can see my Fire TV?

Run:

```cmd
adb devices
```

The tested device appeared as:

```text
192.168.4.91:5555    device
```

If yours is absent, reconnect using its current IP address:

```cmd
adb connect YOUR_FIRE_TV_IP:5555
adb devices
```

You may need to approve an ADB authorization prompt on the television.

### 2. How do I verify that ADB can launch YouTube TV?

Run:

```cmd
adb shell monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1
```

**Observed result:** YouTube TV launched successfully.

If the package name differs on another device, discover it with:

```cmd
adb shell pm list packages | findstr /i "youtube"
```

### 3. How did we identify the physical YouTube button?

The physical button produced these log details:

```text
keyCode=KEYCODE_APP_4
keyCode: 294
scanCode=747
action=ACTION_DOWN
deviceId=9
```

To reproduce the detection:

```cmd
adb shell logcat -c
```

Press the **physical YouTube button once**, then run:

```cmd
adb shell logcat -d -v brief | findstr /i "scanCode=747"
```

**Observed result:** Log lines from `OZ-DCS:KeyPressReceiver` and `OZ-DCS:CallbackBaseManager_KeyPress` confirmed `KEYCODE_APP_4` and `scanCode=747`.

### 4. Why didn't `adb shell input keyevent 747` work?

`747` is the physical **Linux scan code**, not the Android keycode expected by `input keyevent`.

The correct Android keycode is **294**:

```cmd
adb shell input keyevent 294
```

**Observed result:** This triggered the regular YouTube shortcut, not YouTube TV.

An earlier guess of `289` was incorrect. On the tested device, it navigated to the Fire TV interface rather than activating the YouTube shortcut.

### 5. What was the first successful one-press test?

This command cleared the log, waited for one physical YouTube-button press, and launched YouTube TV:

```cmd
adb shell "logcat -c; logcat -v brief | grep -m 1 'Remote button pressed.*keyCode: 294.*ACTION_DOWN.*scanCode=747' >/dev/null && monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1"
```

Press the red YouTube button after starting the command.

**Observed result:** YouTube TV launched. The command then exited.

### 6. How did we make it respond to repeated presses?

This foreground loop launched YouTube TV each time the physical YouTube button was pressed:

```cmd
adb shell "logcat -c; logcat -v brief | grep --line-buffered 'Remote button pressed.*keyCode: 294.*ACTION_DOWN.*scanCode=747' | while read line; do monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1; done"
```

**Observed result:** Repeated **Home → YouTube → Home → YouTube** tests worked.

Press **Ctrl+C** in Command Prompt to stop this foreground test.

### 7. How did we start the remapper in the background?

After stopping the foreground test, the following command was run from **Windows Command Prompt**:

```cmd
adb shell "nohup sh -c 'logcat -c; logcat -v brief | grep --line-buffered \"Remote button pressed.*keyCode: 294.*ACTION_DOWN.*scanCode=747\" | while read line; do monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1; done' >/dev/null 2>&1 </dev/null &"
```

This attempts to detach the log-monitoring loop from the interactive ADB shell.

**Observed result:** The command returned to the Windows prompt, and pressing **Home → YouTube** launched YouTube TV while the process was running.

**Not yet verified:** Whether it keeps running after `adb disconnect`, or after a Fire TV reboot.

> **Note:** This command uses nested Windows and Android shell quoting. Paste it as one line into **Command Prompt (`cmd.exe`)**, not PowerShell, unless you adapt the quoting.

### 8. Does this replace Amazon's original YouTube shortcut?

Not at the system level. The script reacts to an event **after it appears in the Android logs**. Amazon's normal handling may still run, so the original YouTube destination may flash or open before YouTube TV takes over.

The reported outcome on the tested device was that pressing the red YouTube button ultimately opened **YouTube TV**.

### 9. Does this work with the Windows computer turned off?

**Not fully verified yet.** The background command is designed to run on the Fire TV itself rather than relying on a continuous Windows loop. However, a successful background launch alone does not prove it survives an ADB disconnect.

To test:

```cmd
adb disconnect
```

Close Command Prompt and press **Home → YouTube**.

If YouTube TV still launches, the process is operating without an active ADB session. Reconnect later if more configuration is needed.

### 10. Will it survive a Fire TV reboot or power outage?

**No reboot persistence has been implemented or tested.**

A detached shell process is normally lost when the device reboots. An automatic-start mechanism would be required, and compatibility with Fire OS 6.7.1.1 still needs investigation.

Until that is solved, the background command may need to be run again after a restart.

### 11. How can I stop the background remapper?

The foreground version stops with **Ctrl+C**. For the detached version, first reconnect with ADB and inspect relevant processes:

```cmd
adb shell ps -A | findstr /i "logcat grep sh monkey"
```

Because those process names may also belong to unrelated services, **do not blindly kill every matching process**. Identify the remapper's specific process ID before stopping it. A Fire TV reboot will also terminate this temporary background process.

### 12. What if the command shows `Invalid filter expression`?

An earlier attempt used an invalid `logcat` filter:

```text
OZ-DCS:KeyPressReceiver:I
```

The colon inside the log tag interfered with the filter syntax. Instead, use a general log dump and search for the distinctive event:

```cmd
adb shell logcat -d -v brief | findstr /i "KEYCODE_APP_4"
```

Or search for the physical scan code:

```cmd
adb shell logcat -d -v brief | findstr /i "scanCode=747"
```

### 13. Why distinguish physical button presses from simulated ADB presses?

The logs showed:

- **Physical remote:** `scanCode=747`, `deviceId=9`.
- **ADB-injected event:** `scanCode=0`, `deviceId=-1`.

The script matches the physical remote's scan code so a simulated keypress does not trigger it accidentally.

### 14. Do I need GitHub, a paid remapping app, or root access?

No GitHub account, paid app, or root access was used for the **tested proof of concept**. The successful commands relied on ADB and utilities already available in the Fire TV shell.

This does **not** mean the solution is a permanent firmware-level remap.

## Quick reference

**Verify connection**

```cmd
adb devices
```

**Launch YouTube TV directly**

```cmd
adb shell monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1
```

**Inspect a physical YouTube press**

```cmd
adb shell logcat -c
```

Press the red YouTube button, then:

```cmd
adb shell logcat -d -v brief | findstr /i "scanCode=747"
```

**Start the tested background loop**

```cmd
adb shell "nohup sh -c 'logcat -c; logcat -v brief | grep --line-buffered \"Remote button pressed.*keyCode: 294.*ACTION_DOWN.*scanCode=747\" | while read line; do monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1; done' >/dev/null 2>&1 </dev/null &"
```

## Current status and next steps

- [x] ADB connected from Windows to Fire TV.
- [x] YouTube TV package identified and launched through ADB.
- [x] Physical YouTube-button event identified (`KEYCODE_APP_4`, Android `294`, scan code `747`).
- [x] One-time on-device event listener launched YouTube TV.
- [x] Continuous on-device event listener handled repeated presses.
- [x] Detached background command returned to the Windows prompt and worked in subsequent button tests.
- [ ] Verify operation after `adb disconnect`.
- [ ] Verify whether Amazon's original shortcut appears briefly.
- [ ] Investigate safe automatic startup after Fire TV reboot.
- [ ] Retest after Fire OS updates.

---

*Documented from hands-on testing on October 11, 2026. This is an experimental, unofficial configuration. Amazon can change shortcut handling in future Fire OS updates.*
