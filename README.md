# Goodix 5117 (27c6:5117) Linux Support Guide

This branch (`goodix-5117-support`) adds functional support for the Goodix 5117 fingerprint sensor (USB ID `27c6:5117`).

However, Goodix sensors use "whitebox" cryptography and a Pre-Shared Key (PSK) to establish a TLS connection between the host and the fingerprint sensor over USB. This PSK is **unique to your specific laptop/sensor** and is periodically rotated by the Windows driver. 

To use this driver on Linux, you **must** extract your own sensor's PSK from Windows memory and compile it into this driver. If you do not change the keys in the source code, `fprintd` will fail with errors like `cipher operation failed` or `no shared cipher`.

## 🔑 How to Extract Your Master PSK from Windows

To get this driver working, you need your sensor's unique 32-byte Pre-Shared Key (PSK). Because this key is generated and stored securely by the host, the only reliable way to get it is to dump it directly from your Windows machine's RAM during the sensor handshake.

This guide uses `x64dbg` to trap the `mbedtls_ssl_conf_psk` function inside the Windows User-Mode Driver Framework (`wudfhost.exe`) before the handshake completes.

### Prerequisites
1. A Windows installation (VM or dual-boot) where the Goodix fingerprint sensor is currently working.
2. `x64dbg` (You must use the 64-bit version, `x64dbg.exe`).
3. Process Explorer (Run as Administrator).

### Step 1: Locate the Driver Process
Windows hosts biometric drivers inside a generic `wudfhost.exe` process. We need to find the specific one running the Goodix driver.
1. Open Process Explorer as Administrator.
2. Press `Ctrl + F` and search for `gfspi.dll` (the Goodix SPI driver).
3. Note the PID (Process ID) of the `wudfhost.exe` process that has this DLL loaded.

### Step 2: Attach the Debugger
> **Note:** Windows has a strict watchdog timer for biometric drivers. If you pause the driver for too long, Windows will terminate the process. Read Steps 3 and 4 completely before executing them so you can move quickly!

1. Open `x64dbg` as Administrator.
2. Go to `File > Attach` (`Alt + A`) and select the `wudfhost.exe` process with the PID you noted.
3. The debugger will pause on a System Breakpoint. Spam the `F9` key (Run) a few times until the bottom-left corner steadily says `Running` and the program stops pausing.

### Step 3: Set the Trap
We need to catch the cryptography engine exactly when it loads the master key.
1. Go to the **Symbols** tab and click `gfspi.dll` in the left-hand module list.
2. Right-click anywhere in the main CPU window and select `Search for > Current Module > String references`.
3. In the search box at the bottom, search for `mbedtls_ssl_conf`.
4. Double-click the string that says `[FAILED] mbedtls_ssl_conf_psk...`. This will take you to the error-logging code.
5. Scroll UP a few lines to find the actual function call. Look for the instruction: `call gfspi.[Address]`.
6. Click that call instruction and press `F2` to set a breakpoint (the address will turn red).
7. Ensure this is the only breakpoint you have set.

### Step 4: Trigger the Handshake & Dump the Key
1. Ensure `x64dbg` says `Running`.
2. Put your laptop to Sleep and immediately Wake it up (or manually trigger the Windows Hello lock screen).
3. The driver will attempt to establish the TLS tunnel, and `x64dbg` will instantly snap to the foreground, paused exactly on your call breakpoint.
4. Look at the Registers pane on the top right:
   - Verify `R8` equals `0000000000000020` (This confirms the key length is 32 bytes).
   - `RDX` holds the memory address of your key.
5. Right-click `RDX` and select `Follow in Dump` (or `Follow in Dump > Dump 1`).
6. The bottom dump window will reveal your clear-text 32-byte key (it will look like highly randomized hex across two rows).
7. Highlight exactly 32 bytes, right-click, and select `Copy`.

> ⚠️ **Watchdog Crash?** If `x64dbg` says "Terminated" before you can copy the key, Windows killed the process. Simply open `services.msc`, restart the Windows Biometric Service, find the new PID in Process Explorer, and try again a bit faster!

## Step 5: Injecting the PSK into the Driver
Take the copied hex string and format it into a standard C-array. Replace the placeholder array in the driver source code with your actual key before compiling:

1. Open `libfprint/drivers/goodixtls/goodixtls.c`.
2. Find the `actual_psk[]` array inside the `tls_server_psk_server_callback` function (around line 71).
3. Replace the hex values in that array with your **32-byte clear-text PSK**.

```c
// Example format:
const guint8 actual_psk[32] = {
    0xFA, 0xB8, 0x44, 0x88, 0x59, 0xCD, 0x50, 0xF6, 
    0x13, 0x0A, 0xA4, 0xF4, 0x65, 0x61, 0xC8, 0x6A,
    0x4C, 0x63, 0x79, 0xBA, 0xC9, 0x08, 0x86, 0x1A,
    0x7E, 0x77, 0xED, 0x69, 0x3E, 0x0C, 0x28, 0x0A
};
```

> **Note on the Obfuscated PSK:**
> The driver also receives an *obfuscated* version of the PSK over USB during startup. This driver has been modified (in `goodix5xx.c`) to bypass the strict check on this obfuscated blob, so you do not technically need to update `goodix_511_psk_0` in `goodix511.h` anymore—it will just print a harmless warning in your `journalctl` logs. 

## Step 6: Compile and Install
Once the PSK is updated, compile and install the driver:

```bash
meson setup --reconfigure --prefix=/usr --libdir=/usr/lib builddir
ninja -C builddir
sudo ninja -C builddir install
sudo systemctl restart fprintd
```

## Step 7: Test the Sensor
You can test the enrollment process directly:
```bash
fprintd-enroll
```
If everything is configured correctly, it will say `Enrolling right-index-finger finger.` and you can begin swiping your finger!

---
*Special thanks to the community for reverse-engineering the TLS-PSK handshakes.*

## Development note
An LLM performed the majority of the implementation work for this branch. I reviewed, edited, validated, and tested the result on my laptop's Goodix 5117 hardware before submission.
