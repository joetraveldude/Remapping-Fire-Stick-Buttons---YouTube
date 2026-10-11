# Restore the YouTube Button → YouTube TV Remapper

A quick recovery guide for restarting the working YouTube-button remapper on an Amazon Fire TV Stick 4K after a reboot.

> **Important:** This is a temporary background process. It has been tested to keep working after ADB disconnects, but **it does not survive a Fire TV reboot or power outage**.

## Step 1: Open Command Prompt from File Explorer

On your Windows computer:

1. Open **File Explorer**.
2. Navigate to your Platform Tools folder:

   `OneDrive → 04-Joel → zz Archive → Documents → platform-tools`

3. Click the **address bar** at the top of File Explorer, where the folder path appears.
4. Type:

   ```cmd
   cmd
   ```

5. Press **Enter**.

Command Prompt opens directly in the `platform-tools` folder, so you do not have to type the full folder path.

## Step 2: Reconnect to the Fire TV

In Command Prompt, run:

```cmd
adb connect 192.168.4.91:5555
```

Then check the connection:

```cmd
adb devices
```

Expected output:

```text
List of devices attached
192.168.4.91:5555    device
```

> If the Fire TV has a different IP address after rebooting, replace `192.168.4.91` with its current address. You may also need to approve an ADB debugging prompt on the TV.

## Step 3: Restart the YouTube TV remapper

Copy and paste the following **entire command as one line** into **Windows Command Prompt (`cmd.exe`)**:

```cmd
adb shell "nohup sh -c 'logcat -c; logcat -v brief | grep --line-buffered \"Remote button pressed.*keyCode: 294.*ACTION_DOWN.*scanCode=747\" | while read line; do monkey -p com.amazon.firetv.youtube.tv -c android.intent.category.LAUNCHER 1; done' >/dev/null 2>&1 </dev/null &"
```

The command should return to the Command Prompt without displaying an error.

This command watches for the physical YouTube button (`KEYCODE_APP_4`, Android keycode `294`, scan code `747`) and launches the installed YouTube TV app (`com.amazon.firetv.youtube.tv`).

## Step 4: Test the remote

1. Press **Home** on your Fire TV remote.
2. Press the red **YouTube** shortcut button.
3. Confirm that **YouTube TV** launches.

If it does not work, verify that ADB is connected and repeat Step 3. Amazon's original YouTube shortcut may still activate briefly because this method reacts to the button event rather than replacing Amazon's system mapping.

## Step 5: Disconnect the computer

After confirming the remapper works, run:

```cmd
adb disconnect
```

Close Command Prompt.

**Verified behavior:** YouTube TV continued launching from the physical button after the ADB connection was disconnected.

## Quick recovery checklist

- [ ] Open `platform-tools` in File Explorer.
- [ ] Type `cmd` in the File Explorer address bar and press Enter.
- [ ] Run `adb connect 192.168.4.91:5555`.
- [ ] Confirm `adb devices` reports `device`.
- [ ] Paste the background remapper command from Step 3.
- [ ] Test **Home → YouTube**.
- [ ] Run `adb disconnect`.

## Known limitations

- **Reboot:** The background process stops when the Fire TV restarts. Repeat these steps after a reboot or power outage.
- **No automatic startup:** A boot-time launcher has not been configured.
- **Amazon shortcut handling:** The original YouTube action is not blocked at the system level.
- **Software updates:** Future Fire OS updates may affect this method.
- **Network:** ADB connection requires that your Windows computer can reach the Fire TV on your local network.

## Security note

ADB debugging gives connected computers elevated access to device-management functions. Use ADB only on a trusted local network and authorize only computers you trust.

---

*Tested with a Fire TV Stick 4K running Fire OS 6.7.1.1. This guide documents the working manual recovery procedure, not a permanent firmware remap.*
