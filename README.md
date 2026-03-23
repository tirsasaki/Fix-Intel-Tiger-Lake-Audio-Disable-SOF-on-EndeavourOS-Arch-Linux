# 🎧 Fix Intel Tiger Lake Audio (Disable SOF) on EndeavourOS / Arch Linux

## Overview

Some Intel Tiger Lake laptops (11th Gen) use the **SOF (Sound Open Firmware)** driver by default, which can cause issues such as:

* No sound over HDMI
* Dummy output only
* Audio routing problems
* Inconsistent behavior with PipeWire
* Compatibility issues with certain applications

This guide shows how to switch from the SOF driver to the legacy **Intel HDA (`snd_hda_intel`)** driver on EndeavourOS / Arch Linux.

---

## ⚠️ Warning

Switching to the legacy HDA driver may cause:

* Internal speakers not working
* Internal microphone not detected
* Headphone jack issues

Modern laptops are often designed specifically for SOF.

Proceed only if you understand the trade-offs.

---

## 🖥️ Tested On

* **Distribution:** EndeavourOS (Arch-based)
* **Kernel:** Linux 6.x
* **Desktop:** GNOME (Wayland)
* **CPU:** Intel 11th Gen (Tiger Lake)
* **Audio System:** PipeWire
* **Hardware Example:** Huawei MateBook series

---

## 🔍 Check Current Audio Driver (Optional)

Before applying the fix:

```bash
inxi -A
```

If you see entries such as:

```
sof-audio-pci
sof-essx8336
```

Your system is using the SOF driver.

---

## 🚀 Switch to Legacy Intel HDA Driver

### 1. Create a modprobe configuration file

Create a new configuration file:

```bash
sudo nano /etc/modprobe.d/intel-audio.conf
```

Add the following line:

```bash
options snd-intel-dspcfg dsp_driver=1
```

Save and exit.

---

### 2. Rebuild initramfs (Required on Arch-based systems)

```bash
sudo mkinitcpio -P
```

---

### 3. Reboot

```bash
reboot
```

---

## ✅ Verify the Driver After Reboot

Run:

```bash
inxi -A
```

If successful, you should see:

```
driver: snd_hda_intel
```

instead of SOF-related drivers.

---

## 🔄 Revert to SOF Driver (If Needed)

To restore the default behavior:

```bash
sudo rm /etc/modprobe.d/intel-audio.conf
sudo mkinitcpio -P
reboot
```

---

## 💡 Why Create a New File?

It is recommended to create a dedicated configuration file instead of editing existing ones (e.g., `alsa.conf`) because:

* Prevents conflicts with system defaults
* Survives package updates
* Easier troubleshooting
* Easy rollback
* Follows Arch Linux best practices

---

## 🧠 Technical Explanation

Modern Intel platforms support multiple audio driver modes:

| Mode       | Driver          | Description                     |
| ---------- | --------------- | ------------------------------- |
| Auto       | System decides  | Default behavior                |
| Legacy HDA | `snd_hda_intel` | Older, highly compatible driver |
| SOF        | `sof-audio-pci` | Modern firmware-based driver    |
| Hybrid     | SOF + fallback  | Mixed mode                      |

Setting:

```
dsp_driver=1
```

forces the system to use the legacy HDA driver.

---

## 📌 Notes

* HDMI audio often works reliably with the HDA driver
* Internal audio devices may depend on SOF firmware
* PipeWire does not need reconfiguration for this change
* Works on most Arch-based distributions

---

## 📄 License

This guide is provided for educational purposes.
Use at your own risk.

