# Audio & Microphone — Diagnosis Log

Date: 2026-05-25
System: Acer laptop, Intel Sunrise Point-LP HD Audio (8086:9d71), Realtek ALC256
OS: Ubuntu 24.04, kernel 6.17.0-29-generic
Files: `~/e7/kaizen/dsp/`

## Problem

Neither sound output nor internal microphones worked with the default auto-detected audio driver.

## Fix Applied: GRUB + HDA Driver

Forced legacy HDA driver via GRUB cmdline:
```
snd_intel_dspcfg.dsp_driver=1
```

This bypasses the DSP (AVS/SST/SOF), which was completely broken on this hardware — it produced zero audio devices. The legacy `snd_hda_intel` driver drives the ALC256 codec directly.

Also blacklisted the AVS driver modules (`/etc/modprobe.d/blacklist-avs.conf`).

## Result

| Feature | Before | After |
|---------|--------|-------|
| Speakers/headphone out | ❌ | ✅ |
| Headset mic (jack) | ❌ | ✅ (plug in headset) |
| Internal mic array (4 around webcam) | ❌ | ❌ |

## Internal Microphone — Investigation

The 4 microphones around the webcam are physically connected to the DSP via I2S bus, **not** through the ALC256 HDA codec. With `dsp_driver=1`, the DSP is completely bypassed, cutting off that signal path entirely.

### What was attempted:

1. **Pin config override** — Changed HDA codec pin 0x1d from `[N/A]` to `[Fixed] Mic at Int N/A` via `hda-verb` ✅ (hardware-level success)
2. **Mixer unmute** — Tried 5 different hda-verb encodings to unmute the DMIC path in mixer 0x23 ❌ (none worked)
3. **ALSA ACP reconfiguration** — WirePlumber only exposes 1 input route (`analog-input-headset-mic`), no DMIC route exists
4. **SOF/SST switch** — `dsp_driver=3` was tested before the fix and produced zero audio devices

### Conclusion

The internal mics are not recoverable via the HDA driver. The hardware trade-off is:
- **DSP driver** (default/AVS/SOF): no audio at all (output or input)
- **Legacy HDA** (`dsp_driver=1`): audio output + headset jack mic work, internal mics inaccessible

This is a firmware/hardware limitation, not a configuration issue.

## Files Changed

| File | Status | Purpose |
|------|--------|---------|
| `/etc/default/grub` | Modified | Added `snd_intel_dspcfg.dsp_driver=1` + patch param |
| `/etc/modprobe.d/blacklist-avs.conf` | Created | Blacklists `snd_soc_avs` modules |
| `/etc/modprobe.d/alc256-dmic.patch` | Created | DMIC pin config override (exists but unused by driver) |
| `/lib/firmware/alc256-dmic.patch` | Created | Firmware-loadable copy of above |
| `~/e7/kaizen/dsp/BACKOUT.md` | Created | Revert plan for the sound fix |
| `~/e7/kaizen/dsp/MIC-FIX.md` | Created | Revert plan for microphone changes |

## Backout

See `~/e7/kaizen/dsp/BACKOUT.md` to revert the sound fix.
To revert the mic changes: remove `snd-hda-intel.patch=alc256-dmic.patch` from GRUB and delete `/lib/firmware/alc256-dmic.patch`.