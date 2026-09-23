# Intel Tiger Lake audio on Arch Linux: SOF and legacy HDA troubleshooting

A focused workaround for Intel Tiger Lake laptops that have no sound, HDMI audio problems, or a "Dummy Output" while using Sound Open Firmware (SOF).

This guide shows how to force the legacy Intel HDA driver with:

```text
options snd-intel-dspcfg dsp_driver=1
```

It is intended for Arch Linux and Arch-based distributions, including CachyOS and EndeavourOS. The steps work with KDE Plasma and GNOME because the change happens at the kernel-driver level, before the desktop audio stack starts.

> [!WARNING]
> This is a workaround, not a universal audio fix. Forcing legacy HDA can restore speakers, headphones, or HDMI audio, but it can also disable an internal digital microphone (DMIC), SoundWire devices, or audio hardware that requires SOF. Read the rollback steps before rebooting.

## Contents

- [When to use this guide](#when-to-use-this-guide)
- [Tested configurations](#tested-configurations)
- [Check the current driver](#check-the-current-driver)
- [Apply the workaround](#apply-the-workaround)
- [Rebuild the initramfs](#rebuild-the-initramfs)
- [Verify the result](#verify-the-result)
- [Rollback](#rollback)
- [Troubleshooting](#troubleshooting)
- [Technical notes](#technical-notes)
- [References](#references)

## When to use this guide

Try this workaround only when an Intel Tiger Lake system has one or more of these symptoms:

- only "Dummy Output" is available
- HDMI or DisplayPort audio is missing
- speakers or headphones are detected but produce no sound
- audio routing is unreliable after boot or suspend
- SOF fails to load and the kernel log contains firmware, topology, or codec errors

Do not apply it just because `snd_sof` modules are present. SOF is the expected driver on many modern Intel laptops and may already be working correctly.

## Tested configurations

The workaround has been tested by the repository owner on the following Arch-family systems:

| Distribution | Desktop environments | CPU platform | Status |
| --- | --- | --- | --- |
| Arch Linux | KDE Plasma and GNOME | Intel 11th Gen Tiger Lake | Tested |
| CachyOS | KDE Plasma and GNOME | Intel 11th Gen Tiger Lake | Tested |
| EndeavourOS | KDE Plasma and GNOME | Intel 11th Gen Tiger Lake | Tested |

The original hardware used for this workaround was from the Huawei MateBook family. Results can differ between laptop models because manufacturers use different codecs, microphones, and audio wiring.

## Check the current driver

### 1. Identify the audio controller

```bash
lspci -nnk | grep -A3 -i audio
```

You can also use `inxi` if it is installed:

```bash
inxi -Aaz
```

SOF-related driver names may include:

```text
sof-audio-pci-intel-tgl
sof-essx8336
snd_sof_pci_intel_tgl
```

### 2. Check the kernel log

```bash
sudo dmesg | grep -Ei 'sof|snd|hda|soundwire'
```

Look for firmware-loading failures, missing topology files, codec errors, or repeated probe failures. Save this output before changing the driver; it is useful if the workaround does not help.

### 3. Record the current PipeWire devices

```bash
wpctl status
```

This gives you a before-and-after comparison of available audio devices.

## Apply the workaround

Create a dedicated modprobe configuration file:

```bash
printf '%s\n' 'options snd-intel-dspcfg dsp_driver=1' | \
  sudo tee /etc/modprobe.d/intel-audio-legacy-hda.conf
```

Confirm its contents:

```bash
cat /etc/modprobe.d/intel-audio-legacy-hda.conf
```

Expected output:

```text
options snd-intel-dspcfg dsp_driver=1
```

Using a dedicated file makes the change easy to identify and remove. Do not edit files managed by a package under `/usr/lib/modprobe.d/`.

## Rebuild the initramfs

Arch-based distributions do not all use the same initramfs generator. Check which tool is installed:

```bash
if command -v dracut-rebuild >/dev/null 2>&1; then
  echo "Use: sudo dracut-rebuild"
elif command -v mkinitcpio >/dev/null 2>&1; then
  echo "Use: sudo mkinitcpio -P"
else
  echo "No supported initramfs generator found"
fi
```

Run only the matching command.

### EndeavourOS and other systems using Dracut

Modern EndeavourOS installations use Dracut by default:

```bash
sudo dracut-rebuild
```

If `dracut-rebuild` is unavailable but the system uses plain Dracut, rebuild all installed images with:

```bash
sudo dracut --regenerate-all --force
```

### Arch Linux, CachyOS, and systems using mkinitcpio

```bash
sudo mkinitcpio -P
```

Read the output and resolve any initramfs generation error before rebooting.

### Reboot

```bash
systemctl reboot
```

## Verify the result

After rebooting, run these checks.

### 1. Confirm the requested mode

```bash
cat /sys/module/snd_intel_dspcfg/parameters/dsp_driver
```

Expected output:

```text
1
```

### 2. Check the active audio driver

```bash
lspci -nnk | grep -A3 -i audio
```

The audio controller should now show `snd_hda_intel` instead of a Tiger Lake SOF driver.

You can inspect the loaded modules as well:

```bash
lsmod | grep -E 'snd_(hda_intel|sof)'
```

Some `snd_sof` modules may remain installed or loaded as dependencies. The important checks are the driver bound to the PCI audio controller and whether the expected devices now work.

### 3. Check PipeWire

```bash
wpctl status
```

Test every device you need:

- internal speakers
- wired headphones and jack detection
- HDMI or DisplayPort audio
- internal microphone
- suspend and resume

A working speaker or HDMI output does not guarantee that the internal microphone still works.

## Rollback

If audio becomes worse, the internal microphone disappears, or the system still has no sound, remove the override:

```bash
sudo rm /etc/modprobe.d/intel-audio-legacy-hda.conf
```

Rebuild the initramfs with the same tool used during installation.

For Dracut-based systems:

```bash
sudo dracut-rebuild
```

Or, when using plain Dracut:

```bash
sudo dracut --regenerate-all --force
```

For mkinitcpio-based systems:

```bash
sudo mkinitcpio -P
```

Then reboot:

```bash
systemctl reboot
```

The kernel will return to automatic driver selection (`dsp_driver=0`).

## Troubleshooting

### The parameter still shows `0`

Check that the configuration file is valid:

```bash
modprobe --showconfig | grep -F 'snd-intel-dspcfg'
```

Confirm that the correct initramfs generator completed without errors, then reboot again.

### `dracut-rebuild` is not found

Do not install another initramfs generator just for this guide. Check the tools already used by the system:

```bash
command -v dracut-rebuild
command -v dracut
command -v mkinitcpio
```

Use the matching command from the initramfs section.

### `mkinitcpio -P` reports that no presets exist

The system probably uses Dracut rather than mkinitcpio. Check for `dracut-rebuild` and use it instead.

### Legacy HDA loads, but there is still no sound

The laptop may require SOF for its internal codec, DMIC, or SoundWire connection. Roll back the override, reboot, and investigate the original SOF error instead of keeping legacy HDA forced.

Useful diagnostics:

```bash
sudo dmesg | grep -Ei 'sof|snd|hda|soundwire'
wpctl status
pactl info
systemctl --user status pipewire pipewire-pulse wireplumber
```

### Sound works, but the internal microphone is missing

This is a known limitation of forcing legacy HDA on hardware that routes the digital microphone through the DSP. If the microphone is required, roll back to automatic driver selection and troubleshoot SOF.

### PipeWire does not update its device list

First reboot after changing the kernel driver. If PipeWire still shows stale devices, restart the user services:

```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
```

Then check again:

```bash
wpctl status
```

## Technical notes

The `snd-intel-dspcfg` module controls driver selection for supported Intel audio DSP hardware.

| Value | Selection |
| --- | --- |
| `0` | Automatic selection |
| `1` | Legacy Intel HDA |
| `2` | Intel SST |
| `3` | Sound Open Firmware (SOF) |
| `4` | Intel AVS on kernels that support it |

This guide uses:

```text
options snd-intel-dspcfg dsp_driver=1
```

That setting forces legacy HDA at boot. It does not modify PipeWire, KDE Plasma, GNOME, or the bootloader configuration. Rebuilding the initramfs ensures the modprobe option is available when the audio driver is selected early in the boot process.

SOF documentation describes legacy HDA as a useful diagnostic fallback for speakers and headphones, while warning that digital microphone capture may be unavailable. For that reason, keep automatic selection unless legacy HDA fixes a real problem on your hardware.

## References

- [SOF Project documentation](https://thesofproject.github.io/latest/)
- [Linux kernel source: Intel DSP driver selection](https://github.com/torvalds/linux/blob/master/sound/hda/core/intel-dsp-config.c)
- [ArchWiki: mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
- [EndeavourOS PKGBUILDS: eos-dracut](https://github.com/endeavouros-team/PKGBUILDS/blob/master/eos-dracut/README.md)

## Contributing test results

When reporting a result, include:

```text
Distribution and release:
Kernel (`uname -r`):
Desktop (KDE Plasma/GNOME):
Session (Wayland/X11):
Laptop model:
CPU generation:
Audio controller (`lspci -nnk`):
Initramfs tool:
Speakers:
Headphones:
HDMI/DisplayPort:
Internal microphone:
Suspend/resume:
```

Do not include the full `dmesg` output without checking it for serial numbers or other identifying information first.
