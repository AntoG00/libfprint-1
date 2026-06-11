# Goodix 5117 (27c6:5117) Linux Support Guide

This branch (`goodix-5117-support`) adds functional support for the Goodix 5117 fingerprint sensor (USB ID `27c6:5117`).

However, Goodix sensors use "whitebox" cryptography and a Pre-Shared Key (PSK) to establish a TLS connection between the host and the fingerprint sensor over USB. This PSK is **unique to your specific laptop/sensor** and is periodically rotated by the Windows driver. 

To use this driver on Linux, you **must** extract your own sensor's PSK from Windows memory and compile it into this driver. If you do not change the keys in the source code, `fprintd` will fail with errors like `cipher operation failed` or `no shared cipher`.

## Prerequisites
1. A dual-boot Windows installation where the fingerprint sensor works normally.
2. `x64dbg` downloaded and installed on Windows.

## Step 1: Extracting the Clear-Text PSK from Windows
1. Boot into Windows.
2. Open **x64dbg** as Administrator.
3. Attach the debugger to the Windows User-Mode Driver Framework host process (`WUDFHost.exe`) that is hosting the Goodix biometric driver.
4. Set a breakpoint on the OpenSSL TLS callback or the internal driver's memory where the PSK is handled before the TLS handshake.
5. Trigger the fingerprint sensor (e.g., attempt to log in or enroll a finger in Windows Settings).
6. When the breakpoint is hit, dump the **32-byte clear-text PSK** from the CPU registers/memory. It will look something like `FA B8 44 88...`. 

## Step 2: Injecting the PSK into the Driver
Once you have your 32-byte PSK, you need to replace the placeholder arrays in the code before compiling:

1. Open `libfprint/drivers/goodixtls/goodixtls.c`.
2. Find the `actual_psk[]` array inside the `tls_server_psk_server_callback` function (around line 71).
3. Replace the hex values in that array with your **32-byte clear-text PSK**.

> **Note on the Obfuscated PSK:**
> The driver also receives an *obfuscated* version of the PSK over USB during startup. This driver has been modified (in `goodix5xx.c`) to bypass the strict check on this obfuscated blob, so you do not technically need to update `goodix_511_psk_0` in `goodix511.h` anymore—it will just print a harmless warning in your `journalctl` logs. 

## Step 3: Compile and Install
Once the PSK is updated, compile and install the driver:

```bash
meson setup --reconfigure --prefix=/usr --libdir=/usr/lib builddir
ninja -C builddir
sudo ninja -C builddir install
sudo systemctl restart fprintd
```

## Step 4: Test the Sensor
You can test the enrollment process directly:
```bash
fprintd-enroll
```
If everything is configured correctly, it will say `Enrolling right-index-finger finger.` and you can begin swiping your finger!

---
*Special thanks to the community for reverse-engineering the TLS-PSK handshakes.*
