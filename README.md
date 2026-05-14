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
| External WiFi — Atheros AR9271 (`ath9k_htc`) | ✅ Built-in module, ⚠️ Need someone to check |
| External WiFi — Realtek RTL8xxxU (`rtl8xxxu`) | ✅ Built-in module, ⚠️ Need someone to check |
| chroot| ✅ Working |
| Metasploit | ✅ Working |
| All stock LineageOS functionality | ✅ Preserved |
and many more features are working

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
| `boot.img` | NetHunter kernel boot image |
| `dtbo.img` | Device tree overlay |
| `vendor_dlkm.img` | Vendor kernel modules (ath9k, rtl8xxxu, etc.) |
| `bluejay-kali_nethunter-hid.zip` | Magisk module — enables `/dev/hidg0` on every boot |

---

## Installation

> **Note:** The Pixel 6a uses dynamic partitions. The NetHunter installer zip **must** be flashed via Magisk (not TWRP). The NetHunter installer handles flashing the kernel automatically — no manual `fastboot flash boot` needed.

### Step 1 — Flash WiFi kernel modules (optional)

Skip this step if you don't need external WiFi adapters (ath9k, rtl8xxxu). The HID feature works without it.

`vendor_dlkm` is a logical partition and can only be written from fastbootd, not from within Android. Boot into fastboot mode (hold **Power + Volume Down**), then:

```bash
adb reboot fastboot
```

When the device shows the fastbootd menu:

```bash
fastboot flash vendor_dlkm vendor_dlkm.img
fastboot reboot
```

### Step 2 — Install Magisk

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
