# GruperTal/evdi fork — notes for whoever sets this up next

This is a fork of [DisplayLink/evdi](https://github.com/DisplayLink/evdi) with one
extra commit (`0d1c899`, identical to upstream PR
[#593](https://github.com/DisplayLink/evdi/pull/593)). It exists so a Silicon Motion
USB display adapter works on Arch/Omarchy. If you are an agent helping the owner on a
new machine, read this first, then check whether the fork is still needed (see
[When to drop this fork](#when-to-drop-this-fork)).

## Hardware this is for

- **Adapter:** Satechi USB-C Multiport MX Adapter (ST-UCMXAM). It has two HDMI ports
  that work in completely different ways:
  - **Native HDMI (4K60):** plain USB-C DisplayPort alt mode through the laptop GPU.
    No driver.
  - **Driver HDMI (4K30):** a Silicon Motion SM768 USB graphics chip,
    `lsusb` shows `090c:0768 Silicon Motion ... SMI USB Display`. It needs the SMI
    userspace daemon + evdi. Without it, a monitor on that port stays dark (you may
    see the Silicon Motion logo for a second).
  - The adapter also shows up as a USB **mass storage** device. That is just the
    InstantView installer for macOS/Windows. Ignore it.
- **Laptop it was debugged on:** Dell XPS 13 Plus 9320, Intel Alder Lake (i915),
  Arch Linux / Omarchy, Hyprland, kernel 7.2.3, evdi 1.15.0.

## Setup on a new machine

```bash
# 1. Build and install evdi from this fork (provides evdi-dkms-git, so AUR deps are satisfied)
git clone https://github.com/GruperTal/evdi.git ~/Work/evdi
mkdir -p ~/Work/evdi-dkms-grupertal-git
cp ~/Work/evdi/arch/PKGBUILD ~/Work/evdi-dkms-grupertal-git/
cd ~/Work/evdi-dkms-grupertal-git && makepkg -si   # needs linux-headers + dkms

# 2. Install the Silicon Motion daemon (AUR). Its udev rule starts the service on plug-in.
yay -S smi-usbdisplay

# 3. If the service already crash-looped before step 1, clear it
sudo systemctl reset-failed smiusbdisplay && sudo systemctl restart smiusbdisplay
```

The package is deliberately named `evdi-dkms-grupertal-git`, not `evdi-dkms-git`, so
`yay -Syu --devel` does not replace it with the unpatched AUR build.

## Problem 1: SMI daemon segfaults (the reason this fork exists)

**Symptom:** `systemctl status smiusbdisplay` shows `SIGSEGV` / `start-limit-hit`, and
the crash stack is:

```
#0 libc.so.6 (strlen)
#1 evdi_open_attached_to (libevdi.so.1)
#2 SMIDev::plugInMonitor(int, bool) (SMIUSBDisplayManager)
```

**Root cause (two bugs combined):**

1. In libevdi, the deprecated `evdi_open_attached_to()` calls
   `strlen(sysfs_parent_device)` without a NULL check, even though NULL is documented
   as meaning "generic device".
2. `SMIUSBDisplayManager` (closed source, SMI driver v2.24.8.0) calls
   `evdi_get_lib_version()` and picks the entry point like this (from disassembly):
   `if (major == 1 && minor > 13 && patch > 3) evdi_open_attached_to_fixed(NULL, 0); else evdi_open_attached_to(NULL);`
   So on **any libevdi whose patch number is 0–3 (e.g. 1.15.0)**, it takes the unsafe
   path and crashes. evdi 1.14.16, which SMI bundles, has patch 16, so it never
   crashed there.

**Fix:** commit `0d1c899` makes the wrapper pass length 0 for NULL. Quick check that a
libevdi has the fix (it must print instead of segfaulting):

```bash
cat > /tmp/t.c <<'EOF'
#include <stdio.h>
void *evdi_open_attached_to(const char *);
int main(void) { evdi_open_attached_to(NULL); puts("ok"); return 0; }
EOF
gcc /tmp/t.c -o /tmp/t -levdi && /tmp/t   # unpatched: Segmentation fault (exit 139)
```

## Problem 2: the native HDMI port shows nothing

This problem has nothing to do with evdi, but it came up in the same session.

**Symptom:** the monitor on the native port stays dark, even though the cable and
monitor work on the other port. You'll see:

- `/sys/class/drm/card1-DP-*/status` all show `disconnected`.
- `/sys/class/typec/port1-partner/port1-partner.0/displayport/` shows `hpd: 0` and
  `configuration: [USB]`, while the alt mode (`svid=ff01`) shows `active=yes`.
- With `echo 0x106 | sudo tee /sys/module/drm/parameters/debug`, the kernel log shows
  HPD pulses on `DDI TC3`, every AUX read failing with `-6`, and no
  `TC port mode reset` line.

**What fixed it:** **rebooting with the adapter, charger and both monitors already
connected.** Hot-plugging the adapter did not recover it.

**What did not help:** reloading `ucsi_acpi`/`typec_ucsi`, and forcing
`echo detect > /sys/class/drm/card1-DP-3/status`.

For comparison, the owner's home monitor dock (Lenovo monitors over DP MST) hot-plugs
fine on the same kernel and port. So this is specific to how the Satechi's native port
negotiates, not a general USB-C video failure. If it keeps happening, the next things
to try are the laptop's other USB-C port and booting `linux-lts` to rule out a kernel
regression.

Remember to turn drm debug back off afterwards (`echo 0 | sudo tee ...`, or reboot).

## When to drop this fork

- **PR #593 is merged upstream** (`gh pr view 593 -R DisplayLink/evdi`), and the AUR
  `evdi-dkms-git` builds a commit that includes it. Then switch back:
  `yay -S evdi-dkms-git` (it conflicts with and replaces this package).
- **Or SMI ships a daemon that calls `evdi_open_attached_to_fixed` unconditionally.**

## Keeping the fork current

Upstream evdi regularly adds support for new kernels. If the DKMS build fails on a
newer kernel, rebase first:

```bash
cd ~/Work/evdi
git fetch upstream --tags
git rebase upstream/main          # drops 0d1c899 automatically once upstream has it
git push --force-with-lease origin main && git push origin --tags
cd ~/Work/evdi-dkms-grupertal-git && makepkg -si
```

The fork belongs to the owner's personal GitHub account (GruperTal). Push as that
account, not the work account. Commit identity:
`GruperTal <110677684+GruperTal@users.noreply.github.com>`.
