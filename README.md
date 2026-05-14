# bluejay-kali_nethunter

Kali NetHunter kernel for the **Google Pixel 6a (bluejay)** running **LineageOS 23.2 (Android 16)**.

- **Kernel:** `6.1.145-android14-11-maybe-dirty-NetHunter` (GKI 2.0)
- **Base:** LineageOS 23.2 kernel source (Kleaf/Bazel build system)
- **Tested on:** LineageOS 23.2 + Magisk 30.7

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
- LineageOS 23.2 (Android 16) installed
- Unlocked bootloader
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

> **Note:** The Pixel 6a uses dynamic partitions. The NetHunter installer zip **must** be flashed via Magisk (not TWRP). The NetHunter installer handles flashing the kernel automatically — no manual `fastboot flash boot` needed.

### Step 1 — Flash WiFi kernel modules (optional)

Skip this step if you don't need external WiFi adapters (ath9k, rtl8xxxu). The HID feature works without it.

`vendor_dlkm` is a logical partition and can only be written from fastbootd, not from within Android. Boot into fastboot mode (hold **Power + Volume Down**), then:

```bash
fastboot reboot fastboot
```

When the device shows the fastbootd menu:

```bash
fastboot flash vendor_dlkm kernel/vendor_dlkm.img
fastboot reboot
```

### Step 2 — Install Magisk

If Magisk is not already installed:

1. Download the latest **Magisk APK** from [github.com/topjohnwu/Magisk/releases](https://github.com/topjohnwu/Magisk/releases)
2. `adb install Magisk-*.apk`
3. Open the **Magisk** app → tap **Install** → **Direct Install (Recommended)**
4. Tap **Reboot**

### Step 3 — Flash the NetHunter installer via Magisk

The installer bundles the NetHunter kernel and uses AnyKernel3 to patch your `boot` and `dtbo` partitions automatically. It also installs the Kali NetHunter app, terminal, KeX, and chroot environment.

1. Download `nethunter-bluejay-los-sixteen-kalifs_nano.zip` from the [Releases](../../releases) page
2. Copy it to your device
3. Open **Magisk** → **Modules** → **Install from storage**
4. Select the NetHunter zip
5. Tap **Reboot**

### Step 4 — Install the HID Magisk module

This enables `/dev/hidg0` (USB HID keyboard) on every boot.

1. Copy `magisk-module/bluejay-kali_nethunter-hid.zip` to your device
2. Open **Magisk** → **Modules** → **Install from storage**
3. Select `bluejay-kali_nethunter-hid.zip`
4. Tap **Reboot**

After reboot, `/dev/hidg0` will be present and Rucky / NetHunter HID attacks will work.

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


## Authors

- **RishiRats** — [LinkedIn](https://www.linkedin.com/in/rishi-raturi/)
- **Claude** (Anthropic) — kernel build system fixes, HID gadget research & SIGSTOP/SIGCONT solution

## Credits

- [Kali NetHunter](https://www.kali.org/docs/nethunter/) project
- [LineageOS](https://lineageos.org/) kernel source
- [Magisk](https://github.com/topjohnwu/Magisk) by topjohnwu
