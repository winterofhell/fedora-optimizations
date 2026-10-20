# Fedora Linux 45 Performance & Gaming Optimization Guide

> Complete guide for optimizing Fedora 44 and 45 for performance, gaming and everyday use | by winterofhell

## Quick Navigation

| Setup & Kernel | System & Gaming | Resources |
|---|---|---|
| [System Information](#system-information) | [System Services Optimization](#system-services-optimization) | [Monitoring & Verification](#monitoring--verification) |
| [Fedora 45 Notes](#fedora-45-notes) | [Advanced System Tweaks](#advanced-system-tweaks) | [Troubleshooting](#troubleshooting) |
| [Initial Setup & Preparation](#initial-setup--preparation) | [Gaming Optimizations](#gaming-optimizations) | [Russian Translation](#русская-версия--russian-translation) |
| [Kernel Optimization](#kernel-optimization) | [Maintenance & Cleanup](#maintenance--cleanup) | |
| [GRUB Kernel Parameters](#grub-kernel-parameters) | [Graphics Driver Optimization](#graphics-driver-optimization) | |

## System Information

**Testing environment:**

- **Period:** October 14, 2024 - October 20, 2026
- **Distribution:** Fedora 44, Fedora 45 beta
- **Extra testing:** NVIDIA and AMD GPU systems
- These tweaks may work on other distros too, but i cant guarantee it. Always test everything yourself. About 80% of them work on Arch and NixOS (tested by myself) :)

**Hardware:**

| # | CPU | RAM | GPU | Storage |
|---|---|---|---|---|
| 1 | Ryzen 5 5500U | 20GB DDR4 | RX550X / Vega 7 | NVMe |
| 2 | Ryzen 5 5600 | 16GB DDR4 | GTX 1060 | SATA SSD |
| 3 | Ryzen 7 9850X3D | 32GB DDR5 | RX 9070 XT | NVMe |

-----

## Fedora 45 Notes

**What's inside:** Linux 7.2, GNOME 51, KDE Plasma 6.7, GCC 16.2, glibc 2.44, Python 3.15.

**Changes that matter for this guide:**

| Change | What it means for you |
|---|---|
| RPM signature checking enforced by default | Unsigned rpms are rejected. COPR and RPM Fusion packages are signed, so nothing changes for them. Self built rpms need signing or `--nogpgcheck` |
| DNF5 no longer switches package vendor automatically | so replacing Fedora packages with RPM Fusion / COPR ones must be explicit: use `dnf swap` ([Multimedia](#2-enable-rpm-fusion-repositories)) |
| `kernel.yama.ptrace_scope = 1` by default | Some trainers, memory scanners (scanmem, Cheat Engine under Wine) and debuggers can stop working... Fix below |
| kmscon replaces fbcon | The TTY (Ctrl+Alt+F3) looks different |
| Fedora repo configs moved from `/etc` to `/usr` | Your own overrides still go to `/etc`. If you edited a Fedora `.repo` file before, check where it lives now with `dnf repo info <repo-id>` |

**If something old breaks because of ptrace (reduces security):**

```bash
echo 'kernel.yama.ptrace_scope = 0' | sudo tee /etc/sysctl.d/10-ptrace.conf
sudo sysctl --system
```

**Upgrade from 44:**

```bash
sudo dnf upgrade --refresh
# Check that every COPR you use already has a fedora-45 build
dnf copr list
# If some COPR has no F45 build yet, disable it for the upgrade:
# sudo dnf copr disable owner/project
sudo dnf system-upgrade download --releasever=45
sudo dnf offline reboot
```

Nothing in this guide needs to be redone after the upgrade. Kernel params, sysctl files, udev rules and scx config carry over.

-----

## Initial Setup & Preparation

### 1. Minimal Installation

For best results, start with the **Fedora Everything / Minimal ISO**. Fewer packages and services in the background. Not required though!

### 2. Enable RPM Fusion Repositories

RPM Fusion gives you codecs, Steam and proprietary drivers:

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

**Multimedia (full codecs + hardware video decoding on AMD/Intel):**

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
sudo dnf update @multimedia --setopt="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin

# AMD / Intel (Mesa): H.264 / HEVC hardware decoding
sudo dnf swap mesa-va-drivers mesa-va-drivers-freeworld
sudo dnf swap mesa-va-drivers.i686 mesa-va-drivers-freeworld.i686
```

Official guides: [RPM Fusion Configuration](https://rpmfusion.org/Configuration), [Multimedia](https://rpmfusion.org/Howto/Multimedia)

### 3. Faster DNF

```bash
sudo dnf config-manager setopt max_parallel_downloads=10
```

### 4. SELinux Configuration (Optional)

**Security warning:** disabling SELinux reduces security but makes the system a little faster. Only do it if you understand the implications. Personally, i dont care about SELinux and always disable it.

Temporary (until reboot):

```bash
sudo setenforce 0
```

Permanent (reboot required):

```bash
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

If you **keep** SELinux and use the CachyOS kernel, allow it to load modules:

```bash
sudo setsebool -P domain_kernel_load_modules on
```

### 5. System Update

```bash
sudo dnf upgrade --refresh

# CPU microcode
# AMD:
sudo dnf install linux-firmware amd-ucode-firmware
# Intel:
sudo dnf install linux-firmware microcode_ctl
```

-----

## Kernel Optimization

### CachyOS Kernel Installation

The CachyOS kernel gives noticeable gains in responsiveness and gaming.

**Requirement:** CPU must support x86_64_v3.

```bash
/lib64/ld-linux-x86-64.so.2 --help | grep "(supported, searched)"
# No x86_64_v3 in the output = do NOT install the main kernel.
# Only x86_64_v2 = use the LTS kernel.
```

```bash
sudo dnf copr enable bieszczaders/kernel-cachyos
sudo dnf copr enable bieszczaders/kernel-cachyos-addons

# Main kernel
sudo dnf install kernel-cachyos kernel-cachyos-devel-matched

# x86_64_v2 only (older CPUs)
sudo dnf install kernel-cachyos-lts kernel-cachyos-lts-devel-matched
```

> **NVIDIA users:** use the GCC kernel above, not the `kernel-cachyos-lto` COPR. The LTO (Clang) build can break akmods.

### Make the CachyOS Kernel Default

```bash
# List kernel entries
sudo grubby --info=ALL | grep -E "^kernel|^index|^title"

# Current default
sudo grubby --default-kernel
sudo grubby --default-index

# Set CachyOS entry as default
sudo grubby --set-default-index=<INDEX_NUMBER>
# Example: sudo grubby --set-default-index=0

# Reboot and check, output should contain "cachyos"
uname -r
```

More info: [CachyOS COPR](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/), [copr-linux-cachyos on GitHub](https://github.com/CachyOS/copr-linux-cachyos)

-----

## System Services Optimization

### Ananicy-cpp

Automatically manages process priorities. It's packaged in the CachyOS addons COPR, no need to build it:

```bash
sudo dnf install ananicy-cpp cachyos-ananicy-rules
sudo systemctl enable --now ananicy-cpp
```

<details>
<summary>Build from source (if the COPR package doesn't work for you)</summary>

```bash
sudo dnf group install development-tools
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel gcc-c++

git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
sudo systemctl enable --now ananicy-cpp
```

</details>

### CachyOS Optimization Packages

The CachyOS team ships packages that apply many tweaks for you. Simpler and safer than setting dozens of values by hand :)

```bash
sudo dnf install cachyos-settings cachyos-ksm-settings scx-scheds scx-tools
```

`scx-tools` is required: it contains `scx_loader` and `scxctl`.

### SCX CPU Scheduler (scx_lavd)

One of the most impactful tweaks for responsiveness and frametimes. It replaces the default CPU scheduler with `scx_lavd`, which was built for gaming: it detects latency critical threads and prioritizes them over background work.

**Note:** this is an advanced tweak, it changes a core part of the system.

**Step 1: config**

```bash
sudo mkdir -p /etc/scx_loader
sudo tee /etc/scx_loader/config.toml <<'EOF'
default_sched = "scx_lavd"
default_mode = "Gaming"

[scheds.scx_lavd]
gaming_mode = ["--performance"]
EOF
```

**Step 2: irqbalance**

In my tests frametimes were smoother with scx_lavd and irqbalance off. It's not a hard requirement of scx, so if you see no difference, you can leave it on.

```bash
sudo systemctl disable --now irqbalance
```

**Step 3: enable**

```bash
sudo systemctl enable --now scx_loader
```

**Step 4: verify**

```bash
scxctl get
# or
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader \
  org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# Should show scx_lavd
```

Ananicy-cpp works fine alongside scx schedulers.

### Service Management

Disable ONLY what you really dont use:

| Service | Safe to disable when | Breaks |
|---|---|---|
| `bluetooth` | no BT devices | BT headphones, gamepads |
| `ModemManager` | no 3G/4G modem | mobile modems |
| `cups` | no printer | printing |
| `avahi-daemon` | no network printers / mDNS | `.local` discovery, network printers |
| `NetworkManager-wait-online` | almost always | nothing on desktops, boot gets faster |
| `geoclue` | you don't need auto location | auto timezone, night light by location |
| `smartd` | you don't care about disk health alerts | SMART warnings |
| `upower` | **desktop only** | battery info on laptops, some GNOME features |
| `chronyd` | **not recommended** | time sync. Clock drift breaks TLS, logins and some online games over time |

```bash
sudo systemctl disable --now bluetooth ModemManager cups avahi-daemon \
  NetworkManager-wait-online geoclue smartd
```

-----

## GRUB Kernel Parameters

### Configuration

Fedora uses BLS entries, so the safest way is `grubby`. It **adds** parameters and keeps the existing ones (`rhgb quiet`, `rd.luks`, `resume` etc.):

```bash
sudo grubby --update-kernel=ALL --args="amdgpu.ppfeaturemask=0xffffffff split_lock_detect=off amd_pstate=active page_alloc.shuffle=1 pci=pcie_bus_perf pcie_aspm.policy=performance usbcore.autosuspend=-1 nowatchdog nmi_watchdog=0 zswap.enabled=0"
```

Remove a parameter later:

```bash
sudo grubby --update-kernel=ALL --remove-args="zswap.enabled=0"
```

Manual way: add the parameters **to the end** of the existing `GRUB_CMDLINE_LINUX` in `/etc/default/grub` (don't replace the line), then `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`.

Check after reboot: `cat /proc/cmdline`

### Parameter Breakdown

| Parameter | Effect | Notes |
|---|---|---|
| `amd_pstate=active` | EPP driver, fast clock boosting | AMD Zen 2+ with CPPC. Already the default on modern kernels, this just pins it. Intel users remove |
| `amdgpu.ppfeaturemask=0xffffffff` | Unlocks voltage and OC limits | Needed for LACT / CoreCtrl. AMD gpu only |
| `split_lock_detect=off` | Prevents slowdowns and stutters in games that use split locks | Safe for everyone |
| `nowatchdog` + `nmi_watchdog=0` | Disables watchdog timers, less micro-stutter | Safe for everyone |
| `zswap.enabled=0` | Disables zswap | Fedora uses zram anyway. Remove if you rely on disk swap with 16GB or less |
| `pci=pcie_bus_perf` | Max PCIe payload size for GPU/NVMe | Rarely some boards lose devices with it. Remove if something disappears |
| `pcie_aspm.policy=performance` | Disables PCIe power saving, fixes idle latency spikes | Better than `pcie_aspm=off`. More power draw on laptops |
| `usbcore.autosuspend=-1` | USB devices never sleep, no input wake-up lag | Safe for everyone |
| `page_alloc.shuffle=1` | Randomizes the page allocator free lists | Mostly a security / memory-side cache feature. Gaming gains are small, keep it if you want |

### Experimental, use at your own risk

| Parameter | Benefit | Warning |
|---|---|---|
| `mitigations=off` | ~3% FPS on older CPUs | Disables cpu security patches. On Zen 4 / Zen 5 (Ryzen 7000/9000) users report worse 1% lows, so i dont recommend it there. Makes sense mainly for Zen 1/2/3 and Intel 9th gen or older |

Alternative to `split_lock_detect=off` without touching the cmdline (kernel 6.2+):

```bash
echo 'kernel.split_lock_mitigate = 0' | sudo tee /etc/sysctl.d/99-splitlock.conf
sudo sysctl --system
```

-----

## Advanced System Tweaks

### Memory Management

systemd-oomd is already enabled on Fedora Workstation. On some systems it kills games under memory pressure. Check and decide:

```bash
systemctl status systemd-oomd
# to turn off:
# sudo systemctl disable --now systemd-oomd
```

### zram

Fedora already uses zram. You can make it bigger and use zstd:

```bash
sudo tee /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = ram
compression-algorithm = zstd
EOF
sudo systemctl daemon-reload
sudo systemctl restart systemd-zram-setup@zram0
zramctl
```

### sysctl

Use a file in `/etc/sysctl.d/` instead of appending to `/etc/sysctl.conf`:

```bash
sudo tee /etc/sysctl.d/99-gaming.conf <<'EOF'
# zram is fast, so a HIGH swappiness is correct here (not low)
vm.swappiness = 150
# keep dentry/inode caches longer
vm.vfs_cache_pressure = 50
# Fedora already sets 1048576 since F39, this raises it further
vm.max_map_count = 16777216
EOF
sudo sysctl --system
```

> `cachyos-settings` also ships sysctl values. Files are applied in name order, so `99-gaming.conf` wins.

`vm.max_map_count` too low causes crashes in games like Hogwarts Legacy, CS2, The Finals, DayZ, Star Citizen. Check: `sysctl vm.max_map_count`

### Storage

TRIM timer is enabled on Fedora by default. Check and run manually:

```bash
systemctl status fstrim.timer
sudo fstrim -v /
```

### CPU Scaling

AMD:

```bash
echo active | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

Intel: `active` (default) with EPP is usually the best choice. Switch to `passive` only if you want the `schedutil` governor:

```bash
echo passive | sudo tee /sys/devices/system/cpu/intel_pstate/status
```

### File Descriptor Limit

In `/etc/security/limits.conf` (replace `yourusername`):

```bash
yourusername hard nofile 1048576
yourusername soft nofile 1048576
```

Mostly matters for old esync. With NTSync it's not critical.

### IRQ Balance (Intel iGPU)

If you see performance issues on Intel integrated graphics:

```bash
sudo systemctl status irqbalance
sudo systemctl disable --now irqbalance
```

### I/O Scheduler

The `elevator=` kernel parameter no longer works. Use udev:

```bash
sudo tee /etc/udev/rules.d/60-ioschedulers.rules <<'EOF'
# HDD
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
# SATA SSD / eMMC
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
# NVMe: own queue management, no scheduler needed
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
grep "" /sys/block/*/queue/scheduler
```

Thanks to netarchy for mentioning this method! tho `cachyos-settings` may already ship a similar rule, check with `ls /usr/lib/udev/rules.d | grep -i sched`.

-----

## Gaming Optimizations

### Steam

```bash
sudo dnf install steam
```

**Background shader processing:** speeds up Vulkan shader pre-caching. Close Steam, then:

```bash
echo 'unShaderBackgroundProcessingThreads 8' >> ~/.steam/steam/steam_dev.cfg
```

Set the number to roughly half your CPU threads.

### GameMode

```bash
sudo dnf install gamemode
gamemoded -t
```

Use `gamemoderun %command%` in Steam launch options.

### Proton versions

- **Proton 11** (Valve, stable since July 2026): based on Wine 11, uses NTSync automatically when the kernel has it.
- **Proton-CachyOS**: extra patches, fsr4 upgrade built in. Since 11.0-20260702 it puts fsr 4.1.1 into every prefix automatically on RDNA 2-4, `PROTON_FSR4_RDNA3_UPGRADE` is gone.

Install custom Protons with ProtonPlus:

```bash
flatpak install flathub com.vysp3r.ProtonPlus
```

### Windows Games Compatibility

**PortProton** is my favorite Proton launcher, I use it instead of Lutris / Bottles:

```bash
sudo dnf copr enable boria138/portproton
sudo dnf install portproton
```

### NTSync

Mainline since kernel 6.14, Fedora 44/45 and CachyOS kernels have it. Check:

```bash
ls -l /dev/ntsync
# missing? load it and make it permanent:
sudo modprobe ntsync
echo ntsync | sudo tee /etc/modules-load.d/ntsync.conf
```

-----

## Launch Command Reference

### Which command should I use?

| My situation | Command |
|---|---|
| Game has native HDR (Cyberpunk, Elden Ring...) | HDR command |
| Game has no HDR, i dont want fake HDR | No HDR command |
| Game has no HDR, i want Auto HDR (like Windows) | SDR to HDR command |

### Environment Variables & Wrappers

| Variable | What it does |
|---|---|
| `LD_PRELOAD=""` | Fixes Steam micro-stutters after ~25-30 minutes of play. Side effect: Steam overlay won't show |
| `PROTON_USE_NTSYNC=1` | Forces NTSync in GE / older CachyOS Protons. Proton 11 does it automatically, the variable doesnt hurt |
| `AMD_USERQ=1` | **RDNA 3/4.** User queues: work goes straight to GPU hardware queues, less kernel overhead. Needs Mesa 25.2+ and a kernel with user queues enabled (add `amdgpu.user_queue=1` if `dmesg` shows nothing about it). Mainly affects OpenGL (radeonsi), Vulkan gains are not guaranteed |
| `DXVK_HDR=1` | Lets DXVK games output HDR |
| `MANGOHUD_CONFIG="fps_limit=277,no_display"` | MangoHud frame limiter running silently, no overlay |
| `fps_limit=277` | Low latency cap: monitor max Hz minus 3 (280 Hz -> 277). Keeps you inside the VRR range without V-Sync backpressure |
| `mangohud` | Injects the limiter. Put it **after** `--` so it attaches to the game, not to gamescope |
| `gamemoderun` | Feral GameMode for the session |

### Gamescope Arguments

| Argument | What it does |
|---|---|
| `gamescope` | Valve's micro-compositor. Isolates the game, often fixes alt+tab crashes, handles HDR and resolution cleanly |
| `-W 2560 -H 1440` | Gamescope **output** resolution. Set to your monitor's native resolution |
| `-w 1920 -h 1080` | Optional: resolution the **game** renders at. Defaults to `-W/-H`. Use with upscaling |
| `-r 280` | Frame rate limit for the game inside gamescope. Set to your monitor max Hz. If MangoHud's limit is lower, MangoHud wins |
| `--hdr-enabled` | Turns on the HDR output pipeline |
| `--force-grab-cursor` | Keeps the mouse inside the game on multi-monitor setups |
| `--adaptive-sync` | VRR (FreeSync / G-Sync) inside the container |
| `-F fsr --sharpness 2` | Optional: FSR 1 upscaling from `-w/-h` to `-W/-H`. Sharpness 0 = sharpest, 20 = softest. Without `-F` sharpness does nothing |
| `-f` | Fullscreen |
| `--` | End of gamescope flags, the game command starts next |
| `%command%` | Steam replaces this with the game executable |

### Auto SDR to HDR Arguments

Used together with `--hdr-enabled`:

| Argument | What it does |
|---|---|
| `--hdr-itm-enable` | Inverse tone mapping, the actual "Auto HDR" switch |
| `--hdr-itm-sdr-nits 100` | Brightness the SDR source is treated as. 100 is a good start |
| `--hdr-itm-target-nits 1000` | Peak brightness target. Set to your monitor's real peak |
| `--hdr-sdr-content-nits 203` | Paper white for UI and normal textures. 203 is the standard: higher blinds you in menus, lower looks dim |

### Commands

**Native HDR games:**

```bash
LD_PRELOAD="" PROTON_USE_NTSYNC=1 AMD_USERQ=1 DXVK_HDR=1 MANGOHUD_CONFIG="fps_limit=277,no_display" gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --force-grab-cursor --adaptive-sync -f -- mangohud %command%
```

**No HDR:**

```bash
LD_PRELOAD="" PROTON_USE_NTSYNC=1 AMD_USERQ=1 MANGOHUD_CONFIG="fps_limit=277,no_display" gamemoderun gamescope -W 2560 -H 1440 -r 280 --force-grab-cursor --adaptive-sync -f -- mangohud %command%
```

**No HDR + Auto SDR to HDR (like Windows Auto HDR):**

```bash
LD_PRELOAD="" PROTON_USE_NTSYNC=1 AMD_USERQ=1 DXVK_HDR=1 MANGOHUD_CONFIG="fps_limit=277,no_display" gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --hdr-itm-enable --hdr-itm-sdr-nits 100 --hdr-itm-target-nits 1000 --hdr-sdr-content-nits 203 --force-grab-cursor --adaptive-sync -f -- mangohud %command%
```

> Adjust `-W`, `-H`, `-r` and `fps_limit` to your monitor. NVIDIA users: remove `AMD_USERQ=1`.

**FSR4 / DLSS upgrade (Proton-CachyOS / GE):**

```bash
# RDNA 4 (also picks the DLL version on newer Proton-CachyOS)
PROTON_FSR4_UPGRADE=1 %command%
# NVIDIA: newest DLSS DLLs
PROTON_DLSS_UPGRADE=1 %command%
```

-----

## Maintenance & Cleanup

```bash
# DNF cache
sudo dnf clean all

# Journal: keep 7 days or 100MB
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=100M

# Old kernels: keep the last 3
sudo dnf remove $(dnf repoquery --installonly --latest-limit=-3 -q)
```

### Maintenance Script

```bash
#!/bin/bash
# Save as ~/maintenance.sh, then: chmod +x ~/maintenance.sh
echo "Running system maintenance..."
sudo dnf upgrade --refresh
sudo dnf clean all
sudo journalctl --vacuum-time=7d
sudo fstrim -v /
flatpak update -y
flatpak uninstall --unused -y
echo "Maintenance complete!"
```

-----

## Desktop Environment Recommendations

### Lightweight Alternatives

If you installed from the minimal ISO:

- **Sway** - Wayland tiling compositor (~900MB idle for me)
- **i3** - X11 tiling WM (~750MB idle)
- **Hyprland** - Wayland compositor with animations (~1gb idle)
- **XFCE** - lightweight classic desktop
- **LXQt** - Qt-based lightweight desktop

**KDE Plasma** is an official Fedora edition alongside Workstation (GNOME), with good integration out of the box.

### Power Profile (Desktop)

Fedora uses tuned + tuned-ppd. On a desktop just switch to performance:

```bash
powerprofilesctl set performance
powerprofilesctl get
```

### Laptop Power Management (TLP)

TLP conflicts with tuned / tuned-ppd. Remove them first, otherwise both fight over the same settings:

```bash
sudo dnf remove tuned tuned-ppd
sudo dnf install tlp tlp-rdw powertop
sudo systemctl enable --now tlp
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
```

Performance on AC in `/etc/tlp.conf` (`TLP_DEFAULT_MODE` only accepts `AC` or `BAT`):

```ini
TLP_DEFAULT_MODE=AC
CPU_ENERGY_PERF_POLICY_ON_AC=performance
PLATFORM_PROFILE_ON_AC=performance
```

Apply: `sudo tlp start`. Check: `sudo tlp-stat -s`

### Flatpak Steam

```bash
flatpak install flathub com.valvesoftware.Steam
# Give access to a game library on another drive:
flatpak override --user --filesystem=/mnt/games com.valvesoftware.Steam
```

### GNOME Tweaks

```bash
sudo dnf install gnome-tweaks gnome-extensions-app

# Disable animations
gsettings set org.gnome.desktop.interface enable-animations false

# Workspaces on all monitors, not only the primary
gsettings set org.gnome.mutter workspaces-only-on-primary false
```

-----

## Graphics Driver Optimization

<details>
<summary>🔴 AMD Graphics Optimization</summary>

### Drivers

Mesa is installed by default. Make sure you have the Vulkan driver and RPM Fusion VA-API (see [RPM Fusion](#2-enable-rpm-fusion-repositories)):

```bash
sudo dnf install mesa-vulkan-drivers mesa-vulkan-drivers.i686

# ROCm for compute (optional)
sudo dnf install rocm-opencl rocm-smi
```

> `mesa-vdpau-drivers` is gone: Mesa removed VDPAU in 25.3, Fedora 44+ doesn't ship it. Use VA-API.

### Environment Tweaks

Optional! Some of these can lower FPS or make games unstable. Test after each change.

```bash
sudo mkdir -p /etc/environment.d
sudoedit /etc/environment.d/99-amd-gaming.conf
```

```ini
# OpenGL threading. Great for Minecraft, can lower FPS in other games
mesa_glthread=true

# VA-API driver
LIBVA_DRIVER_NAME=radeonsi

# Only matters if AMDVLK is installed. AMDVLK is no longer developed, RADV is the default
# AMD_VULKAN_ICD=RADV

# Not needed anymore: gpl, nggc, rt and sam are default in current Mesa
# RADV_PERFTEST=gpl,nggc,sam,rt
```

**Resizable BAR:** there's no env variable for it, enable it in BIOS (Above 4G Decoding + ReBAR). Check:

```bash
sudo dmesg | grep -i "BAR="
```

### GPU Performance Level

Pins GPU clocks to the max state. More idle power and heat. Don't use together with LACT, LACT does the same thing better.

```bash
# Now:
for f in /sys/class/drm/card*/device/power_dpm_force_performance_level; do echo high | sudo tee "$f"; done
```

Persistent service:

```bash
sudo tee /etc/systemd/system/amd-gpu-performance.service <<'EOF'
[Unit]
Description=AMD GPU Performance Mode
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'for f in /sys/class/drm/card*/device/power_dpm_force_performance_level; do echo high > "$f"; done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now amd-gpu-performance.service
```

### AMD User Queues (RDNA 3/4)

Add `AMD_USERQ=1` to Steam launch options. Details in [Launch Command Reference](#launch-command-reference).

### FSR4, Anti-Lag 2 and HDR: my notes (rx9070XT)

Tested on rx9070xt, should be the same for rx 7000 and 9000. Use these only when a game needs them, not by default.

| Problem | Fix |
|---|---|
| FSR4 doesnt show up in the game menu | `PROTON_FSR4_UPGRADE=1 PROTON_MLFG_UPGRADE=1` |
| OptiScaler picks XeLL instead of Anti-Lag 2 | `LOW_LATENCY_LAYER=1` |
| HDR option is greyed out or missing | `PROTON_ENABLE_WAYLAND=1 DXVK_HDR=1` |

**FSR4.** Most games pick it up on their own. Some still don't show FSR4 until you add both variables:

```bash
PROTON_FSR4_UPGRADE=1 PROTON_MLFG_UPGRADE=1 %command%
```

**Anti-Lag 2 through OptiScaler.** OptiScaler sets XeLL by default. If you want Anti-Lag 2, add:

```bash
LOW_LATENCY_LAYER=1 %command%
```

**HDR.** Some games don't let you turn HDR on at all without these:

```bash
PROTON_ENABLE_WAYLAND=1 DXVK_HDR=1 %command%
```

All of them can be combined in one line with the rest of your launch options.

### LACT

Like AMD Adrenalin on Windows: voltage offset, memory clock, power limit, fan curve. Works thanks to `amdgpu.ppfeaturemask=0xffffffff` from GRUB.

```bash
sudo dnf copr enable ilyaz/LACT
sudo dnf install lact     # alternatives: lact-headless, lact-libadwaita
sudo systemctl enable --now lactd
```

rx9070xt starting point: Performance Level: Manual, -60mV offset, memory clock locked at 2764MHz.
- Overclock: Power Limit 355W, about +4-6% FPS.
- Undervolt: Power Limit 250-260W, about -3% FPS.

</details>

<details>
<summary>🟢 NVIDIA Graphics Optimization</summary>

## NVIDIA Graphics Optimization for Fedora

### Which driver for which GPU

| GPU | Driver branch | Package |
|---|---|---|
| RTX 20/30/40/50, GTX 16 (Turing+) | 595+ (default on F44/F45, open kernel modules) | `akmod-nvidia` |
| GTX 750/900/10 (Maxwell, Pascal), Titan V | 580xx legacy | `akmod-nvidia-580xx` |

NVIDIA 590+ dropped Maxwell, Pascal and Volta. These cards now only get quarterly security updates on the 580 legacy branch. If you upgraded a GTX 10 system to Fedora 44 and the driver stopped loading, this is why.

### Installation (RPM Fusion)

```bash
# RPM Fusion must be enabled (see Initial Setup)
sudo dnf upgrade --refresh
```

**Turing and newer:**

```bash
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda xorg-x11-drv-nvidia-libs.i686
```

**Maxwell / Pascal (GTX 750-1080 Ti):**

```bash
sudo dnf install akmod-nvidia-580xx xorg-x11-drv-nvidia-580xx-cuda xorg-x11-drv-nvidia-580xx-libs.i686
```

Wait ~5 minutes for akmods to build the module, then check and reboot:

```bash
modinfo -F version nvidia
sudo reboot
```

> Don't install `akmod-nvidia-open` for a normal setup. On RPM Fusion `akmod-nvidia` already uses the open kernel modules on Turing+.

**Secure Boot:** sign the module once, then enroll the key on next boot:

```bash
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
sudo reboot
```

Suspend / resume services:

```bash
sudo systemctl enable nvidia-suspend nvidia-resume nvidia-hibernate
```

#### Verification

```bash
nvidia-smi
nvidia-smi -q | grep "CUDA Version"
echo $XDG_SESSION_TYPE      # should be wayland
cat /sys/module/nvidia_drm/parameters/modeset   # should be Y
```

#### Wayland modeset (only if needed)

With RPM Fusion, `akmod-nvidia` enables modeset itself. Do this only if Wayland doesn't start or you used the `.run` installer:

```bash
echo 'options nvidia_drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-drm-modeset.conf
sudo dracut --force
sudo reboot
```

-----

### Environment Variables

> **Warning:** don't put everything into `/etc/environment` blindly. Global NVIDIA variables can break Wayland login, cause black screens, stutter or input lag depending on GPU, driver and desktop. Test per game in Steam launch options first.

`/etc/environment`:

```bash
# GTX 10 and older ONLY. On RTX 20+ these two cause system-wide input lag and stutter
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia

# CPU-GPU parallelism. Can cause a black screen on some RTX cards, better per game
__GL_THREADED_OPTIMIZATIONS=1

# Shader cache. Default path is ~/.cache/nvidia (survives reboots).
# Don't move it to /tmp: it gets wiped on reboot and you get stutter again
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_SIZE=10737418240
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1

# No VSync in OpenGL (lower input lag)
__GL_SYNC_TO_VBLANK=0

# DLSS and other NVAPI features in Proton
PROTON_ENABLE_NVAPI=1

# Fixes Steam stutter after 25-30+ minutes
LD_PRELOAD=""

# Native Wayland in Proton. Better per game: some titles lose exclusive fullscreen or break
# PROTON_ENABLE_WAYLAND=1

# wlroots compositors only (Sway, Hyprland), not needed on GNOME/KDE.
# With driver 595+ and explicit sync support you may not need them, test without first
# WLR_DRM_NO_ATOMIC=1
# WLR_NO_HARDWARE_CURSORS=1
```

If you got a black screen after setting `__GL_THREADED_OPTIMIZATIONS=1` globally:
1. Ctrl+Alt+F3, log in
2. `sudoedit /etc/environment`, remove the line
3. `sudo reboot`

Thanks to @lemonadeforlife for this one.

### Kernel Module Parameters (if you know what you're doing)

Test each option on your hardware. `/etc/modprobe.d/nvidia-power-management.conf`:

```bash
# Runtime power management (laptops)
options nvidia NVreg_DynamicPowerManagement=0x02
# Page Attribute Table
options nvidia NVreg_UsePageAttributeTable=1
# Resizable BAR (RTX 30/40/50)
options nvidia NVreg_EnableResizableBar=1
# Keep VRAM over suspend
options nvidia NVreg_PreserveVideoMemoryAllocations=1
# Stream memory operations
options nvidia NVreg_EnableStreamMemOPs=1
```

Then `sudo dracut --force` and reboot.

### GNOME Wayland

Fedora 44 (GNOME 50) and 45 (GNOME 51): VRR is stable. Enable it in **Settings > Displays > Variable Refresh Rate**. The old gsettings command isn't needed anymore, and it **overwrites** other experimental features (like fractional scaling), so avoid it.

```bash
gsettings set org.gnome.desktop.interface enable-animations false
```

### KDE Plasma Wayland (6.x)

VRR: **System Settings > Display & Monitor > Adaptive Sync**.

The old `kwinrc` keys `LatencyPolicy` and `RenderTimeEstimator` were removed in Plasma 6 (KWin now measures render time itself), so these commands do nothing anymore.

-----

### Steam Launch Options

Native Linux games:

```bash
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 __GL_SYNC_TO_VBLANK=0 %command%
```

Proton games:

```bash
# Standard
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Maximum performance
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 PROTON_DLSS_UPGRADE=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%
```

`PROTON_DLSS_UPGRADE=1` works in Proton-CachyOS / GE.

### Lutris

```bash
sudo dnf install lutris wine
```

System Options > Environment variables:

```
__GL_THREADED_OPTIMIZATIONS=1
__GL_SHADER_DISK_CACHE=1
PROTON_ENABLE_NVAPI=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
```

### GameMode

```bash
sudo dnf install gamemode
sudoedit /etc/gamemode.ini
```

```ini
[general]
renice=10
ioprio=0

[gpu]
# Warning: tries to overclock the GPU through nvidia-settings. Test carefully
apply_gpu_optimisations=accept-responsibility
gpu_device=0
```

-----

### Advanced

#### VRR check

```bash
sudo dnf install drm_info
drm_info | grep -i vrr
```

#### HDR (experimental)

```bash
# Unverified parameter, not in official NVIDIA docs. Test before using
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf
```

#### Monitoring script

```bash
sudo dnf install nvtop mangohud goverlay

sudo tee /usr/local/bin/nvidia-monitor.sh <<'EOF'
#!/bin/bash
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
nvidia-smi dmon -s pucvmet
EOF
sudo chmod +x /usr/local/bin/nvidia-monitor.sh
```

#### Laptop runtime PM (if you know what you're doing)

```bash
echo 'options nvidia NVreg_DynamicPowerManagement=0x02' | sudo tee /etc/modprobe.d/nvidia-power.conf

sudo tee /etc/udev/rules.d/80-nvidia-pm.rules <<'EOF'
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
EOF
```

-----

### NVIDIA Troubleshooting

**Black screen / Wayland doesn't start:**

1. Ctrl+Alt+F3 (or GRUB > previous kernel)
2. Comment out `__GL_THREADED_OPTIMIZATIONS=1` in `/etc/environment`
3. Check modeset:

```bash
cat /sys/module/nvidia_drm/parameters/modeset   # Y
sudo dracut --force
sudo reboot
```

**GTX 10 / 900 driver doesn't load after an update:** you're on 595, switch to 580xx (see the table above).

**Low performance:**

```bash
nvidia-smi dmon -s pucvmet -c 10
nvidia-smi --query-gpu=power.limit,power.draw --format=csv
watch -n 1 'nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv'
```

In games power draw and clocks should get close to max.

**Tearing:** enable VRR in display settings, disable VSync in game, add `__GL_SYNC_TO_VBLANK=0 %command%`.

**High idle power (laptops):**

```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status   # suspended when idle
nvidia-smi --query-gpu=power.draw --format=csv --loop=1
```

-----

### MangoHud config

```bash
mkdir -p ~/.config/MangoHud
tee ~/.config/MangoHud/MangoHud.conf <<'EOF'
gpu_stats
cpu_stats
gpu_temp
cpu_temp
fps
frametime
frame_timing
vram
ram
position=top-left
font_size=22
background_alpha=0.6
EOF
```

### Resources

- [RPM Fusion NVIDIA Guide](https://rpmfusion.org/Howto/NVIDIA)
- [NVIDIA Linux Driver Docs](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
- [MangoHud](https://github.com/flightlessmango/MangoHud)

</details>

-----

## Monitoring & Verification

```bash
sudo dnf install btop nvtop inxi
inxi -Fxz          # full system summary
systemd-analyze blame | head -20   # slowest boot services
```

### Benchmarks

```bash
sudo dnf install glmark2 sysbench stress-ng
```

Unigine Superposition: download from [unigine.com](https://benchmark.unigine.com/superposition).

-----

## Troubleshooting

**1. Boot issues after kernel parameters:**

- Pick the previous kernel in the GRUB menu (hold Shift / press Esc during boot)
- `sudo grubby --update-kernel=ALL --remove-args="<param>"`

**2. Graphics issues:**

```bash
lspci -k | grep -A 2 -E "(VGA|3D)"
lsmod | grep -E "(amdgpu|nvidia|i915|xe)"
```

**3. Performance regression:**

- Resources: `btop`, `iotop`
- Thermal throttling: `watch sensors`
- Failed services: `systemctl list-units --failed`
- Turn off one tweak at a time and compare

### Recovery Commands

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
# if you re-enable SELinux:
sudo touch /.autorelabel && sudo reboot
sudo dnf check
sudo rpm -Va
```

-----

## Expected Performance Gains

Based on my testing:

| Area | Gain |
|---|---|
| Boot time | 10-20% faster |
| Gaming | +5-15% FPS |
| Responsiveness | noticeably lower input lag |
| Idle RAM | 5-15% less |
| Storage | better SSD performance with TRIM |

-----

## Русская версия | (Russian Translation)

<details>
<summary>Нажмите для просмотра русской версии</summary>

# Гайд по оптимизации производительности и гейминга Fedora Linux 44 и 45

> Полный гайд по оптимизации Fedora 44 и 45 для производительности, игр и повседневного использования | от winterofhell

## Быстрая навигация

| Настройка и ядро | Система и гейминг | Ресурсы |
|---|---|---|
| [Информация о системе](#информация-о-системе) | [Оптимизация системных сервисов](#оптимизация-системных-сервисов) | [Мониторинг и проверка](#мониторинг-и-проверка) |
| [Fedora 45: что нового](#fedora-45-что-нового) | [Продвинутые системные твики](#продвинутые-системные-твики) | [Решение проблем](#решение-проблем) |
| [Начальная настройка](#начальная-настройка-и-подготовка) | [Оптимизация для гейминга](#оптимизация-для-гейминга) | [Английская версия](#fedora-linux-44--45-performance--gaming-optimization-guide) |
| [Оптимизация ядра](#оптимизация-ядра) | [Обслуживание и очистка](#обслуживание-и-очистка) | |
| [Параметры ядра GRUB](#параметры-ядра-grub) | [Оптимизация графических драйверов](#оптимизация-графических-драйверов) | |

## Информация о системе

**Тестовое окружение:**

- **Период:** 14 октября 2024 - октябрь 2026
- **Дистрибутив:** Fedora 44, добавлены заметки по Fedora 45
- **Доп. тесты:** системы с NVIDIA и AMD
- Твики могут работать и на других дистрибутивах, но гарантий нет. Всё проверяйте сами. Примерно 90% работают на Arch и NixOS, проверял лично :)

**Железо:**

| # | CPU | RAM | GPU | Накопитель |
|---|---|---|---|---|
| 1 | Ryzen 5 5500U | 20GB DDR4 | RX550X / Vega 7 | NVMe |
| 2 | Ryzen 5 5600 | 16GB DDR4 | GTX 1060 | SATA SSD |
| 3 | Ryzen 5 7500F / Ryzen 7 9850X3D | 32GB DDR5 | RX 9070 XT | NVMe |

-----

## Fedora 45: что нового

Релиз планировали на 20 октября 2026, но первый Go/No-Go закончился NO-GO. Текущая цель **27 октября**, запасная дата **3 ноября**. Fedora 43 перестанет поддерживаться примерно через 4 недели после выхода 45.

**Внутри:** Linux 7.2, GNOME 51, KDE Plasma 6.7, GCC 16.2, glibc 2.44, Python 3.15.

**Что важно для этого гайда:**

| Изменение | Что это значит |
|---|---|
| Проверка подписей RPM включена по умолчанию | Неподписанные RPM не ставятся. Пакеты из COPR и RPM Fusion подписаны, для них ничего не меняется. Свои самосборные RPM придётся подписывать или ставить с `--nogpgcheck` |
| DNF5 больше сам не меняет вендора пакета | Замену пакетов Fedora на пакеты из RPM Fusion / COPR делайте явно через `dnf swap` (см. [Мультимедиа](#2-включение-rpm-fusion)) |
| `kernel.yama.ptrace_scope = 1` по умолчанию | Могут перестать работать трейнеры, сканеры памяти (scanmem, Cheat Engine под Wine) и отладчики. Фикс ниже |
| kmscon вместо fbcon | TTY (Ctrl+Alt+F3) выглядит по-другому. Шаги восстановления из гайда работают так же |
| Конфиги репозиториев Fedora переехали из `/etc` в `/usr` | Ваши правки по-прежнему идут в `/etc`. Если раньше правили `.repo` от Fedora, проверьте, где он теперь: `dnf repo info <repo-id>` |

**Если что-то старое сломалось из-за ptrace (снижает безопасность):**

```bash
echo 'kernel.yama.ptrace_scope = 0' | sudo tee /etc/sysctl.d/10-ptrace.conf
sudo sysctl --system
```

**Обновление с 44:**

```bash
sudo dnf upgrade --refresh
# Проверьте, что у всех ваших COPR уже есть сборки под fedora-45
dnf copr list
# Если у какого-то COPR сборки под F45 ещё нет, отключите его на время:
# sudo dnf copr disable owner/project
sudo dnf system-upgrade download --releasever=45
sudo dnf offline reboot
```

После апгрейда ничего из гайда переделывать не нужно. Параметры ядра, sysctl, udev-правила и конфиг scx переезжают как есть.

-----

## Начальная настройка и подготовка

### 1. Минимальная установка

Лучший старт: **Fedora Everything / Minimal ISO**. Меньше лишних пакетов и сервисов в фоне. Не обязательно.

### 2. Включение RPM Fusion

RPM Fusion даёт кодеки, Steam и проприетарные драйверы:

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

**Мультимедиа (полные кодеки + аппаратное декодирование видео на AMD/Intel):**

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
sudo dnf update @multimedia --setopt="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin

# AMD / Intel (Mesa): аппаратное декодирование H.264 / HEVC
sudo dnf swap mesa-va-drivers mesa-va-drivers-freeworld
sudo dnf swap mesa-va-drivers.i686 mesa-va-drivers-freeworld.i686
```

Официальные гайды: [RPM Fusion Configuration](https://rpmfusion.org/Configuration), [Multimedia](https://rpmfusion.org/Howto/Multimedia)

### 3. Быстрее DNF

```bash
sudo dnf config-manager setopt max_parallel_downloads=10
```

### 4. SELinux (опционально)

**Предупреждение:** отключение SELinux снижает безопасность, но чуть ускоряет систему. Делайте, только если понимаете последствия. Мне на SELinux всё равно, я его всегда отключаю.

Временно, до перезагрузки:

```bash
sudo setenforce 0
```

Навсегда, нужна перезагрузка:

```bash
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

Если SELinux **оставляете** и ставите ядро CachyOS, разрешите загрузку модулей:

```bash
sudo setsebool -P domain_kernel_load_modules on
```

### 5. Обновление системы

```bash
sudo dnf upgrade --refresh

# Микрокод CPU
# AMD:
sudo dnf install linux-firmware amd-ucode-firmware
# Intel:
sudo dnf install linux-firmware microcode_ctl
```

-----

## Оптимизация ядра

### Ядро CachyOS

Даёт заметный прирост отзывчивости и производительности в играх.

**Требование:** CPU с поддержкой x86_64_v3.

```bash
/lib64/ld-linux-x86-64.so.2 --help | grep "(supported, searched)"
# Нет x86_64_v3 в выводе = основное ядро НЕ ставить.
# Только x86_64_v2 = ставьте LTS.
```

```bash
sudo dnf copr enable bieszczaders/kernel-cachyos
sudo dnf copr enable bieszczaders/kernel-cachyos-addons

# Основное ядро
sudo dnf install kernel-cachyos kernel-cachyos-devel-matched

# Только для x86_64_v2 (старые CPU)
sudo dnf install kernel-cachyos-lts kernel-cachyos-lts-devel-matched
```

> **Владельцам NVIDIA:** используйте GCC-ядро выше, а не COPR `kernel-cachyos-lto`. LTO-сборка (Clang) может ломать akmods.

### Ядро CachyOS по умолчанию

```bash
# Список ядер
sudo grubby --info=ALL | grep -E "^kernel|^index|^title"

# Текущее по умолчанию
sudo grubby --default-kernel
sudo grubby --default-index

# Выставить CachyOS
sudo grubby --set-default-index=<INDEX_NUMBER>
# Пример: sudo grubby --set-default-index=0

# Перезагрузка и проверка, в выводе должно быть "cachyos"
uname -r
```

Подробнее: [CachyOS COPR](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/), [copr-linux-cachyos на GitHub](https://github.com/CachyOS/copr-linux-cachyos)

-----

## Оптимизация системных сервисов

### Ananicy-cpp

Автоматически управляет приоритетами процессов. Уже собран в addons COPR от CachyOS, компилировать не нужно:

```bash
sudo dnf install ananicy-cpp cachyos-ananicy-rules
sudo systemctl enable --now ananicy-cpp
```

<details>
<summary>Сборка из исходников (если пакет из COPR не подошёл)</summary>

```bash
sudo dnf group install development-tools
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel gcc-c++

git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
sudo systemctl enable --now ananicy-cpp
```

</details>

### Пакеты оптимизации CachyOS

Команда CachyOS собрала пакеты, которые применяют кучу твиков за вас. Проще и безопаснее, чем руками крутить десятки параметров :)

```bash
sudo dnf install cachyos-settings cachyos-ksm-settings scx-scheds scx-tools
```

`scx-tools` обязателен: в нём `scx_loader` и `scxctl`.

### Планировщик CPU SCX (scx_lavd)

Один из самых сильных твиков для отзывчивости и frametime. Заменяет стандартный планировщик на `scx_lavd`, который делали под игры: он находит потоки, критичные к задержке, и ставит их выше фоновых задач.

**Важно:** продвинутый твик, меняет базовую часть системы.

**Шаг 1: конфиг**

```bash
sudo mkdir -p /etc/scx_loader
sudo tee /etc/scx_loader/config.toml <<'EOF'
default_sched = "scx_lavd"
default_mode = "Gaming"

[scheds.scx_lavd]
gaming_mode = ["--performance"]
EOF
```

**Шаг 2: irqbalance**

В моих тестах frametime ровнее со scx_lavd и выключенным irqbalance. Это не жёсткое требование scx: если разницы не видите, можно оставить.

```bash
sudo systemctl disable --now irqbalance
```

**Шаг 3: включить**

```bash
sudo systemctl enable --now scx_loader
```

**Шаг 4: проверить**

```bash
scxctl get
# или
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader \
  org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# Должно быть scx_lavd
```

Ananicy-cpp нормально работает вместе с планировщиками scx.

### Управление сервисами

Отключайте только то, чем реально не пользуетесь:

| Сервис | Можно отключить, если | Что сломается |
|---|---|---|
| `bluetooth` | нет BT-устройств | BT-наушники, геймпады |
| `ModemManager` | нет 3G/4G-модема | мобильные модемы |
| `cups` | нет принтера | печать |
| `avahi-daemon` | нет сетевых принтеров / mDNS | поиск `.local`, сетевые принтеры |
| `NetworkManager-wait-online` | почти всегда | на десктопе ничего, загрузка быстрее |
| `geoclue` | не нужна автогеолокация | автоматический часовой пояс, ночной режим по геолокации |
| `smartd` | не нужны предупреждения о здоровье диска | SMART-уведомления |
| `upower` | **только десктоп** | инфо о батарее на ноутбуке, часть функций GNOME |
| `chronyd` | **не рекомендую** | синхронизация времени. Со временем часы уплывают, ломаются TLS, логины и некоторые онлайн-игры |

```bash
sudo systemctl disable --now bluetooth ModemManager cups avahi-daemon \
  NetworkManager-wait-online geoclue smartd
```

-----

## Параметры ядра GRUB

### Настройка

Fedora использует BLS-записи, поэтому надёжнее всего `grubby`. Он **добавляет** параметры и не трогает существующие (`rhgb quiet`, `rd.luks`, `resume` и т.д.):

```bash
sudo grubby --update-kernel=ALL --args="amdgpu.ppfeaturemask=0xffffffff split_lock_detect=off amd_pstate=active page_alloc.shuffle=1 pci=pcie_bus_perf pcie_aspm.policy=performance usbcore.autosuspend=-1 nowatchdog nmi_watchdog=0 zswap.enabled=0"
```

Убрать параметр потом:

```bash
sudo grubby --update-kernel=ALL --remove-args="zswap.enabled=0"
```

Вручную: допишите параметры **в конец** существующей `GRUB_CMDLINE_LINUX` в `/etc/default/grub` (строку не заменяйте), затем `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`.

Проверка после перезагрузки: `cat /proc/cmdline`

### Что делают параметры

| Параметр | Эффект | Примечания |
|---|---|---|
| `amd_pstate=active` | EPP-драйвер, быстрый буст частот | AMD Zen 2+ с CPPC. На свежих ядрах и так по умолчанию, параметр просто закрепляет. Intel удалить |
| `amdgpu.ppfeaturemask=0xffffffff` | Разблокирует напряжение и лимиты разгона | Нужно для LACT / CoreCtrl. Только AMD GPU |
| `split_lock_detect=off` | Убирает замедления и фризы в играх со split locks | Безопасно для всех |
| `nowatchdog` + `nmi_watchdog=0` | Отключает watchdog-таймеры, меньше микрофризов | Безопасно для всех |
| `zswap.enabled=0` | Отключает zswap | Fedora всё равно использует zram. Уберите, если у вас swap на диске и 16GB RAM или меньше |
| `pci=pcie_bus_perf` | Максимальный payload PCIe для GPU/NVMe | Изредка на некоторых платах отваливаются устройства. Пропало что-то, уберите |
| `pcie_aspm.policy=performance` | Отключает энергосбережение PCIe, убирает скачки задержки в простое | Лучше, чем `pcie_aspm=off`. На ноутбуке выше расход |
| `usbcore.autosuspend=-1` | USB-устройства не засыпают, нет задержки «пробуждения» ввода | Безопасно для всех |
| `page_alloc.shuffle=1` | Перемешивает free-list аллокатора страниц | В основном про безопасность и memory-side cache. В играх прирост маленький, оставляйте по желанию |

### Экспериментально, на свой риск

| Параметр | Плюс | Предупреждение |
|---|---|---|
| `mitigations=off` | ~3% FPS на старых CPU | Отключает защиту от уязвимостей CPU. На Zen 4 / Zen 5 (Ryzen 7000/9000) по отзывам хуже 1% lows, там не рекомендую. Смысл в основном для Zen 1/2/3 и Intel 9-го поколения и старше |

Альтернатива `split_lock_detect=off` без правки cmdline (ядро 6.2+):

```bash
echo 'kernel.split_lock_mitigate = 0' | sudo tee /etc/sysctl.d/99-splitlock.conf
sudo sysctl --system
```

-----

## Продвинутые системные твики

### Управление памятью

systemd-oomd на Fedora Workstation уже включён. На некоторых системах он убивает игры при нехватке памяти. Проверьте и решите сами:

```bash
systemctl status systemd-oomd
# выключить:
# sudo systemctl disable --now systemd-oomd
```

### zram

Fedora уже использует zram. Можно увеличить размер и включить zstd:

```bash
sudo tee /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = ram
compression-algorithm = zstd
EOF
sudo systemctl daemon-reload
sudo systemctl restart systemd-zram-setup@zram0
zramctl
```

### sysctl

Используйте файл в `/etc/sysctl.d/`, а не дописывайте в `/etc/sysctl.conf`:

```bash
sudo tee /etc/sysctl.d/99-gaming.conf <<'EOF'
# zram быстрый, поэтому ВЫСОКИЙ swappiness тут правильный (не низкий)
vm.swappiness = 150
# дольше держать кэш dentry/inode
vm.vfs_cache_pressure = 50
# Fedora с F39 уже ставит 1048576, тут поднимаем ещё выше
vm.max_map_count = 16777216
EOF
sudo sysctl --system
```

> `cachyos-settings` тоже ставит свои sysctl. Файлы применяются по алфавиту, `99-gaming.conf` побеждает.

Слишком низкий `vm.max_map_count` вызывает краши в Hogwarts Legacy, CS2, The Finals, DayZ, Star Citizen. Проверка: `sysctl vm.max_map_count`

### Накопитель

Таймер TRIM на Fedora включён по умолчанию. Проверить и запустить вручную:

```bash
systemctl status fstrim.timer
sudo fstrim -v /
```

### CPU scaling

AMD:

```bash
echo active | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

Intel: `active` (по умолчанию) с EPP обычно лучший вариант. `passive` нужен, только если хотите governor `schedutil`:

```bash
echo passive | sudo tee /sys/devices/system/cpu/intel_pstate/status
```

### Лимит файловых дескрипторов

В `/etc/security/limits.conf` (замените `yourusername`):

```bash
yourusername hard nofile 1048576
yourusername soft nofile 1048576
```

Важно в основном для старого esync. С NTSync не критично.

### IRQ Balance (Intel iGPU)

Если есть проблемы с производительностью на встройке Intel:

```bash
sudo systemctl status irqbalance
sudo systemctl disable --now irqbalance
```

### I/O scheduler

Параметр ядра `elevator=` больше не работает. Делаем через udev:

```bash
sudo tee /etc/udev/rules.d/60-ioschedulers.rules <<'EOF'
# HDD
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
# SATA SSD / eMMC
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
# NVMe: своё управление очередями, планировщик не нужен
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
grep "" /sys/block/*/queue/scheduler
```

Спасибо netarchy за этот метод! В `cachyos-settings` может уже быть похожее правило, проверка: `ls /usr/lib/udev/rules.d | grep -i sched`.

-----

## Оптимизация для гейминга

### Steam

```bash
sudo dnf install steam
```

**Фоновая обработка шейдеров:** ускоряет предкомпиляцию Vulkan-шейдеров. Закройте Steam, затем:

```bash
echo 'unShaderBackgroundProcessingThreads 8' >> ~/.steam/steam/steam_dev.cfg
```

Число ставьте примерно в половину потоков CPU.

### GameMode

```bash
sudo dnf install gamemode
gamemoded -t
```

В параметрах запуска Steam: `gamemoderun %command%`.

### Версии Proton

- **Proton 11** (Valve, стабильный с июля 2026): на базе Wine 11, сам использует NTSync, если он есть в ядре.
- **Proton-CachyOS**: доп. патчи и встроенный апгрейд FSR4. С версии 11.0-20260702 сам кладёт FSR 4.1.1 в каждый префикс на RDNA 2-4, переменная `PROTON_FSR4_RDNA3_UPGRADE` больше не нужна.

Кастомные Proton удобно ставить через ProtonPlus:

```bash
flatpak install flathub com.vysp3r.ProtonPlus
```

### Windows-игры

**PortProton** мой любимый лаунчер Proton, использую его вместо Lutris / Bottles:

```bash
sudo dnf copr enable boria138/portproton
sudo dnf install portproton
```

### NTSync

В mainline с ядра 6.14, в ядрах Fedora 44/45 и CachyOS есть. Проверка:

```bash
ls -l /dev/ntsync
# нет? загрузить и закрепить:
sudo modprobe ntsync
echo ntsync | sudo tee /etc/modules-load.d/ntsync.conf
```

-----

## Справка по параметрам запуска

### Какую команду использовать?

| Ситуация | Команда |
|---|---|
| В игре есть нативный HDR (Cyberpunk, Elden Ring...) | HDR |
| HDR нет, фейковый не нужен | Без HDR |
| HDR нет, хочу Auto HDR как в Windows | SDR в HDR |

### Переменные окружения и обёртки

| Переменная | Что делает |
|---|---|
| `LD_PRELOAD=""` | Фиксит микрофризы Steam через ~25-30 минут игры. Побочка: не будет оверлея Steam |
| `PROTON_USE_NTSYNC=1` | Принудительно включает NTSync в GE / старых CachyOS Proton. Proton 11 делает это сам, переменная не мешает |
| `AMD_USERQ=1` | **RDNA 3/4.** User queues: работа идёт прямо в аппаратные очереди GPU, меньше overhead ядра. Нужны Mesa 25.2+ и ядро с включёнными user queues (добавьте `amdgpu.user_queue=1`, если в `dmesg` про них пусто). В первую очередь влияет на OpenGL (radeonsi), для Vulkan прирост не гарантирован |
| `DXVK_HDR=1` | Разрешает HDR-вывод в DXVK-играх |
| `MANGOHUD_CONFIG="fps_limit=277,no_display"` | Лимитер кадров MangoHud без оверлея |
| `fps_limit=277` | Лимит с низкой задержкой: макс. герцовка минус 3 (280 Hz -> 277). Держит кадры в диапазоне VRR без input lag от V-Sync |
| `mangohud` | Подключает лимитер. Ставьте **после** `--`, чтобы он цеплялся к игре, а не к gamescope |
| `gamemoderun` | Feral GameMode на время сессии |

### Аргументы Gamescope

| Аргумент | Что делает |
|---|---|
| `gamescope` | Микрокомпозитор Valve. Изолирует игру, часто чинит краши при Alt+Tab, нормально работает с HDR и разрешением |
| `-W 2560 -H 1440` | **Выходное** разрешение gamescope. Ставьте нативное разрешение монитора |
| `-w 1920 -h 1080` | Опционально: разрешение, в котором рендерит **игра**. По умолчанию равно `-W/-H`. Нужно для апскейла |
| `-r 280` | Лимит FPS игры внутри gamescope. Ставьте макс. герцовку монитора. Если лимит MangoHud ниже, работает он |
| `--hdr-enabled` | Включает HDR-вывод |
| `--force-grab-cursor` | Не даёт мышке уйти на второй монитор |
| `--adaptive-sync` | VRR (FreeSync / G-Sync) внутри контейнера |
| `-F fsr --sharpness 2` | Опционально: апскейл FSR 1 из `-w/-h` в `-W/-H`. Резкость 0 = максимум, 20 = мягко. Без `-F` резкость ничего не делает |
| `-f` | Полный экран |
| `--` | Конец флагов gamescope, дальше команда игры |
| `%command%` | Steam подставляет сюда исполняемый файл игры |

### Аргументы Auto SDR в HDR

Используются вместе с `--hdr-enabled`:

| Аргумент | Что делает |
|---|---|
| `--hdr-itm-enable` | Inverse tone mapping, тот самый «Auto HDR» |
| `--hdr-itm-sdr-nits 100` | Какой яркостью считать SDR-исходник. 100 хороший старт |
| `--hdr-itm-target-nits 1000` | Целевая пиковая яркость. Ставьте реальный пик монитора |
| `--hdr-sdr-content-nits 203` | Paper white для UI и обычных текстур. 203 стандарт: выше слепит в меню, ниже тускло |

### Команды

**Нативный HDR:**

```bash
LD_PRELOAD="" PROTON_USE_NTSYNC=1 AMD_USERQ=1 DXVK_HDR=1 MANGOHUD_CONFIG="fps_limit=277,no_display" gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --force-grab-cursor --adaptive-sync -f -- mangohud %command%
```

**Без HDR:**

```bash
LD_PRELOAD="" PROTON_USE_NTSYNC=1 AMD_USERQ=1 MANGOHUD_CONFIG="fps_limit=277,no_display" gamemoderun gamescope -W 2560 -H 1440 -r 280 --force-grab-cursor --adaptive-sync -f -- mangohud %command%
```

**Без HDR + Auto SDR в HDR (как Auto HDR в Windows):**

```bash
LD_PRELOAD="" PROTON_USE_NTSYNC=1 AMD_USERQ=1 DXVK_HDR=1 MANGOHUD_CONFIG="fps_limit=277,no_display" gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --hdr-itm-enable --hdr-itm-sdr-nits 100 --hdr-itm-target-nits 1000 --hdr-sdr-content-nits 203 --force-grab-cursor --adaptive-sync -f -- mangohud %command%
```

> Поменяйте `-W`, `-H`, `-r` и `fps_limit` под свой монитор. На NVIDIA уберите `AMD_USERQ=1`.

**Апгрейд FSR4 / DLSS (Proton-CachyOS / GE):**

```bash
# RDNA 4 (в новых Proton-CachyOS также выбирает версию DLL)
PROTON_FSR4_UPGRADE=1 %command%
# NVIDIA: свежие DLL DLSS
PROTON_DLSS_UPGRADE=1 %command%
```

-----

## Обслуживание и очистка

```bash
# Кэш DNF
sudo dnf clean all

# Журнал: 7 дней или 100MB
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=100M

# Старые ядра: оставить последние 3
sudo dnf remove $(dnf repoquery --installonly --latest-limit=-3 -q)
```

### Скрипт обслуживания

```bash
#!/bin/bash
# Сохраните как ~/maintenance.sh, затем: chmod +x ~/maintenance.sh
echo "Запуск обслуживания..."
sudo dnf upgrade --refresh
sudo dnf clean all
sudo journalctl --vacuum-time=7d
sudo fstrim -v /
flatpak update -y
flatpak uninstall --unused -y
echo "Готово!"
```

-----

## Окружения рабочего стола

### Лёгкие альтернативы

Если ставили с minimal ISO:

- **Sway** - Wayland tiling compositor (~700MB в простое у меня)
- **i3** - X11 tiling WM (~600MB)
- **Hyprland** - Wayland compositor с анимациями (~900MB)
- **XFCE** - лёгкий классический desktop
- **LXQt** - лёгкий desktop на Qt

**KDE Plasma** официальная редакция Fedora наряду с Workstation (GNOME), хорошо интегрирована из коробки.

### Профиль питания (десктоп)

Fedora использует tuned + tuned-ppd. На десктопе просто переключитесь на performance:

```bash
powerprofilesctl set performance
powerprofilesctl get
```

### Питание ноутбука (TLP)

TLP конфликтует с tuned / tuned-ppd. Сначала удалите их, иначе они будут перетягивать одни и те же настройки:

```bash
sudo dnf remove tuned tuned-ppd
sudo dnf install tlp tlp-rdw powertop
sudo systemctl enable --now tlp
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
```

Производительность от сети в `/etc/tlp.conf` (`TLP_DEFAULT_MODE` принимает только `AC` или `BAT`):

```ini
TLP_DEFAULT_MODE=AC
CPU_ENERGY_PERF_POLICY_ON_AC=performance
PLATFORM_PROFILE_ON_AC=performance
```

Применить: `sudo tlp start`. Проверить: `sudo tlp-stat -s`

### Steam во Flatpak

```bash
flatpak install flathub com.valvesoftware.Steam
# Дать доступ к библиотеке игр на другом диске:
flatpak override --user --filesystem=/mnt/games com.valvesoftware.Steam
```

### GNOME

```bash
sudo dnf install gnome-tweaks gnome-extensions-app

# Отключить анимации
gsettings set org.gnome.desktop.interface enable-animations false

# Рабочие столы на всех мониторах, а не только на основном
gsettings set org.gnome.mutter workspaces-only-on-primary false
```

-----

## Оптимизация графических драйверов

<details>
<summary>🔴 Оптимизация AMD</summary>

### Драйверы

Mesa стоит по умолчанию. Убедитесь, что есть Vulkan-драйвер и VA-API из RPM Fusion (см. [RPM Fusion](#2-включение-rpm-fusion)):

```bash
sudo dnf install mesa-vulkan-drivers mesa-vulkan-drivers.i686

# ROCm для вычислений (опционально)
sudo dnf install rocm-opencl rocm-smi
```

> `mesa-vdpau-drivers` больше нет: VDPAU удалили из Mesa в 25.3, в Fedora 44+ пакета нет. Используйте VA-API.

### Переменные окружения

Опционально! Некоторые могут снизить FPS или сделать игры нестабильными. Проверяйте после каждого изменения.

```bash
sudo mkdir -p /etc/environment.d
sudoedit /etc/environment.d/99-amd-gaming.conf
```

```ini
# Многопоточность OpenGL. Отлично для Minecraft, в других играх может снизить FPS
mesa_glthread=true

# Драйвер VA-API
LIBVA_DRIVER_NAME=radeonsi

# Имеет смысл, только если стоит AMDVLK. AMDVLK больше не развивается, RADV и так по умолчанию
# AMD_VULKAN_ICD=RADV

# Больше не нужно: gpl, nggc, rt и sam в текущей Mesa уже включены
# RADV_PERFTEST=gpl,nggc,sam,rt
```

**Resizable BAR:** переменной окружения для него нет, включается в BIOS (Above 4G Decoding + ReBAR). Проверка:

```bash
sudo dmesg | grep -i "BAR="
```

### Уровень производительности GPU

Фиксирует частоты GPU на максимуме. Выше потребление и нагрев в простое. Не используйте вместе с LACT, он делает то же самое лучше.

```bash
# Сейчас:
for f in /sys/class/drm/card*/device/power_dpm_force_performance_level; do echo high | sudo tee "$f"; done
```

Постоянный сервис:

```bash
sudo tee /etc/systemd/system/amd-gpu-performance.service <<'EOF'
[Unit]
Description=AMD GPU Performance Mode
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'for f in /sys/class/drm/card*/device/power_dpm_force_performance_level; do echo high > "$f"; done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now amd-gpu-performance.service
```

### AMD User Queues (RDNA 3/4)

Добавьте `AMD_USERQ=1` в параметры запуска Steam. Подробности в [справке по параметрам запуска](#справка-по-параметрам-запуска).

### FSR4, Anti-Lag 2 и HDR: мои заметки (RX 9070 XT)

Тестил на RX 9070 XT, на RX 7000 и 9000 должно быть так же. Ставьте только если игре это нужно, а не всегда.

| Проблема | Решение |
|---|---|
| FSR4 не появляется в настройках игры | `PROTON_FSR4_UPGRADE=1 PROTON_MLFG_UPGRADE=1` |
| OptiScaler ставит XeLL вместо Anti-Lag 2 | `LOW_LATENCY_LAYER=1` |
| Опция HDR серая или её нет | `PROTON_ENABLE_WAYLAND=1 DXVK_HDR=1` |

**FSR4.** Большинство игр видят его сами. Но в некоторых FSR4 не появляется, пока не добавишь обе переменные:

```bash
PROTON_FSR4_UPGRADE=1 PROTON_MLFG_UPGRADE=1 %command%
```

**Anti-Lag 2 через OptiScaler.** По умолчанию OptiScaler выставляет XeLL. Если нужен Anti-Lag 2, добавьте:

```bash
LOW_LATENCY_LAYER=1 %command%
```

**HDR.** Некоторые игры вообще не дают включить HDR без этого:

```bash
PROTON_ENABLE_WAYLAND=1 DXVK_HDR=1 %command%
```

Всё это можно комбинировать в одной строке с остальными параметрами запуска.

### LACT

Аналог AMD Adrenalin: voltage offset, частота памяти, power limit, кривая вентиляторов. Работает благодаря `amdgpu.ppfeaturemask=0xffffffff` из GRUB.

```bash
sudo dnf copr enable ilyaz/LACT
sudo dnf install lact     # альтернативы: lact-headless, lact-libadwaita
sudo systemctl enable --now lactd
```

Старт для RX 9070 XT: Performance Level: Manual, offset -70mV, частота памяти 2750MHz.
- Разгон: Power Limit 355W, примерно +3-4% FPS.
- Андервольт: Power Limit 260-270W, примерно -3% FPS.

</details>

<details>
<summary>🟢 Оптимизация NVIDIA</summary>

## Оптимизация NVIDIA для Fedora

### Какой драйвер для какой карты

| GPU | Ветка драйвера | Пакет |
|---|---|---|
| RTX 20/30/40/50, GTX 16 (Turing+) | 595+ (по умолчанию на F44/F45, открытые модули ядра) | `akmod-nvidia` |
| GTX 750/900/10 (Maxwell, Pascal), Titan V | 580xx legacy | `akmod-nvidia-580xx` |

NVIDIA 590+ убрала поддержку Maxwell, Pascal и Volta. Эти карты теперь получают только квартальные обновления безопасности в legacy-ветке 580. Если после апгрейда на Fedora 44 драйвер на GTX 10 перестал грузиться, причина в этом.

### Установка (RPM Fusion)

```bash
# RPM Fusion должен быть включён (см. Начальная настройка)
sudo dnf upgrade --refresh
```

**Turing и новее:**

```bash
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda xorg-x11-drv-nvidia-libs.i686
```

**Maxwell / Pascal (GTX 750-1080 Ti):**

```bash
sudo dnf install akmod-nvidia-580xx xorg-x11-drv-nvidia-580xx-cuda xorg-x11-drv-nvidia-580xx-libs.i686
```

Подождите ~5 минут, пока akmods соберёт модуль, проверьте и перезагрузитесь:

```bash
modinfo -F version nvidia
sudo reboot
```

> Не ставьте `akmod-nvidia-open` для обычной установки. В RPM Fusion `akmod-nvidia` уже использует открытые модули на Turing+.

**Secure Boot:** подпишите модуль один раз и подтвердите ключ при следующей загрузке:

```bash
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
sudo reboot
```

Сервисы сна:

```bash
sudo systemctl enable nvidia-suspend nvidia-resume nvidia-hibernate
```

#### Проверка

```bash
nvidia-smi
nvidia-smi -q | grep "CUDA Version"
echo $XDG_SESSION_TYPE      # должно быть wayland
cat /sys/module/nvidia_drm/parameters/modeset   # должно быть Y
```

#### Modeset для Wayland (только если нужно)

С RPM Fusion `akmod-nvidia` включает modeset сам. Делайте вручную, только если Wayland не стартует или ставили драйвер через `.run`:

```bash
echo 'options nvidia_drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-drm-modeset.conf
sudo dracut --force
sudo reboot
```

-----

### Переменные окружения

> **Внимание:** не добавляйте всё подряд в `/etc/environment`. Глобальные переменные NVIDIA могут сломать вход в Wayland, дать чёрный экран, статтеры или input lag в зависимости от карты, драйвера и DE. Сначала тестируйте per-game в параметрах запуска Steam.

`/etc/environment`:

```bash
# ТОЛЬКО для GTX 10 и старше. На RTX 20+ эти две дают системный input lag и статтеры
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia

# Параллелизм CPU-GPU. На некоторых RTX даёт чёрный экран, лучше per-game
__GL_THREADED_OPTIMIZATIONS=1

# Кэш шейдеров. Путь по умолчанию ~/.cache/nvidia (переживает перезагрузку).
# Не переносите его в /tmp: он чистится при перезагрузке, и статтеры возвращаются
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_SIZE=10737418240
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1

# Без VSync в OpenGL (меньше input lag)
__GL_SYNC_TO_VBLANK=0

# DLSS и другие функции NVAPI в Proton
PROTON_ENABLE_NVAPI=1

# Фикс статтеров Steam через 25-30+ минут
LD_PRELOAD=""

# Нативный Wayland в Proton. Лучше per-game: часть игр теряет эксклюзивный fullscreen или ломается
# PROTON_ENABLE_WAYLAND=1

# Только для wlroots (Sway, Hyprland), на GNOME/KDE не нужно.
# С драйвером 595+ и explicit sync могут быть не нужны, сначала тестируйте без них
# WLR_DRM_NO_ATOMIC=1
# WLR_NO_HARDWARE_CURSORS=1
```

Если после глобального `__GL_THREADED_OPTIMIZATIONS=1` получили чёрный экран:
1. Ctrl+Alt+F3, войдите
2. `sudoedit /etc/environment`, удалите строку
3. `sudo reboot`

Спасибо @lemonadeforlife за решение.

### Параметры модуля ядра (если понимаете, зачем)

Тестируйте каждый на своём железе. `/etc/modprobe.d/nvidia-power-management.conf`:

```bash
# Runtime power management (ноутбуки)
options nvidia NVreg_DynamicPowerManagement=0x02
# Page Attribute Table
options nvidia NVreg_UsePageAttributeTable=1
# Resizable BAR (RTX 30/40/50)
options nvidia NVreg_EnableResizableBar=1
# Сохранять VRAM при сне
options nvidia NVreg_PreserveVideoMemoryAllocations=1
# Stream memory operations
options nvidia NVreg_EnableStreamMemOPs=1
```

Потом `sudo dracut --force` и перезагрузка.

### GNOME Wayland

Fedora 44 (GNOME 50) и 45 (GNOME 51): VRR стабилен. Включается в **Настройки > Дисплеи > Variable Refresh Rate**. Старая команда gsettings больше не нужна и при этом **перезаписывает** другие экспериментальные функции (например, дробное масштабирование), так что лучше её не использовать.

```bash
gsettings set org.gnome.desktop.interface enable-animations false
```

### KDE Plasma Wayland (6.x)

VRR: **Параметры системы > Дисплей и монитор > Адаптивная синхронизация**.

Старые ключи `kwinrc` `LatencyPolicy` и `RenderTimeEstimator` убрали в Plasma 6 (KWin теперь сам измеряет время рендера), команды с ними больше ничего не делают.

-----

### Параметры запуска Steam

Нативные Linux-игры:

```bash
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 __GL_SYNC_TO_VBLANK=0 %command%
```

Игры через Proton:

```bash
# Стандарт
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Максимум производительности
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 PROTON_DLSS_UPGRADE=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%
```

`PROTON_DLSS_UPGRADE=1` работает в Proton-CachyOS / GE.

### Lutris

```bash
sudo dnf install lutris wine
```

Настройки системы > Переменные окружения:

```
__GL_THREADED_OPTIMIZATIONS=1
__GL_SHADER_DISK_CACHE=1
PROTON_ENABLE_NVAPI=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
```

### GameMode

```bash
sudo dnf install gamemode
sudoedit /etc/gamemode.ini
```

```ini
[general]
renice=10
ioprio=0

[gpu]
# Внимание: пытается разогнать GPU через nvidia-settings. Тестируйте осторожно
apply_gpu_optimisations=accept-responsibility
gpu_device=0
```

-----

### Продвинутое

#### Проверка VRR

```bash
sudo dnf install drm_info
drm_info | grep -i vrr
```

#### HDR (экспериментально)

```bash
# Непроверенный параметр, его нет в официальной документации NVIDIA. Тестируйте перед применением
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf
```

#### Скрипт мониторинга

```bash
sudo dnf install nvtop mangohud goverlay

sudo tee /usr/local/bin/nvidia-monitor.sh <<'EOF'
#!/bin/bash
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
nvidia-smi dmon -s pucvmet
EOF
sudo chmod +x /usr/local/bin/nvidia-monitor.sh
```

#### Runtime PM на ноутбуке (если понимаете, что делаете)

```bash
echo 'options nvidia NVreg_DynamicPowerManagement=0x02' | sudo tee /etc/modprobe.d/nvidia-power.conf

sudo tee /etc/udev/rules.d/80-nvidia-pm.rules <<'EOF'
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
EOF
```

-----

### Решение проблем NVIDIA

**Чёрный экран / Wayland не стартует:**

1. Ctrl+Alt+F3 (или в GRUB выберите прошлое ядро)
2. Закомментируйте `__GL_THREADED_OPTIMIZATIONS=1` в `/etc/environment`
3. Проверьте modeset:

```bash
cat /sys/module/nvidia_drm/parameters/modeset   # Y
sudo dracut --force
sudo reboot
```

**На GTX 10 / 900 драйвер не грузится после обновления:** у вас 595, переходите на 580xx (таблица выше).

**Низкая производительность:**

```bash
nvidia-smi dmon -s pucvmet -c 10
nvidia-smi --query-gpu=power.limit,power.draw --format=csv
watch -n 1 'nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv'
```

В игре потребление и частоты должны подходить к максимуму.

**Тиринг:** включите VRR в настройках дисплея, VSync в игре выключите, добавьте `__GL_SYNC_TO_VBLANK=0 %command%`.

**Высокое потребление в простое (ноутбуки):**

```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status   # в простое suspended
nvidia-smi --query-gpu=power.draw --format=csv --loop=1
```

-----

### Конфиг MangoHud

```bash
mkdir -p ~/.config/MangoHud
tee ~/.config/MangoHud/MangoHud.conf <<'EOF'
gpu_stats
cpu_stats
gpu_temp
cpu_temp
fps
frametime
frame_timing
vram
ram
position=top-left
font_size=22
background_alpha=0.6
EOF
```

### Ресурсы

- [RPM Fusion NVIDIA Guide](https://rpmfusion.org/Howto/NVIDIA)
- [NVIDIA Linux Driver Docs](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
- [MangoHud](https://github.com/flightlessmango/MangoHud)

</details>

-----

## Мониторинг и проверка

```bash
sudo dnf install btop nvtop inxi
inxi -Fxz          # полная сводка по системе
systemd-analyze blame | head -20   # самые медленные сервисы при загрузке
```

### Бенчмарки

```bash
sudo dnf install glmark2 sysbench stress-ng
```

Unigine Superposition: качается с [unigine.com](https://benchmark.unigine.com/superposition).

-----

## Решение проблем

**1. Не грузится после параметров ядра:**

- Выберите прошлое ядро в меню GRUB (Shift / Esc при загрузке)
- `sudo grubby --update-kernel=ALL --remove-args="<параметр>"`

**2. Проблемы с графикой:**

```bash
lspci -k | grep -A 2 -E "(VGA|3D)"
lsmod | grep -E "(amdgpu|nvidia|i915|xe)"
```

**3. Регресс производительности:**

- Ресурсы: `btop`, `iotop`
- Троттлинг: `watch sensors`
- Упавшие сервисы: `systemctl list-units --failed`
- Отключайте твики по одному и сравнивайте

### Команды восстановления

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
# если включаете SELinux обратно:
sudo touch /.autorelabel && sudo reboot
sudo dnf check
sudo rpm -Va
```

-----

## Ожидаемый прирост

По моим тестам:

| Область | Прирост |
|---|---|
| Загрузка | на 10-20% быстрее |
| Игры | +5-15% FPS |
| Отзывчивость | заметно ниже input lag |
| RAM в простое | на 5-15% меньше |
| Накопитель | лучше работа SSD с TRIM |

-----

## Участие

Нашли улучшения или есть идеи:

- Откройте issue на GitHub
- Отправьте pull request
- Поделитесь своими результатами

## Ресурсы

- [Fedora Documentation](https://docs.fedoraproject.org/)
- [RPM Fusion](https://rpmfusion.org/)
- [CachyOS Kernel](https://github.com/CachyOS/linux-cachyos)
- [copr-linux-cachyos](https://github.com/CachyOS/copr-linux-cachyos)
- [sched-ext/scx](https://github.com/sched-ext/scx)
- [Ananicy-cpp](https://gitlab.com/ananicy-cpp/ananicy-cpp)
- [Gaming on Linux](https://www.gamingonlinux.com/)

## Дисклеймер

Гайд меняет системные настройки, это может повлиять на стабильность и безопасность. Перед применением:

- Сделайте бэкап
- Понимайте, что делает каждое изменение
- Держите под рукой загрузочную флешку

Результаты зависят от железа и сценария.

<details>
<summary>Список изменений</summary>

- **v1.0** - первый релиз
- **v1.1** - troubleshooting и русский перевод
- **v1.2** - мониторинг и обслуживание
- **v1.3** - NVIDIA и другие разделы
- **v1.4** - твики AMD GPU
- **v1.5** - быстрая навигация
- **v1.6** - переход с UKSMD на KSMD, подробнее про ядро CachyOS
- **v1.7** - больше флагов NVIDIA, исправления, подготовка к Fedora 43
- **v1.8** - обновлена почти вся структура, особенно AMD
- **v1.9** - больше деталей, обновление под Fedora 44
- **v2.0** - раздел Fedora 45, NVIDIA 580xx для Pascal/Maxwell, параметры ядра через grubby, исправлены флаги gamescope и конфиг scx_lavd, ananicy-cpp из COPR, мультимедиа RPM Fusion, zram, Proton 11 / FSR4, все команды переписаны так, чтобы копировались и работали как есть

</details>

*Последнее обновление: октябрь 2026*

</details>

-----

## Contributing

Found improvements or have suggestions? Feel free to:

- Open an issue on GitHub
- Submit a pull request
- Share your optimization results

## Additional Resources

- [Fedora Documentation](https://docs.fedoraproject.org/)
- [RPM Fusion](https://rpmfusion.org/)
- [CachyOS Kernel](https://github.com/CachyOS/linux-cachyos)
- [copr-linux-cachyos](https://github.com/CachyOS/copr-linux-cachyos)
- [sched-ext/scx](https://github.com/sched-ext/scx)
- [Ananicy-cpp](https://gitlab.com/ananicy-cpp/ananicy-cpp)
- [Gaming on Linux](https://www.gamingonlinux.com/)

## Disclaimer

This guide changes system settings that can affect stability and security. Always:

- Back up your system first
- Understand what each change does
- Keep recovery media at hand

Results vary with hardware and use case.

<details>
<summary>Changelog</summary>

- **v1.0** - Initial guide
- **v1.1** - Troubleshooting section and Russian translation
- **v1.2** - Monitoring tools and maintenance section
- **v1.3** - NVIDIA drivers, performance guide and more
- **v1.4** - AMD GPU tweaks
- **v1.5** - Quick Navigation
- **v1.6** - UKSMD to KSMD, deeper CachyOS kernel setup
- **v1.7** - More NVIDIA flags, corrections, Fedora 43 prep
- **v1.8** - Updated almost every section, especially AMD
- **v1.9** - More details, updated for Fedora 44
- **v2.0** - Fedora 45 section, NVIDIA 580xx for Pascal/Maxwell, kernel params via grubby, fixed gamescope flags and scx_lavd config, ananicy-cpp from COPR, RPM Fusion multimedia, zram, Proton 11 / FSR4, every command rewritten to copy and run as is

</details>

*Last updated: October 2026*
