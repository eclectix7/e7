# Microphone Fix — Pin Config Override for ALC256 DMIC

Created: 2026-05-25

## Problem

Sound output was fixed by forcing the legacy HDA driver (`snd_intel_dspcfg.dsp_driver=1` in GRUB). However, the internal digital microphones (4 mics around the webcam) are not detected.

**Root cause:** The ALC256 codec has a digital microphone input at HDA pin node 0x1d. The BIOS configures it as `0x4058ba2d` = `[N/A]` (not connected / disabled). With `dsp_driver=1`, the DSP path that would normally handle these mics is bypassed, so we need to enable the DMIC through the HDA codec directly.

## The Fix

Two changes (both additive to the existing sound fix — no reverting):

1. **Create patch file:** `/etc/modprobe.d/alc256-dmic.patch` and copy to `/lib/firmware/alc256-dmic.patch`
   - Contains: `0x10ec0256 0x10251272 0x1d 0x90a6012e`
   - This tells the `snd-hda-intel` driver to override pin 0x1d's config
   - `0x90a6012e` marks the pin as a Fixed-function internal digital microphone
   - **Note:** Must be in `/lib/firmware/` — the kernel's HDA patch system uses `request_firmware()` (firmware search path), NOT the regular filesystem

2. **Edit GRUB:** `snd-hda-intel.patch=alc256-dmic.patch` added alongside existing `snd_intel_dspcfg.dsp_driver=1`
   - Use the bare filename (no path) — the firmware loader finds it automatically

### Full GRUB line after change:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash snd_intel_dspcfg.dsp_driver=1 snd-hda-intel.patch=/etc/modprobe.d/alc256-dmic.patch"
```

## Why This Works

The ALC256 codec has two independent input paths:
- **Analog path:** Headset mic jack (pin 0x19) → mixer 0x23 → ADC 0x08 → "ALC256 Analog" ALSA device
- **Digital path:** Internal DMIC array (pin 0x1d) → mixer 0x23 → ADC 0x08 → same ALSA device (different channel)

Both feed into the same ALSA capture device. The DMIC path exists at the hardware level but is disabled in firmware (BIOS says N/A). The HDA driver respects this unless overridden. The `patch` parameter forces the driver to see the DMIC as present.

After the fix, the internal mics should appear as an additional capture source in PipeWire, or as channels within the existing "Built-in Audio Analog Stereo" source.

## Files Changed

| File | Type | Change |
|------|------|--------|
| `/etc/default/grub` | Edited | Added `snd-hda-intel.patch=/etc/modprobe.d/alc256-dmic.patch` |
| `/etc/modprobe.d/alc256-dmic.patch` | Created | ALC256 pin config override for node 0x1d |
| `~/e7/kaizen/dsp/` | Reference | Diagnostics and logs |

## If the DMIC Still Doesn't Appear After Reboot

### Step 1 — Verify the patch was applied
Check if the driver accepted the patch:
```bash
cat /sys/class/sound/hwC0D0/driver_pin_configs
```
Should show `0x1d 0x90a6012e` in the list (alongside the existing `0x19` override).

Also check the proc dump:
```bash
cat /proc/asound/card0/codec#0 | grep -A10 'Node 0x1d'
```
If `Pin Default` still shows `0x4058ba2d: [N/A]`, the patch wasn't applied.

### Step 2 — Check if the DMIC is muted in the mixer
```bash
cat /proc/asound/card0/codec#0 | grep -A12 'Node 0x23'
```
Look at the `Amp-In vals` line. Connection 5 (which is pin 0x1d) should have `[0x00 0x00]` for unmuted. If it shows `[0x80 0x80]`, it's muted.

### Step 3 — Try an alternate pin config value
Some ALC256 implementations use different pin config values. Edit the patch file:

```
# Option A: 0x90a6012e (tried first — most common)
# Option B: 0x92a6012e (different location bits)
# Option C: 0x90a60a2e (different default device)
```

Example of switching:
```bash
echo '0x10ec0256 0x10251272 0x1d 0x92a6012e' | sudo tee /etc/modprobe.d/alc256-dmic.patch
```
Then reboot.

### Step 4 — Try the `dmic_detect` module parameter
If the pin override alone doesn't work, the DMIC might need the HDA codec's `dmic_detect` logic to handle it. You could add to GRUB:
```
snd-hda-intel.dmic_detect=Y
```
(It's already Y by default, but just in case)

### Step 5 — Check webcam USB audio directly
It's possible the 4 mics are actually part of the webcam module via USB Audio, not the ALC256 codec. Check:
```bash
lsusb -t | grep -i audio
pactl list sources short
arecord -l
```
If none of the above work, the mics might be connected only via the DSP (I2S), which is inaccessible with `dsp_driver=1`.

## Backout Plan — If Audio Output Breaks

This is extremely unlikely since we're only adding a pin config patch (not changing the driver selection), but if something goes wrong:

### Boot-time revert
At the GRUB boot menu (hold Shift during boot):
- Select the same kernel entry
- Press `e` to edit
- Remove `snd-hda-intel.patch=/etc/modprobe.d/alc256-dmic.patch` from the command line
- Press Ctrl+X or F10 to boot
This temporarily undoes the change for one boot.

### Permanent revert
```bash
# Remove the patch parameter from GRUB
sudo sed -i 's| snd-hda-intel.patch=alc256-dmic.patch||' /etc/default/grub
sudo update-grub
# (Optional) Delete the patch files
sudo rm /etc/modprobe.d/alc256-dmic.patch /lib/firmware/alc256-dmic.patch
sudo reboot
```

This reverts only the microphone change. The sound fix (`snd_intel_dspcfg.dsp_driver=1`) stays intact.

## What This Does NOT Change

- `snd_intel_dspcfg.dsp_driver=1` — still active, sound output continues working
- `/etc/modprobe.d/blacklist-avs.conf` — still active, AVS driver stays blacklisted
- No kernel/driver changes — just a codec pin config override