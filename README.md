# bluejay-kali_nethunter

Kali NetHunter kernel for the **Google Pixel 6a (bluejay)** running **LineageOS 21 (Android 14)**.

- **Kernel:** `6.1.145-android14-11-maybe-dirty-NetHunter` (GKI 2.0)
- **Base:** LineageOS 21 kernel source (Kleaf/Bazel build system)
- **Tested on:** LineageOS 21 + Magisk 30.7

---

## Features

| Feature | Status |
|---|---|
| USB HID keyboard (`/dev/hidg0`) | ✅ Working |
| Rucky / NetHunter HID attacks | ✅ Working |
| External WiFi — Atheros AR9271 (`ath9k_htc`) | ✅ Built-in module |
| External WiFi — Realtek RTL8xxxU (`rtl8xxxu`) | ✅ Built-in module |
| All stock LineageOS functionality | ✅ Preserved |

---

## Requirements

- Google Pixel 6a (`bluejay`)
- LineageOS 21 (Android 14) installed
- Unlocked bootloader
- Magisk (any recent version) installed
- `adb` and `fastboot` on your PC

---

## Files

| File | Description |
|---|---|
| `kernel/boot.img` | NetHunter kernel boot image |
| `kernel/dtbo.img` | Device tree overlay |
| `kernel/vendor_dlkm.img` | Vendor kernel modules (ath9k, rtl8xxxu, etc.) |
| `magisk-module/bluejay-kali_nethunter-hid.zip` | Magisk module — enables `/dev/hidg0` on every boot |

---

## Installation

### Step 1 — Flash the kernel

Boot your device into fastboot mode (hold **Power + Volume Down**):

```bash
fastboot flash boot kernel/boot.img
fastboot flash dtbo kernel/dtbo.img
fastboot reboot fastboot
```

When the device reboots into **fastbootd** (you'll see the fastbootd menu):

```bash
fastboot flash vendor_dlkm kernel/vendor_dlkm.img
fastboot reboot
```

> `vendor_dlkm` is a logical partition and must be flashed from fastbootd, not from the bootloader.

### Step 2 — Install the HID Magisk module

1. Copy `magisk-module/bluejay-kali_nethunter-hid.zip` to your device storage
2. Open the **Magisk** app
3. Go to **Modules** → **Install from storage**
4. Select `bluejay-kali_nethunter-hid.zip`
5. Tap **Reboot**

### Step 3 — Verify

After reboot, confirm HID is active:

```bash
adb shell ls -la /dev/hidg0
# Expected: crw-rw-rw- 1 root root 481, 0 ...
```

Check the setup log:

```bash
adb shell cat /data/local/tmp/nethunter-hid.log
```

---

## Using HID Attacks

Open **Rucky** or the **Kali NetHunter** app. Both should detect `/dev/hidg0` and allow HID keyboard payload delivery.

---

## How the HID Module Works

Android's USB HAL (`android.hardware.usb.gadget-service.gs101`) owns the USB configfs gadget entirely. The Linux kernel refuses to link a new USB function to an active config while the UDC is bound (returns `EINVAL`), and stopping the HAL kills it — causing `hwservicemanager` to detect a HAL death and trigger a system restart loop every 2-3 minutes.

The module solves this with a **SIGSTOP/SIGCONT freeze**:

1. Wait for boot to complete and HAL to finish initial USB setup
2. Create `hid.usb0` configfs function with a standard boot keyboard descriptor
3. `SIGSTOP` the HAL process — freezes it without killing it, so `hwservicemanager` sees no death event and no watchdog fires
4. Unbind the UDC — USB disconnects briefly (~100ms)
5. Link `hid.usb0` into `configs/b.1/` — succeeds because UDC is now unbound
6. Rebind the UDC — USB reconnects presenting ADB + HID as a composite device
7. `SIGCONT` the HAL — it wakes up idle, gadget already configured, does nothing
8. `/dev/hidg0` appears with `crw-rw-rw-` permissions

The entire freeze window is under 100ms — well below any HAL watchdog timeout.

---

## Building from Source

The kernel is built using the **Kleaf/Bazel** build system from LineageOS kernel sources.

Key customizations:
- `CONFIG_LOCALVERSION="-NetHunter"` in `aosp/arch/arm64/configs/gki_defconfig`
- `CONFIG_USB_CONFIGFS_F_HID=y` (enables HID gadget function)
- `CONFIG_ATH9K_HTC=m` and `CONFIG_RTL8XXXU=m` in `private/devices/google/bluejay/bluejay_defconfig`
- `kmi_symbol_list = None` in `private/devices/google/bluejay/BUILD.bazel` to allow out-of-tree module symbols
- GKI built from source (`--config=use_source_tree_aosp --config=bluejay`) to match vermagic with device modules

Build command:
```bash
tools/bazel run --config=use_source_tree_aosp --config=bluejay \
    //private/devices/google/bluejay:bluejay -- --dist_dir=out/bluejay/dist
```

---

## Troubleshooting

**`/dev/hidg0` not present after reboot**
- Check the log: `adb shell cat /data/local/tmp/nethunter-hid.log`
- Make sure the Magisk module is enabled (not showing a red cross in Magisk → Modules)
- Try rebooting again — the module waits up to 35 seconds for the USB gadget to be ready

**Device bootloops after module install**
- Boot into safe mode: reboot and hold **Volume Down** when the boot animation appears
- This disables all Magisk modules for that boot
- Use `adb root` then disable the module: `touch /data/adb/modules/nethunter-hid/disable`

**Rucky says "kernel not supported"**
- Rucky checks for `/dev/hidg0` — confirm it exists with `adb shell ls /dev/hidg0`
- If it exists but Rucky still complains, check Rucky has root access granted in Magisk

---

## Credits

- [Kali NetHunter](https://www.kali.org/docs/nethunter/) project
- [LineageOS](https://lineageos.org/) kernel source
- [Magisk](https://github.com/topjohnwu/Magisk) by topjohnwu
