# Audio Fix — Backout Plan

Created: 2026-05-25
Applied: `snd_intel_dspcfg.dsp_driver=1` in GRUB_CMDLINE_LINUX_DEFAULT

## If audio breaks after reboot (or other issues appear)

### Step 1 — Boot into the old kernel cmdline
At the GRUB boot menu (hold Shift / press Esc during boot):
- Select "Advanced options for ..."
- Choose the same kernel entry but **without** the `snd_intel_dspcfg.dsp_driver=1` parameter
- This is the default recovery kernel or the unmodified entry from before the change

This boots you with the original `quiet splash` only — temporarily undoing the change.

### Step 2 — Once booted, revert the GRUB file
```
sudo sed -i 's/snd_intel_dspcfg.dsp_driver=1//' /etc/default/grub
sudo update-grub
```

This removes the parameter. Future boots will use the original broken AVS driver again.

### Step 3 — (Optional) Remove the AVS blacklist
Since we created `/etc/modprobe.d/blacklist-avs.conf`, it stays harmless even after reverting — but if you want a completely clean slate:
```
sudo rm /etc/modprobe.d/blacklist-avs.conf
```

### Step 4 — Reboot back to normal
```
sudo reboot
```

## What the change did
`snd_intel_dspcfg.dsp_driver=1` forces the kernel to use the **legacy HDA driver** (`snd_hda_intel`) instead of the broken AVS driver. This is a widely-used, stable parameter on Intel Kaby Lake / Sunrise Point hardware. The only downside is no DSP audio offloading (irrelevant for this chipset).

## What was created/changed
| Path | Type | Change |
|------|------|--------|
| `/etc/default/grub` | Edited | Added `snd_intel_dspcfg.dsp_driver=1` |
| `/etc/modprobe.d/blacklist-avs.conf` | Created | Blacklists `snd_soc_avs` and `snd_soc_avs_probe` |
| `~/e7/kaizen/dsp/` | Created | Diagnostics and logs |