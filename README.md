# Fedora Linux 44 Performance & Gaming Optimization Guide

> **Complete guide for optimizing Fedora 44 for maximum performance, gaming, general use, etc | by winterofhell**

## Quick Navigation

| Setup & Kernel                                                              | System & Gaming                                                            | Resources                                                                 |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [**System Information**](#system-information)                              | [**Advanced System Tweaks**](#advanced-system-tweaks)                     | [**Monitoring & Verification**](#monitoring--verification)               |
| [**Initial Setup & Preparation**](#initial-setup--preparation)             | [**Gaming Optimizations**](#gaming-optimizations)                         | [**Troubleshooting**](#troubleshooting)                                  |
| [**Kernel Optimization**](#kernel-optimization)                            | [**Maintenance & Cleanup**](#maintenance--cleanup)                        | [**Russian Translation**](#русская-версия--russian-translation)     |
| [**GRUB Kernel Parameters**](#grub-kernel-parameters)                       | [**Graphics Driver Optimization**](#graphics-driver-optimization)          |                                                                           |

## System Information

**Testing Environment:**

- **Period:** October 14, 2024 - July 7, 2026
- **Distribution:** Fedora 44
- **Additional Testing:** NVIDIA and AMD gpu systems
- **These optimizations may also work on any other distro, but i cannot guarantee it. It is always necessary to test everything. About 90% of these tweaks work on Arch and NixOS (tested by myself) :)**

**Hardware Configurations (tested on):**

- **First:** Ryzen 5 5500U, 20GB DDR4, RX550X/RX Vega 7, Nvme ssd
- **Second:** Ryzen 5 5600, 16GB DDR4, GTX 1060, SATA SSD
- **Third:** Ryzen 5 7500f / 7 9850x3d, 32Gb DDR5, RX 9070 XT, Nvme ssd

-----

## Initial Setup & Preparation

### 1. Minimal Installation

For optimal performance, always start with the **Fedora Minimal ISO**. This approach eliminates unnecessary packages and services that can impact system resources. But it is not necessary to do this.

### 2. Enable RPM Fusion Repositories

RPM Fusion provides essential multimedia codecs and proprietary drivers:

```bash
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

**Official Guide:** [RPM Fusion Configuration](https://rpmfusion.org/Configuration)

### 3. SELinux Configuration (Optional)

**Security Warning:** Disabling SELinux reduces system security but also makes ur system a little faster. Only proceed if you understand the implications. (Personally, I don't care about SELinux and i always disable it)

**Temporary disable (until reboot):**

```bash
sudo setenforce 0
```

**Permanent disable (requires reboot):**

```bash
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# Reboot required for changes to take effect
```

### 4. System Update

Always start with a fully updated system:

```bash
sudo dnf upgrade --refresh

# Install firmware updates (including CPU microcode)
sudo dnf install linux-firmware intel-ucode amd-ucode
```

-----

## Kernel Optimization

### CachyOS Kernel Installation

The CachyOS kernel provides significant performance improvements for gaming and general system responsiveness.

**Prerequisites:** CPU must support x86_64_v3 instruction set !!

```bash
# Checking for cpu support
# Check support by running the following command

/lib64/ld-linux-x86-64.so.2 --help | grep "(supported, searched)"

# If it does not detect x86_64_v3 support do NOT install this kernel. If it detects only x86_64_v2, you can use the LTS kernel.
```

```bash
# Add CachyOS COPR repository
sudo dnf copr enable bieszczaders/kernel-cachyos
sudo dnf copr enable bieszczaders/kernel-cachyos-addons

# Install CachyOS kernel
sudo dnf install kernel-cachyos kernel-cachyos-devel

# For x86_64_v2 only (older CPUs):
sudo dnf install kernel-cachyos-lts kernel-cachyos-lts-devel-matched
```

### Make CachyOS Kernel the Default Kernel

After installing the CachyOS kernel, Fedora may still boot another kernel by default.  
You can use `grubby` to check and change the default kernel entry.

```bash
# List all installed kernel entries
sudo grubby --info=ALL | grep -E "^kernel|^index|^title"

# Check the current default kernel
sudo grubby --default-kernel
sudo grubby --default-index

# Find the entry that corresponds to the CachyOS kernel, then set it as default:

sudo grubby --set-default-index=<INDEX_NUMBER>

# Example:

sudo grubby --set-default-index=0

# Reboot and verify:

uname -r

# The output should contain cachyos if the CachyOS kernel is currently running.
```

**More Info:** [CachyOS Kernel Installation](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/)

-----

## System Services Optimization

### Ananicy-cpp Installation

Ananicy-cpp automatically manages process priorities and reduces system latency:

# Install build dependencies
```bash
sudo dnf group install "Development Tools"
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel make automake gcc gcc-c++
```

# Clone and build
```bash
git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
```

# Enable service
```bash
sudo systemctl enable --now ananicy-cpp
```

# Leveraging Automated CachyOS Tweaks

#### The CachyOS team provides powerful packages that can automate many of the advanced tweaks. This is a simpler and safer approach than manually setting dozens of system variables :)

# 1. Install CachyOS Optimization Packages
```bash
sudo dnf install cachyos-settings cachyos-ksm-settings scx-scheds
```

# 2. Advanced CPU Scheduler Optimization (SCX)
### This is one of the most impactful tweaks for system responsiveness and gaming performance. We will replace the default Linux CPU scheduler with a specialized one from the `scx-scheds` package we installed earlier.
#### Note: This is an advanced tweak. While it provides significant gains, it changes a core component of the system !!

# Configure the Default Scheduler
### We will set `scx_lavd` as our default scheduler. It is currently the best scheduler for gaming latency
#### on modern cpus, as it prioritizes critical game threads over background noise

**Step 1 — Open the config file:**
```bash
sudo nano /etc/scx_loader/config.toml
```

**Step 2 — Paste this configuration:**
```toml
default_sched = "scx_lavd"
default_mode = "Gaming"

[scheds.scx_lavd]
auto_mode = ["--performance"]
gaming_mode = ["-m", "performance"]
```

> `scx_lavd` is currently the best scheduler for gaming latency on modern CPUs — it prioritizes critical game threads over background tasks.

> **Important:** If you enable `scx_scheds`, you **must** disable `irqbalance`, they conflict and cause micro-stutters.

```bash
sudo systemctl disable --now irqbalance
```

**Step 3 — Enable and start the scheduler:**
```bash
sudo systemctl enable --now scx_loader
```

**Step 4 — Verify it's running:**
```bash
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader \
  org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# Output should show: string "scx_lavd"
```


### Service Management

Disable unnecessary services to free system resources:

```bash
sudo systemctl disable --now \
    bluetooth.service \
    ModemManager.service \
    cups.service \
    avahi-daemon.service \
    chronyd.service \
    NetworkManager-wait-online.service \
    geoclue.service \
    smartd.service \
    upower.service
```

**Tip:** Only disable services you don’t need. Review each service before disabling to avoid breaking functionality you rely on. Also you can search for services manually using internet/some apps

-----

## GRUB Kernel Parameters

### Configuration

Edit `/etc/default/grub` and modify the kernel command line:

```bash
sudo nano /etc/default/grub
```

Add these parameters to `GRUB_CMDLINE_LINUX`:

```bash
GRUB_CMDLINE_LINUX="quiet splash amdgpu.ppfeaturemask=0xffffffff split_lock_detect=off amd_pstate=active page_alloc.shuffle=1 pci=pcie_bus_perf pcie_aspm.policy=performance usbcore.autosuspend=-1 nowatchdog nmi_watchdog=0 zswap.enabled=0"
```
### Parameter Breakdown

| Parameter | Effect | Notes |
|---|---|---|
| `amd_pstate=active` | Enables the EPP driver for millisecond-speed clock boosting | **AMD Zen 3/4/5 only** — Intel users remove this |
| `amdgpu.ppfeaturemask=0xffffffff` | Unlocks voltage control and overclocking limits | Required for LACT / CoreCtrl. **AMD GPU only** |
| `split_lock_detect=off` | Prevents SIGBUS crashes and stutters in games that use split locks (anti-cheat commonly does this) | Safe for everyone |
| `nowatchdog` + `nmi_watchdog=0` | Disables CPU watchdog timers — reduces micro-stutter | Safe for everyone |
| `zswap.enabled=0` | Disables ZSWAP to lower latency | **Only if you have 32 GB+ RAM.** Remove this if you have 4–16 GB |
| `pci=pcie_bus_perf` | Forces PCIe bus to maximum payload size for peak GPU/NVMe bandwidth | Safe for everyone |
| `pcie_aspm.policy=performance` | Disables PCIe power saving states, fixes idle latency spikes | Better than `pcie_aspm=off` |
| `usbcore.autosuspend=-1` | Prevents USB devices from sleeping — fixes input "wake-up" lag | Safe for everyone |
| `page_alloc.shuffle=1` | Randomizes free-list order to improve average-case cache performance | Safe for everyone |

### Experimental — Use at your own risk

| Parameter | Benefit | Warning |
|---|---|---|
| `mitigations=off` | ~3% FPS gain by disabling CPU security patches | **Do NOT use on Zen 4 / Zen 5 (Ryzen 7000/9000)** — can actually lower 1% lows due to branch prediction changes. Recommended only for Zen 1/2/3 and Intel 9th gen or older |

Good for: **Older CPUs (Zen 1/2/3, Intel 9th gen and older).** ~3% fps gain.

Bad for: **Zen 4 / Zen 5 (Ryzen 7000/9000).** Can actually lower 1% low FPS due to branch prediction logic. **Zen 4 and 5 users should NOT use this!**

### Update GRUB Configuration

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

-----

## Advanced System Tweaks

### Memory Management

**Enable systemd-oomd (Out-of-Memory Daemon)(but for games, it may be actually a bad tweak, depends on a system):**

```bash
sudo systemctl enable --now systemd-oomd
```

### Storage Optimization

**Enable SSD TRIM:**

```bash
# Enable automatic TRIM
sudo systemctl enable --now fstrim.timer

# Run manual TRIM
sudo fstrim -v /
```

### CPU Scaling Configuration

**For AMD systems:**

```bash
echo "active" | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

**For Intel systems:**

```bash
echo "passive" | sudo tee /sys/devices/system/cpu/intel_pstate/status
```

### System Limits

**Increase file descriptor limit** in `/etc/security/limits.conf`:

```bash
# Replace 'yourusername' with your actual username
yourusername hard nofile 1048576
yourusername soft nofile 1048576
```

### IRQ Balance (Intel iGPU Users)

If experiencing performance issues with Intel integrated graphics:
(Do NOT enable this if you already enabled scx_lavd!)

```bash
# Check status
sudo systemctl status irqbalance

# Disable if needed
sudo systemctl disable --now irqbalance
```

### I/O Scheduler Configuration

Modern Linux systems use udev rules to configure I/O schedulers per device type. The `elevator=` kernel parameter is deprecated and no longer works on newer kernel versions

**Create udev rule for optimal I/O scheduling:**
```bash
sudo tee /etc/udev/rules.d/60-ioschedulers.rules
# HDD (rotational drives) - use mq-deadline for better performance
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"

# SSD (non-rotational drives) - use mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"

# NVMe SSD - use 'none' for best performance
# NVMe drives have their own advanced queue management and don't benefit from additional scheduling
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

thanks to netarchy for mentioning this new method!
```

**Apply the changes immediately:**
```bash
# Reload udev rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# Verify your current I/O schedulers
cat /sys/block/*/queue/scheduler

# or
grep "" /sys/block/*/queue/scheduler
```

-----

## Gaming Optimizations

### GameMode Installation

GameMode applies system optimizations when gaming:

```bash
sudo dnf install gamemode gamemode-devel

# Verify installation
gamemoded -t
```

**Usage:** Launch games with `gamemoderun` prefix or configure in Steam launch options.

### Windows Games Compatibility

**PortProton** offers excellent compatibility for Windows executables (i use portproton instead of Lutris / Bottles and this is my fav proton executor app!):

```bash
sudo dnf copr enable boria138/portproton
sudo dnf install portproton
```

### Steam Optimizations

If you installed the **CachyOS Kernel**, you have access to NTSync and FSR4 features (amd rdna 4 (rx90**) gpus).

## Launch Command Reference

### Which command should I use?

| My situation | Command to use |
|---|---|
| Game has native HDR (Cyberpunk, Elden Ring…) | HDR command |
| Game has no HDR, I don't want fake HDR | No-HDR command |
| Game has no HDR, I want Auto HDR (like Windows) | SDR→HDR command |

---

### Environment Variables & Wrappers

| Variable | What it does |
|---|---|
| `MANGOHUD_CONFIG="fps_limit=277,no_display"` | Configures MangoHud silently — the frame limiter runs in the background with no visible overlay |
| `fps_limit=277` | Manual low-latency cap: take your monitor's max Hz and subtract 3 (e.g. 280 Hz → 277). Keeps frames inside your FreeSync range and prevents V-Sync input lag backpressure |
| `mangohud` | Actually injects the limiter layer — without this prefix, the config above does nothing |
| `LD_PRELOAD=""` | Fixes Steam micro-stutters that appear after ~25–30 minutes of play |
| `AMD_USERQ=1` | **(RDNA 3/4 only)** Enables User Queues — lets the game talk directly to GPU hardware queues, bypassing kernel driver overhead. Essentially free CPU performance |
| `PROTON_USE_NTSYNC=1` | Replaces fsync/esync Windows thread emulation with a proper kernel level driver. Significantly reduces cpu overhead in multi-threaded games. **On Fedora 44, the NTSYNC module loads automatically when Steam or Wine is installed** |
| `gamemoderun` | Activates Feral GameMode optimizations for the duration of the game session |

### Gamescope Arguments

| Argument | What it does |
|---|---|
| `gamescope` | Valves micro-compositor, isolates the game window from your desktop, often fixing alt+tab crashes and handling HDR/resolution cleanly |
| `-W 2560 -H 1440` | Internal resolution the game thinks its running at — **set this to your monitor's native resolution** |
| `-r 280` | Refresh rate cap for the Gamescope container — **match this exactly to your monitor's max Hz** (e.g. 144, 165, 280…) |
| `--hdr-enabled` | Unlocks the HDR output pipeline — required for oled/ips monitors to actually enter HDR mode |
| `--force-grab-cursor` | Prevents your mouse from accidentally clicking outside the game onto a second monitor |
| `--adaptive-sync` | Explicitly enables VRR (FreeSync / G Sync) inside the container |
| `--sharpness 2` | Applies CAS sharpening. Scale is **0 = sharpest, 20 = softest** — 2 gives a crisp, detailed image |
| `-f` | Forces the container to display in fullscreen |
| `--` | Separator — tells Gamescope "my flags end here, the game command starts next" |
| `%command%` | Steam automatically replaces this with the actual game executable |

### Auto SDR → HDR Arguments

Only needed for the SDR to HDR command. These sit alongside `--hdr-enabled`:

| Argument | What it does |
|---|---|
| `--hdr-itm-enabled` | Enables itm — the actual "Auto HDR" switch that expands SDR color range to use your display's full capabilities |
| `--hdr-itm-target-nits 1000` | Peak brightness target in nits, set this to your monitor's rated peak. Ensures bright effects (fire, sun, magic) reach full brightness without clipping |
| `--hdr-sdr-content-nits 203` | Paper-white / base brightness for standard textures and UI. 203 is the industry standard — too high blinds you in menus, too low makes the game look dim |

---

### Commands

**Native HDR games (Cyberpunk, Elden Ring, any game with hdr support):**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_USE_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --force-grab-cursor --adaptive-sync -f -- %command%
```

**No HDR:**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_USE_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --force-grab-cursor --adaptive-sync -f -- %command%
```

**No HDR + Auto SDR->HDR (like Windows auto HDR):**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_USE_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --hdr-itm-enabled --hdr-itm-target-nits 1000 --hdr-sdr-content-nits 203 --force-grab-cursor --adaptive-sync --sharpness 2 -f -- %command%
```

> **Remember to adjust** `-W`, `-H`, and `-r` to match your monitors resolution and refresh rate

-----

## Maintenance & Cleanup

### Package Cache Management

**Clean DNF cache:**

```bash
sudo dnf clean all
```

**Clean system journals:**

```bash
# Keep only last 7 days of logs
sudo journalctl --vacuum-time=7d

# Or limit by size (keep only 100MB)
sudo journalctl --vacuum-size=100M
```

### Automated Maintenance

Create a simple maintenance script:

```bash
#!/bin/bash
# Save as ~/maintenance.sh and make executable

echo "Running system maintenance..."

# Update system
sudo dnf upgrade --refresh

# Clean caches
sudo dnf clean all

# Clean old journal entries
sudo journalctl --vacuum-time=7d

# Run TRIM on SSD
sudo fstrim -v /

echo "Maintenance complete!"
```

-----

## Desktop Environment Recommendations

### Lightweight Alternatives

For maximum performance, consider these lightweight desktop environments (if installing using Minimal iso):

- **Sway** - Wayland-based tiling compositor (i have 700mb on idle with it)
- **i3** - X11 tiling window manager (600mb on idle)
- **Hyprland** - Modern Wayland compositor with animations (900mb on idle)
- **XFCE** - Lightweight traditional desktop
- **LXQt** - Qt-based lightweight desktop

**KDE Plasma Edition:**
KDE Plasma is now an official Fedora edition alongside Workstation (GNOME). 
This means better integration, support, and optimization out of the box.

### Laptop Power Management

**Install Power Optimization Tools:**
```bash
sudo dnf install powertop tlp tlp-rdw

sudo systemctl enable --now tlp

# Configure TLP for gaming/performance mode

sudo nano /etc/tlp.conf

# Set: TLP_DEFAULT_MODE=performance (when plugged in)
```

### Advanced Memory Management

**Configure Swap Behavior:**
```bash
# Reduce swappiness for better performance
echo 'vm.swappiness=150' | sudo tee -a /etc/sysctl.conf

# Improve memory allocation for gaming, etc.
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf

# Increase the maximum number of memory map areas a process can have.
# The default value (`65530`) is too low for many modern games and will cause crashes or launch failures in titles like Hogwarts Legacy, cs2, The Finals, DayZ, and Star Citizen.

echo 'vm.max_map_count=16777216' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

> This is one of the most commonly missing tweaks on fresh Linux installs. Fedora, Ubuntu, and Pop!_OS have all raised this default — if you're on a clean Fedora 44 install it may already be set, but it's worth verifying:
> ```bash
> sysctl vm.max_map_count
> ```
> If the output is `65530`, apply the fix above.
```

**Container/Flatpak Gaming Section**


### Container Gaming Optimization

**Steam Flatpak Optimization:**
```bash
# Install Steam as Flatpak for sandboxing
flatpak install com.valvesoftware.Steam

# Grant necessary permissions for gaming
flatpak override --user --filesystem=~/.local/share/Steam com.valvesoftware.Steam
```


### GNOME Optimizations

If staying with GNOME:

```bash
# Install GNOME tweaks
sudo dnf install gnome-tweaks gnome-extensions-app

# You can disable animations for better performance
gsettings set org.gnome.desktop.interface enable-animations false

# Reduce resource usage
gsettings set org.gnome.shell.overrides workspaces-only-on-primary false
```

-----

## Graphics Driver Optimization

<details>
<summary>🔴 AMD Graphics Optimization</summary>

### AMD GPU Driver Installation

```bash
# Mesa drivers are included by default, ensure latest version
sudo dnf install mesa-vulkan-drivers mesa-vdpau-drivers mesa-va-drivers

# Install ROCm for compute (optional)
sudo dnf install rocm-opencl rocm-smi
```

### AMD Performance Tweaks

```bash
## Optional! Some of these tweaks can reduce your fps or make games unstable!
# Check performance and stability of games after making any changes

sudo nano /etc/environment.d/99-amd-gaming.conf
# Enable GPU Threading (its a really good option for minecraft, but may lower your fps in most of games)
mesa_glthread=true

# RadeonSI optimizations (only if needed!)
RADV_PERFTEST=gpl,nggc,sam,rt
AMD_VULKAN_ICD=RADV

# Video acceleration
VDPAU_DRIVER=radeonsi
LIBVA_DRIVER_NAME=radeonsi

# Enable Resizable BAR (if for some reason rebar does not work for you)
AMD_GPU_ALLOW_RESIZE_BAR=1
```

### AMD GPU Power Management

```bash
# Set performance mode (in terminal):
echo "performance" | sudo tee /sys/class/drm/card*/device/power_dpm_state
echo "high" | sudo tee /sys/class/drm/card*/device/power_profile

# Create persistent service
sudo nano /etc/systemd/system/amd-gpu-performance.service
[Unit]
Description=AMD GPU Performance Mode
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo performance > /sys/class/drm/card*/device/power_dpm_state'
ExecStart=/bin/bash -c 'echo high > /sys/class/drm/card*/device/power_profile'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target


sudo systemctl enable --now amd-gpu-performance.service
```

    
### Enable AMD User Queues (RDNA 3/4 specific)
Allows the game to talk directly to the GPU, bypassing kernel overhead
Or you can simply add it to the game launch options in Steam

```bash
AMD_USERQ=1
```


### AMD Lact (like AMD Adrenalin on Windows, allows configurin mv offset, memory clock, wattage control, fan curve etc.)

```bash
### Since we added `amdgpu.ppfeaturemask=0xffffffff` to GRUB, you can now use LACT to undervolt/overclock your card for stability and higher boost clocks!

# Enable the copr repository:
sudo dnf copr enable ilyaz/LACT

# Install the package (alternative packages: lact-headless, lact-libadwaita):
sudo dnf install lact

# Enable the service:
sudo systemctl enable --now lactd

## As for rx9070xt users, i recommend to start with "Performance Level: Manual", -70mv voltage offset, lock your memory clock on 2750Mhz, set Power Limit to 355W(for overclock, +3-4% fps). For undervolt, set PL to 260-270W(-3% fps).
```

</details>

<details>
<summary>🟢 NVIDIA Graphics Optimization</summary>

## NVIDIA Graphics Optimization for Fedora

> **Comprehensive optimization guide for NVIDIA GPUs on Fedora with Wayland display server**

### NVIDIA System Requirements

**Supported GPUs:**

- GTX 700/900/1000 series and newer (Maxwell, Pascal, Turing, Ampere, Ada Lovelace, Blackwell)
- RTX 20/30/40/50 series with full feature support

**Driver Compatibility:**

- **Best:** NVIDIA 595+ drivers for optimal Wayland support and performance
- **Note:** NVIDIA driver stack is seeing much better Wayland support with its latest drivers

-----

### NVIDIA Driver Installation

#### Method 1: RPM Fusion (Strongly Recommended)

RPM Fusion remains the most reliable method for NVIDIA drivers on Fedora. This approach ensures proper integration with the Wayland display server and system updates.

```bash
# Enable RPM Fusion repositories (if not already enabled)
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Update system packages
sudo dnf update

# Install NVIDIA drivers with Wayland support
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# Install 32-bit compatibility libraries (essential for Steam, Wine, gaming)
sudo dnf install xorg-x11-drv-nvidia-libs.i686

# Install NVIDIA settings utility
sudo dnf install nvidia-settings
```

#### Post-Installation Verification

Understanding what each command tells us helps ensure your system is properly configured for optimal performance.

```bash
# Verify driver installation and check version
nvidia-smi
# This should show your GPU, driver version (595+), and current utilization

# Confirm CUDA support is available
nvidia-smi -q | grep "CUDA Version"
# Essential for AI workloads and some games using GPU compute

# Verify Wayland is using NVIDIA GPU
echo $XDG_SESSION_TYPE
# Should output "wayland"

# Check that GBM backend is working
nvidia-smi --query-gpu=name,driver_version --format=csv
# Confirms proper driver loading
```

#### Enable Wayland for NVIDIA (Essential Step)

Wayland requires specific configuration to work properly with NVIDIA drivers. This step ensures your desktop environment can utilize hardware acceleration.

```bash
# Enable DRM kernel mode setting (required for Wayland)
echo 'options nvidia_drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-drm-modeset.conf

# Enable early loading of NVIDIA modules
echo -e 'nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm' | sudo tee /etc/modules-load.d/nvidia.conf

# Rebuild initramfs to include changes
sudo dracut --force

# Reboot to apply kernel module changes
sudo reboot
```
> **Fedora + RPM Fusion users:** This step is likely **not needed** — the `akmod-nvidia` package enables modeset automatically. Only apply this manually if Wayland fails to start after driver installation, or if you installed the driver via the official `.run` file instead of RPM Fusion.

-----

### NVIDIA Wayland Performance Optimizations

#### 1. Environment Variables for Wayland

These environment variables optimize NVIDIA GPU behavior specifically for Wayland compositors. Unlike X11, Wayland handles many optimizations automatically, but these variables fine-tune performance.

> **Important NVIDIA Wayland Warning**
>
> Do not blindly add all NVIDIA environment variables globally to `/etc/environment`.
> On modern Fedora + RPM Fusion NVIDIA setups, many Wayland-related settings are already handled automatically.
>
> Global NVIDIA variables can break Wayland login, cause black screens, stuttering, or input lag depending on GPU generation, driver version, and desktop environment.
>
> Prefer testing these options per-game first, for example in Steam launch options.

Add to `/etc/environment`:

```bash
# Core NVIDIA Wayland optimizations
#
# Critical Warning for Modern NVIDIA GPUs (RTX 20 series and newer)
#
# Based on user feedback and testing, the following two env variables (`GBM_BACKEND` and `__GLX_VENDOR_LIBRARY_NAME`) can cause severe system-wide input lag, stuttering, and application unresponsiveness on NVIDIA RTX 20, 30, 40, and 50 series gpus
#
# ! Recommendation for RTX 20-series and newer: DO NOT use these variables. Modern nvidia drivers and wayland compositors handle this automatically.
# ! Recommendation for older gpus (GTX 10-Series and older): These variables can still be beneficial for ensuring wayland compatibility on older hardware.
#
# For older cards ONLY (GTX 10-Series and below), you might still need:
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia

# Enable threaded optimizations (improves CPU-GPU parallelism)
__GL_THREADED_OPTIMIZATIONS=1
# Warning: can cause black screens on some RTX cards. Set this per-game instead (see troubleshooting section)
# If you get a black screen after setting this globally:
#    - press Ctrl+Alt+F3 to switch to TTY, log in
#    - sudo nano /etc/environment, remove the line, save (Ctrl+O, Enter, Ctrl+X)
#    - sudo reboot
# Thanks to @lemonadeforlife for pointing out this problem and solution

# Shader compilation caching (reduces stutter in games)
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_PATH=/tmp/nvidia-shader-cache
__GL_SHADER_DISK_CACHE_SIZE=1073741824

# Disable VSync for gaming (reduces input lag)
__GL_SYNC_TO_VBLANK=0

# Enables NVAPI for features like DLSS in Proton
PROTON_ENABLE_NVAPI=1

# Force Proton to use native Wayland rendering path
PROTON_ENABLE_WAYLAND=1

# Fixes Steam stuttering after 25-30+ minutes of play
LD_PRELOAD=""

# Skip shader cache cleanup on launch (prevents stutter spikes)
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1

# The following tweaks are not recommended globally.
# For wl-roots compositors ONLY (Sway, Hyprland) — not needed on GNOME/KDE.
# Note: with driver 595+ and Sway 1.11+ / Hyprland (explicit sync support),
# WLR_NO_HARDWARE_CURSORS may no longer be necessary. Test without it first.
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1
```

#### 2. Kernel Module Parameters (use if you know what you're doing)

Modern NVIDIA drivers benefit from specific kernel parameters that can improve performance
Test each option if it works in ur case and with you hardware

Create `/etc/modprobe.d/nvidia-power-management.conf`:

```bash
# Enable modern power management features
options nvidia NVreg_DynamicPowerManagement=0x02

# Enable Page Attribute Table (improves memory performance)
options nvidia NVreg_UsePageAttributeTable=1

# Enable ResizableBAR support (RTX 30/40/50 series)
options nvidia NVreg_EnableResizableBar=1

# Preserve video memory during suspend
options nvidia NVreg_PreserveVideoMemoryAllocations=1

# Enable stream memory operations (required for some workloads)
options nvidia NVreg_EnableStreamMemOPs=1
```

#### 3. GNOME Wayland Specific Settings

GNOME on Wayland requires particular attention to achieve optimal NVIDIA performance. These settings address common issues with the GNOME compositor.

```bash
Enable Variable Refresh Rate (VRR):

# Fedora 44 with GNOME 50: VRR is now stable — enable it directly in Settings → Displays
# Or via command line:
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"
```

> **Fedora 44 note:** With GNOME 50, VRR is no longer experimental. You can enable it directly in **Settings → Displays → Variable Refresh Rate** without any gsettings command. The command above still works but is no longer required.

# Configure GNOME for gaming performance
gsettings set org.gnome.shell.extensions.dash-to-dock click-action 'cycle-windows'
gsettings set org.gnome.desktop.interface enable-animations false

# Set scaling factor for high-DPI displays (adjust as needed)
gsettings set org.gnome.desktop.interface scaling-factor 1
```

#### 4. KDE Plasma Wayland Configuration

KDE Plasma has excellent Wayland support and works particularly well with NVIDIA drivers when properly configured.

```bash
# Enable variable refresh rate support (or using kde/gnome settings which is better)
kwriteconfig6 --file kwinrc --group Compositing --key VariableRefreshRate true

# Optimize compositor settings for gaming
kwriteconfig6 --file kwinrc --group Compositing --key LatencyPolicy Low
kwriteconfig6 --file kwinrc --group Compositing --key RenderTimeEstimator 1

# Restart KWin to apply changes
qdbus org.kde.KWin /KWin reconfigure
```

-----

### Gaming-Specific NVIDIA Optimizations

#### 1. Steam Launch Options for Wayland

Steam gaming on Wayland requires specific launch parameters to ensure games use the NVIDIA GPU properly and achieve optimal performance.

**For native Linux games:**

```bash
# Basic optimization with GameMode
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 %command%

# Enhanced performance for competitive gaming
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 %command%
```

**For Proton/Wine games:**

```bash
# Standard Proton optimization
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Advanced optimization with GPL pipeline compilation (replaces old dxvk-async)
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# For games requiring maximum performance
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%
```

#### 2. Lutris Gaming Optimization

Lutris provides excellent integration with NVIDIA drivers on Wayland. Configure these settings for optimal gaming performance.

```bash
# Install Lutris with NVIDIA support
sudo dnf install lutris wine

# Configure Lutris environment variables (in Lutris preferences)
# Add these to System Options → Environment variables:
__GL_THREADED_OPTIMIZATIONS=1
__GL_SHADER_DISK_CACHE=1
PROTON_ENABLE_NVAPI=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
```

#### 3. GameMode Integration

GameMode automatically applies system optimizations during gaming. While it works well out of the box, you can fine-tune its behavior.

```bash
# Install GameMode
sudo dnf install gamemode

# Configure GameMode for NVIDIA optimization
sudo nano /etc/gamemode.ini
[general]
renice=10
ioprio=0

[gpu]
# Warning: attempts to overclock GPU via nvidia-settings. Test carefully.
apply_gpu_optimisations=accept-responsibility
gpu_device=0
```

-----

### Advanced NVIDIA Wayland Tweaks

#### 1. Variable Refresh Rate (VRR) Support

Modern NVIDIA drivers support variable refresh rate on Wayland, providing smoother gaming experiences with compatible monitors.

```bash
# Enable VRR in GNOME
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"
# Or in Display settings (if supported)

# Verify VRR is working
sudo dnf install drm_info
drm_info | grep -i vrr
```

#### 2. HDR Support (Experimental)

High Dynamic Range support is gradually improving on Wayland with NVIDIA drivers. These settings enable experimental HDR functionality.

```bash
# Unverified parameter — test before applying, not in official NVIDIA docs
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf
```

#### 3. Performance Monitoring and Tuning

Effective performance monitoring helps identify bottlenecks and verify that optimizations are working correctly.

```bash
# Install monitoring tools
sudo dnf install nvtop mangohud goverlay

# Create monitoring script for gaming sessions
sudo nano /usr/local/bin/nvidia-monitor.sh
#!/bin/bash
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
nvidia-smi dmon -s pucvmet

sudo chmod +x /usr/local/bin/nvidia-monitor.sh
```

-----

### Power Management and Thermal Optimization

#### Advanced Power Management (apply only if you know what youre doing)

Proper power management ensures consistent performance while preventing unnecessary power consumption during idle periods.

```bash
# Configure advanced power management
echo 'options nvidia NVreg_DynamicPowerManagement=0x02' | sudo tee -a /etc/modprobe.d/nvidia-power.conf

# Enable runtime power management for laptops
sudo nano /etc/udev/rules.d/80-nvidia-pm.rules
# Enable runtime PM for NVIDIA VGA/3D controller devices
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
```

-----

### NVIDIA Wayland Troubleshooting

#### Common Issues and Modern Solutions

Understanding how to diagnose and resolve issues ensures optimal performance and system stability.

**1. Wayland Session Not Starting with NVIDIA:**

This is the most common issue when transitioning from X11 to Wayland with NVIDIA drivers.

```bash
### NVIDIA Black Screen on Boot

**Problem:** System shows black screen after boot with NVIDIA drivers

**Cause:** The `__GL_THREADED_OPTIMIZATIONS=1` environment variable in `/etc/environment` can cause display initialization issues on some RTX GPUs

**Solution:**
1. Boot into recovery mode (hold Shift during boot to access GRUB menu)
2. Edit `/etc/environment` and remove or comment out:
   ```bash
   # __GL_THREADED_OPTIMIZATIONS=1

# Verify kernel module parameters are correct
cat /etc/modprobe.d/nvidia-drm-modeset.conf
# Should contain: options nvidia_drm modeset=1 fbdev=1

# Check if DRM modeset is enabled
cat /sys/module/nvidia_drm/parameters/modeset
# Should output: Y

# Rebuild initramfs and reboot if necessary
sudo dracut --force
sudo reboot

# Verify Wayland session after reboot
echo $XDG_SESSION_TYPE
# Should output: wayland
```

**2. Poor Gaming Performance Despite Good Hardware:**

Performance issues often stem from incorrect GPU usage or power management settings.

```bash
# Verify GPU is being utilized
nvidia-smi dmon -s pucvmet -c 10

# Check for power limiting
nvidia-smi --query-gpu=power.limit,power.draw --format=csv
# Power draw should approach power limit during gaming

# Monitor GPU clocks during gaming
watch -n 1 'nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv'
# Clocks should reach maximum values during gaming
```

**3. Screen Tearing or Stuttering:**

Modern Wayland compositors handle tearing better than X11, but some configuration may be needed.

```bash
# For GNOME, ensure VRR is enabled
gsettings get org.gnome.mutter experimental-features
# Should include 'variable-refresh-rate'

# For games, disable VSync in-game and use compositor VSync
# Add to Steam launch options:
__GL_SYNC_TO_VBLANK=0 %command%
```

**4. High Idle Power Consumption:**

Preventing unnecessary power draw during idle periods improves battery life and reduces heat.

```bash
# Enable runtime power management
echo 'auto' | sudo tee /sys/bus/pci/devices/0000:*/power/control

# Verify power management is working
cat /sys/bus/pci/devices/0000:*/power/runtime_status
# Should show 'suspended' for idle GPU

# Monitor idle power consumption
nvidia-smi --query-gpu=power.draw --format=csv --loop=1
```

-----

### Performance Monitoring and Benchmarking

#### Comprehensive Monitoring Setup

Effective monitoring helps optimize performance and identify potential issues before they impact gaming or work performance.

```bash
# Install comprehensive monitoring suite
sudo dnf install nvtop btop mangohud goverlay
```

#### Gaming Performance Overlay

MangoHud provides real-time performance metrics during gaming sessions.

```bash
# Configure MangoHud for optimal display
mkdir -p ~/.config/MangoHud

nano ~/.config/MangoHud/MangoHud.conf
# GPU and CPU information
gpu_stats
cpu_stats
gpu_temp
cpu_temp

# Frame rate and timing
fps
frametime
frame_timing

# Memory usage
vram
ram

# Position and appearance
position=top-left
font_size=22
alpha=0.8

# Limit logging to prevent performance impact
log_duration=60
```

-----

### Additional Resources

**Official NVIDIA Documentation:**
- [NVIDIA Linux Driver Installation Guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
- [CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

**Fedora-Specific Resources:**
- [RPM Fusion NVIDIA Guide](https://rpmfusion.org/Howto/NVIDIA)

**Wayland and Gaming:**
- [Gaming on Linux with NVIDIA](https://www.gamingonlinux.com/)
- [MangoHud Documentation](https://github.com/flightlessmango/MangoHud)

-----

</details>

-----

## Monitoring & Verification

### Performance Monitoring Tools

```bash
# Install monitoring tool
sudo dnf install btop

# For detailed system information
sudo dnf install hardinfo2
```

### Benchmark Tools

```bash
# Gaming benchmarks
sudo dnf install glmark2 unigine-superposition

# System benchmarks
sudo dnf install sysbench stress-ng
```

-----

## Troubleshooting

### Common Issues

**1. Boot Issues After Kernel Parameters:**

- Boot with previous kernel from GRUB menu
- Remove problematic parameters from `/etc/default/grub`
- Regenerate GRUB config

**2. Graphics Issues:**

- Check driver installation: `lspci -k | grep -A 2 -E "(VGA|3D)"`
- Verify correct driver loading: `lsmod | grep -E "(amdgpu|nvidia|i915)"`

**3. Performance Regression:**

- Monitor system resources: `htop`, `iotop`
- Check for thermal throttling: `watch sensors`
- Verify services status: `systemctl list-units --failed`

### Recovery Commands

```bash
# Reset GRUB to defaults
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Reset SELinux context (if re-enabling SELinux)
sudo restorecon -R /

# Check system integrity
sudo dnf check
sudo rpm -Va
```

-----

## Expected Performance Gains

Based on testing, users can expect:

- **Boot Time:** 10-20% improvement
- **Gaming Performance:** 5-15% FPS increase
- **System Responsiveness:** Significantly reduced input lag
- **Memory Usage:** 5-15% reduction in idle RAM usage
- **Storage Performance:** Improved SSD performance with trim

-----

## Русская версия | (Russian Translation)

<details>
<summary>👀 Нажмите для просмотра русской версии</summary>
    
# Гайд по оптимизации производительности и гейминга Fedora Linux 44

> **Полный гайд по оптимизации Fedora 44 для максимальной производительности, игр, повседневного использования и т.д. | от winterofhell**

## Быстрая навигация

| Настройка и ядро | Система и гейминг | Ресурсы |
|---|---|---|
| [**Информация о системе**](#информация-о-системе) | [**Продвинутые системные твики**](#продвинутые-системные-твики) | [**Мониторинг и проверка**](#мониторинг-и-проверка) |
| [**Начальная настройка**](#начальная-настройка-и-подготовка) | [**Оптимизация для гейминга**](#оптимизация-для-гейминга) | [**Решение проблем**](#решение-проблем) |
| [**Оптимизация ядра**](#оптимизация-ядра) | [**Обслуживание и очистка**](#обслуживание-и-очистка) | [**Английская версия**](#fedora-linux-44-performance--gaming-optimization-guide) |
| [**Параметры ядра GRUB**](#параметры-ядра-grub) | [**Оптимизация графических драйверов**](#оптимизация-графических-драйверов) | |

## Информация о системе

**Тестовое окружение:**

- **Период:** 14 октября 2024 — 7 июля 2026
- **Дистрибутив:** Fedora 44
- **Дополнительное тестирование:** системы с NVIDIA и AMD GPU
- **Эти оптимизации могут работать и на других дистрибутивах, но я это не гарантирую. Всё нужно тестировать самому. Примерно 90% этих твиков работают на Arch и NixOS — проверял лично :)**

**Железо, на котором тестировалось:**

- **Первое:** Ryzen 5 5500U, 20 GB DDR4, RX550X/RX Vega 7, NVMe SSD
- **Второе:** Ryzen 5 5600, 16 GB DDR4, GTX 1060, SATA SSD
- **Третье:** Ryzen 5 7500F / Ryzen 7 9850X3D, 32 GB DDR5, RX 9070 XT, NVMe SSD

-----

## Начальная настройка и подготовка

### 1. Минимальная установка

Для максимальной производительности лучше начинать с **Fedora Minimal ISO**. Так в системе не будет лишних пакетов и сервисов, которые едят ресурсы. Это не обязательное условие, но хороший старт.

### 2. Включение RPM Fusion

RPM Fusion даёт нужные мультимедийные кодеки и проприетарные драйверы:

```bash
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

**Официальный гайд:** [RPM Fusion Configuration](https://rpmfusion.org/Configuration)

### 3. Настройка SELinux (опционально)

**Предупреждение по безопасности:** отключение SELinux снижает безопасность, но может чуть ускорить систему. Делайте это только если понимаете последствия. Лично мне на SELinux всё равно, поэтому я его обычно отключаю.

**Временно отключить до перезагрузки:**

```bash
sudo setenforce 0
```

**Отключить навсегда, нужна перезагрузка:**

```bash
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# Для применения изменений нужна перезагрузка
```

### 4. Обновление системы

Начинайте с полностью обновлённой системы:

```bash
sudo dnf upgrade --refresh

# Установка обновлений прошивок, включая микрокод CPU
sudo dnf install linux-firmware intel-ucode amd-ucode
```

-----

## Оптимизация ядра

### Установка ядра CachyOS

Ядро CachyOS даёт заметный прирост отзывчивости системы и производительности в играх.

**Требование:** CPU должен поддерживать набор инструкций x86_64_v3 !!

```bash
# Проверка поддержки CPU
# Запустите команду ниже

/lib64/ld-linux-x86-64.so.2 --help | grep "(supported, searched)"

# Если x86_64_v3 не определяется, НЕ ставьте это ядро.
# Если определяется только x86_64_v2, можно использовать LTS-ядро.
```

```bash
# Добавить COPR-репозиторий CachyOS
sudo dnf copr enable bieszczaders/kernel-cachyos
sudo dnf copr enable bieszczaders/kernel-cachyos-addons

# Установить ядро CachyOS
sudo dnf install kernel-cachyos kernel-cachyos-devel

# Только для x86_64_v2, то есть старых CPU:
sudo dnf install kernel-cachyos-lts kernel-cachyos-lts-devel-matched
```

### Сделать ядро CachyOS ядром по умолчанию

После установки Fedora может продолжать грузить другое ядро. Через `grubby` можно проверить записи и выставить нужную по умолчанию.

```bash
# Показать все установленные записи ядер
sudo grubby --info=ALL | grep -E "^kernel|^index|^title"

# Проверить текущее ядро по умолчанию
sudo grubby --default-kernel
sudo grubby --default-index

# Найдите запись CachyOS и выставьте её по индексу:

sudo grubby --set-default-index=<INDEX_NUMBER>

# Пример:

sudo grubby --set-default-index=0

# Перезагрузитесь и проверьте:

uname -r

# В выводе должно быть cachyos, если система загружена на ядре CachyOS.
```

**Подробнее:** [CachyOS Kernel Installation](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/)

-----

## Оптимизация системных сервисов

### Установка Ananicy-cpp

Ananicy-cpp автоматически управляет приоритетами процессов и снижает задержки:

# Установить зависимости для сборки
```bash
sudo dnf group install "Development Tools"
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel make automake gcc gcc-c++
```

# Склонировать и собрать
```bash
git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
```

# Включить сервис
```bash
sudo systemctl enable --now ananicy-cpp
```

# Автоматические твики CachyOS

#### Команда CachyOS даёт пакеты, которые автоматизируют многие продвинутые твики. Это проще и безопаснее, чем руками менять десятки системных параметров :)

# 1. Установить пакеты оптимизации CachyOS
```bash
sudo dnf install cachyos-settings cachyos-ksm-settings scx-scheds
```

# 2. Продвинутая оптимизация планировщика CPU (SCX)
### Это один из самых сильных твиков для отзывчивости системы и FPS/frametime в играх. Мы заменим стандартный планировщик Linux на специализированный из пакета `scx-scheds`, который уже установили выше.
#### Важно: это продвинутый твик. Он может дать заметный профит, но меняет базовый компонент системы !!

# Настройка планировщика по умолчанию
### Ставим `scx_lavd` как планировщик по умолчанию. Сейчас это один из лучших вариантов для игровой задержки
#### на современных CPU: он приоритизирует важные игровые потоки над фоновым шумом.

**Шаг 1 — открыть конфиг:**
```bash
sudo nano /etc/scx_loader/config.toml
```

**Шаг 2 — вставить конфигурацию:**
```toml
default_sched = "scx_lavd"
default_mode = "Gaming"

[scheds.scx_lavd]
auto_mode = ["--performance"]
gaming_mode = ["-m", "performance"]
```

> `scx_lavd` сейчас один из лучших планировщиков для низкой игровой задержки на современных CPU — он отдаёт приоритет критичным игровым потокам вместо фоновых задач.

> **Важно:** если включаете `scx_scheds`, обязательно отключите `irqbalance`. Они конфликтуют и могут вызывать микрофризы.

```bash
sudo systemctl disable --now irqbalance
```

**Шаг 3 — включить и запустить планировщик:**
```bash
sudo systemctl enable --now scx_loader
```

**Шаг 4 — проверить, что он работает:**
```bash
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader \
  org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# В выводе должно быть: string "scx_lavd"
```

### Управление сервисами

Отключите ненужные сервисы, чтобы освободить ресурсы:

```bash
sudo systemctl disable --now \
    bluetooth.service \
    ModemManager.service \
    cups.service \
    avahi-daemon.service \
    chronyd.service \
    NetworkManager-wait-online.service \
    geoclue.service \
    smartd.service \
    upower.service
```

**Совет:** отключайте только то, чем не пользуетесь. Перед отключением проверьте каждый сервис, чтобы не сломать нужный функционал. При необходимости ищите описание сервиса в интернете или через системные утилиты.

-----

## Параметры ядра GRUB

### Настройка

Откройте `/etc/default/grub` и измените командную строку ядра:

```bash
sudo nano /etc/default/grub
```

Добавьте параметры в `GRUB_CMDLINE_LINUX`:

```bash
GRUB_CMDLINE_LINUX="quiet splash amdgpu.ppfeaturemask=0xffffffff split_lock_detect=off amd_pstate=active page_alloc.shuffle=1 pci=pcie_bus_perf pcie_aspm.policy=performance usbcore.autosuspend=-1 nowatchdog nmi_watchdog=0 zswap.enabled=0"
```

### Что делают параметры

| Параметр | Эффект | Примечания |
|---|---|---|
| `amd_pstate=active` | Включает EPP-драйвер для быстрого буста частот с задержкой в миллисекунды | **Только AMD Zen 3/4/5** — пользователям Intel удалить |
| `amdgpu.ppfeaturemask=0xffffffff` | Разблокирует управление напряжением и лимиты разгона | Нужно для LACT / CoreCtrl. **Только AMD GPU** |
| `split_lock_detect=off` | Убирает SIGBUS-краши и фризы в играх, которые используют split locks. Античиты часто делают именно это | Безопасно для всех |
| `nowatchdog` + `nmi_watchdog=0` | Отключает CPU watchdog-таймеры и уменьшает микрофризы | Безопасно для всех |
| `zswap.enabled=0` | Отключает ZSWAP ради меньшей задержки | **Только если у вас 32 GB+ RAM.** При 4–16 GB удалите параметр |
| `pci=pcie_bus_perf` | Принудительно выставляет максимальный payload size для PCIe, чтобы выжать максимум из GPU/NVMe | Безопасно для всех |
| `pcie_aspm.policy=performance` | Отключает энергосберегающие состояния PCIe и убирает скачки задержки в простое | Лучше, чем `pcie_aspm=off` |
| `usbcore.autosuspend=-1` | Не даёт USB-устройствам засыпать, убирает задержку «пробуждения» ввода | Безопасно для всех |
| `page_alloc.shuffle=1` | Рандомизирует порядок free-list для лучшей средней производительности кэша | Безопасно для всех |

### Экспериментально — на свой риск

| Параметр | Плюс | Предупреждение |
|---|---|---|
| `mitigations=off` | Примерно +3% FPS за счёт отключения CPU security patches | **Не используйте на Zen 4 / Zen 5 (Ryzen 7000/9000)** — может ухудшить 1% lows из-за изменений branch prediction. Имеет смысл в основном для Zen 1/2/3 и Intel 9-го поколения или старше |

Хорошо для: **старых CPU — Zen 1/2/3, Intel 9-го поколения и старше.** Примерно +3% FPS.

Плохо для: **Zen 4 / Zen 5 — Ryzen 7000/9000.** Может снизить 1% low FPS из-за логики предсказания ветвлений. **Пользователям Zen 4/5 лучше не использовать это.**

### Обновление конфигурации GRUB

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

-----

## Продвинутые системные твики

### Управление памятью

**Включить systemd-oomd, демон Out-of-Memory. Для игр это может быть плохим твиком — зависит от системы:**

```bash
sudo systemctl enable --now systemd-oomd
```

### Оптимизация накопителя

**Включить TRIM для SSD:**

```bash
# Включить автоматический TRIM
sudo systemctl enable --now fstrim.timer

# Запустить TRIM вручную
sudo fstrim -v /
```

### Настройка CPU scaling

**Для AMD:**

```bash
echo "active" | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

**Для Intel:**

```bash
echo "passive" | sudo tee /sys/devices/system/cpu/intel_pstate/status
```

### Системные лимиты

**Увеличить лимит файловых дескрипторов** в `/etc/security/limits.conf`:

```bash
# Замените 'yourusername' на свой реальный username
yourusername hard nofile 1048576
yourusername soft nofile 1048576
```

### IRQ Balance для пользователей Intel iGPU

Если есть проблемы с производительностью на встроенной графике Intel:
**Не включайте это, если уже включили `scx_lavd`!**

```bash
# Проверить статус
sudo systemctl status irqbalance

# Отключить при необходимости
sudo systemctl disable --now irqbalance
```

### Настройка I/O scheduler

Современные Linux-системы настраивают I/O schedulers через udev-правила под каждый тип накопителя. Параметр ядра `elevator=` устарел и на новых ядрах больше не работает.

**Создайте udev-правило для нормального I/O scheduling:**
```bash
sudo tee /etc/udev/rules.d/60-ioschedulers.rules
# HDD, то есть вращающиеся диски — mq-deadline для лучшей производительности
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"

# SSD, то есть невращающиеся диски — mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"

# NVMe SSD — none для лучшей производительности
# У NVMe уже есть продвинутое управление очередями, дополнительный scheduler обычно не нужен
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

спасибо netarchy за этот метод!
```

**Применить изменения сразу:**
```bash
# Перезагрузить udev-правила
sudo udevadm control --reload-rules
sudo udevadm trigger

# Проверить текущие I/O schedulers
cat /sys/block/*/queue/scheduler

# или
grep "" /sys/block/*/queue/scheduler
```

-----

## Оптимизация для гейминга

### Установка GameMode

GameMode применяет системные оптимизации во время игры:

```bash
sudo dnf install gamemode gamemode-devel

# Проверить установку
gamemoded -t
```

**Использование:** запускайте игры с префиксом `gamemoderun` или добавьте его в параметры запуска Steam.

### Совместимость с Windows-играми

**PortProton** даёт отличную совместимость для Windows exe. Я использую PortProton вместо Lutris / Bottles, это мой любимый лаунчер Proton:

```bash
sudo dnf copr enable boria138/portproton
sudo dnf install portproton
```

### Оптимизация Steam

Если вы поставили **ядро CachyOS**, у вас есть доступ к NTSync и FSR4 — для AMD RDNA 4 GPU, например RX 90**.

## Справка по параметрам запуска

### Какую команду использовать?

| Ситуация | Команда |
|---|---|
| У игры есть нативный HDR, например Cyberpunk или Elden Ring | HDR-команда |
| У игры нет HDR, фейковый HDR не нужен | No-HDR команда |
| У игры нет HDR, но хочется Auto HDR как в Windows | SDR→HDR команда |

---

### Переменные окружения и обёртки

| Переменная | Что делает |
|---|---|
| `MANGOHUD_CONFIG="fps_limit=277,no_display"` | Настраивает MangoHud тихо: лимитер кадров работает в фоне, оверлей не показывается |
| `fps_limit=277` | Ручной low-latency лимит: берём максимальную герцовку монитора и вычитаем 3, например 280 Hz → 277. Так кадры остаются внутри диапазона FreeSync и не создают input-lag через V-Sync backpressure |
| `mangohud` | Фактически внедряет лимитер. Без этого префикса конфиг выше ничего не делает |
| `LD_PRELOAD=""` | Фиксит микрофризы Steam, которые появляются примерно через 25–30 минут игры |
| `AMD_USERQ=1` | **Только RDNA 3/4.** Включает User Queues: игра общается напрямую с аппаратными очередями GPU, обходя часть overhead в kernel driver. По сути бесплатная CPU-производительность |
| `PROTON_USE_NTSYNC=1` | Заменяет fsync/esync-эмуляцию Windows-потоков нормальным kernel-level драйвером. Сильно снижает CPU overhead в многопоточных играх. **На Fedora 44 модуль NTSYNC грузится автоматически, если установлен Steam или Wine** |
| `gamemoderun` | Включает оптимизации Feral GameMode на время игровой сессии |

### Аргументы Gamescope

| Аргумент | Что делает |
|---|---|
| `gamescope` | Микрокомпозитор Valve. Изолирует окно игры от рабочего стола, часто чинит Alt+Tab-краши и нормально обрабатывает HDR/разрешение |
| `-W 2560 -H 1440` | Внутреннее разрешение, которое видит игра. **Поставьте нативное разрешение монитора** |
| `-r 280` | Лимит частоты обновления контейнера Gamescope. **Ставьте ровно максимум вашего монитора**: 144, 165, 280 и т.д. |
| `--hdr-enabled` | Включает HDR pipeline. Нужно, чтобы OLED/IPS монитор реально перешёл в HDR-режим |
| `--force-grab-cursor` | Не даёт мышке случайно кликать за пределами игры на втором мониторе |
| `--adaptive-sync` | Явно включает VRR — FreeSync / G-Sync — внутри контейнера |
| `--sharpness 2` | Включает CAS sharpening. Шкала: **0 = максимально резко, 20 = максимально мягко**. Значение 2 даёт чёткую картинку без мыла |
| `-f` | Принудительно открывает контейнер в fullscreen |
| `--` | Разделитель: говорит Gamescope «мои флаги закончились, дальше команда игры» |
| `%command%` | Steam автоматически заменяет это на настоящий исполняемый файл игры |

### Аргументы Auto SDR → HDR

Нужны только для команды SDR → HDR. Они используются вместе с `--hdr-enabled`:

| Аргумент | Что делает |
|---|---|
| `--hdr-itm-enabled` | Включает ITM — фактический переключатель Auto HDR, который расширяет SDR-цвета под возможности дисплея |
| `--hdr-itm-target-nits 1000` | Целевая пиковая яркость в нитах. Ставьте под пиковую яркость монитора. Так огонь, солнце и яркие эффекты не клипуются и не выглядят тускло |
| `--hdr-sdr-content-nits 203` | Paper-white / базовая яркость для обычных текстур и UI. 203 — стандартное значение: выше может слепить в меню, ниже делает картинку тусклой |

---

### Команды

**Нативный HDR — Cyberpunk, Elden Ring и любые игры с HDR:**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_USE_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --force-grab-cursor --adaptive-sync -f -- %command%
```

**Без HDR:**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_USE_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --force-grab-cursor --adaptive-sync -f -- %command%
```

**Без HDR + Auto SDR→HDR как Windows Auto HDR:**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_USE_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --hdr-itm-enabled --hdr-itm-target-nits 1000 --hdr-sdr-content-nits 203 --force-grab-cursor --adaptive-sync --sharpness 2 -f -- %command%
```

> **Не забудьте поменять** `-W`, `-H` и `-r` под разрешение и герцовку вашего монитора.

-----

## Обслуживание и очистка

### Управление кэшем пакетов

**Очистить кэш DNF:**

```bash
sudo dnf clean all
```

**Очистить системные журналы:**

```bash
# Оставить только логи за последние 7 дней
sudo journalctl --vacuum-time=7d

# Или ограничить размером, например 100 MB
sudo journalctl --vacuum-size=100M
```

### Автоматическое обслуживание

Простой скрипт обслуживания:

```bash
#!/bin/bash
# Сохраните как ~/maintenance.sh и сделайте исполняемым

echo "Запуск обслуживания системы..."

# Обновить систему
sudo dnf upgrade --refresh

# Очистить кэши
sudo dnf clean all

# Очистить старые записи журнала
sudo journalctl --vacuum-time=7d

# Запустить TRIM на SSD
sudo fstrim -v /

echo "Обслуживание завершено!"
```

-----

## Рекомендации по окружению рабочего стола

### Лёгкие альтернативы

Для максимальной производительности можно использовать лёгкие окружения, особенно если ставили систему через Minimal ISO:

- **Sway** — Wayland tiling compositor, у меня около 700 MB в idle
- **i3** — X11 tiling window manager, около 600 MB в idle
- **Hyprland** — современный Wayland compositor с анимациями, около 900 MB в idle
- **XFCE** — лёгкий классический desktop
- **LXQt** — лёгкий desktop на Qt

**KDE Plasma Edition:**
KDE Plasma теперь официальная редакция Fedora наряду с Workstation GNOME. Это даёт лучшую интеграцию, поддержку и оптимизацию из коробки.

### Управление питанием ноутбука

**Установить инструменты энергосбережения:**
```bash
sudo dnf install powertop tlp tlp-rdw

sudo systemctl enable --now tlp

# Настроить TLP под gaming/performance mode

sudo nano /etc/tlp.conf

# Поставить: TLP_DEFAULT_MODE=performance при питании от сети
```

### Продвинутое управление памятью

**Настроить поведение swap:**
```bash
# Настроить swappiness для лучшей производительности
echo 'vm.swappiness=150' | sudo tee -a /etc/sysctl.conf

# Улучшить поведение кэша VFS для игр и общей отзывчивости
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf

# Увеличить максимальное число memory map areas на процесс.
# Дефолтное значение (`65530`) слишком низкое для многих современных игр и вызывает краши или проблемы запуска
# в Hogwarts Legacy, CS2, The Finals, DayZ, Star Citizen и других.

echo 'vm.max_map_count=16777216' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

> Это один из самых частых недостающих твиков на свежих Linux-установках. Fedora, Ubuntu и Pop!_OS уже подняли дефолт. На чистой Fedora 44 значение может быть уже нормальным, но лучше проверить:
> ```bash
> sysctl vm.max_map_count
> ```
> Если вывод `65530`, примените фикс выше.

**Раздел про Container/Flatpak gaming**

### Оптимизация игр в контейнерах

**Оптимизация Steam Flatpak:**
```bash
# Установить Steam как Flatpak для sandboxing
flatpak install com.valvesoftware.Steam

# Выдать нужные права для игр
flatpak override --user --filesystem=~/.local/share/Steam com.valvesoftware.Steam
```

### Оптимизация GNOME

Если остаётесь на GNOME:

```bash
# Установить GNOME Tweaks
sudo dnf install gnome-tweaks gnome-extensions-app

# Можно отключить анимации ради лучшей отзывчивости
gsettings set org.gnome.desktop.interface enable-animations false

# Снизить расход ресурсов
gsettings set org.gnome.shell.overrides workspaces-only-on-primary false
```

-----

## Оптимизация графических драйверов

<details>
<summary>🔴 Оптимизация AMD-графики</summary>

### Установка драйвера AMD GPU

```bash
# Mesa-драйверы уже идут по умолчанию, но убедитесь, что стоит свежая версия
sudo dnf install mesa-vulkan-drivers mesa-vdpau-drivers mesa-va-drivers

# ROCm для compute, опционально
sudo dnf install rocm-opencl rocm-smi
```

### Твики производительности AMD

```bash
## Опционально! Некоторые твики могут снизить FPS или сделать игры нестабильными.
# После любых изменений проверяйте производительность и стабильность.

sudo nano /etc/environment.d/99-amd-gaming.conf
# Включить GPU threading. Хорошо для Minecraft, но в большинстве игр может снизить FPS.
mesa_glthread=true

# RadeonSI-оптимизации, только если нужно!
RADV_PERFTEST=gpl,nggc,sam,rt
AMD_VULKAN_ICD=RADV

# Видео acceleration
VDPAU_DRIVER=radeonsi
LIBVA_DRIVER_NAME=radeonsi

# Включить Resizable BAR, если по какой-то причине ReBAR не работает
AMD_GPU_ALLOW_RESIZE_BAR=1
```

### Управление питанием AMD GPU

```bash
# Выставить performance mode в терминале:
echo "performance" | sudo tee /sys/class/drm/card*/device/power_dpm_state
echo "high" | sudo tee /sys/class/drm/card*/device/power_profile

# Создать постоянный systemd-сервис
sudo nano /etc/systemd/system/amd-gpu-performance.service
[Unit]
Description=AMD GPU Performance Mode
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo performance > /sys/class/drm/card*/device/power_dpm_state'
ExecStart=/bin/bash -c 'echo high > /sys/class/drm/card*/device/power_profile'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target


sudo systemctl enable --now amd-gpu-performance.service
```

### Включить AMD User Queues, только RDNA 3/4

Позволяет игре напрямую общаться с GPU, обходя часть kernel overhead. Можно просто добавить в параметры запуска игры в Steam:

```bash
AMD_USERQ=1
```

### AMD LACT

LACT — аналог AMD Adrenalin на Windows: даёт voltage offset, memory clock, wattage control, fan curve и т.д.

```bash
### Так как мы добавили `amdgpu.ppfeaturemask=0xffffffff` в GRUB, теперь можно использовать LACT для андервольта/разгона карты, лучшей стабильности и более высоких boost clocks.

# Включить COPR-репозиторий:
sudo dnf copr enable ilyaz/LACT

# Установить пакет. Альтернативы: lact-headless, lact-libadwaita
sudo dnf install lact

# Включить сервис:
sudo systemctl enable --now lactd

## Для RX 9070 XT рекомендую начать с "Performance Level: Manual", voltage offset -70 mV, залочить memory clock на 2750 MHz и поставить Power Limit 355 W для overclock, это примерно +3–4% FPS. Для undervolt ставьте PL 260–270 W, потеря около 3% FPS.
```

</details>

<details>
<summary>🟢 Оптимизация NVIDIA-графики</summary>

## Оптимизация NVIDIA-графики для Fedora

> **Полный гайд по оптимизации NVIDIA GPU на Fedora с Wayland display server**

### Системные требования NVIDIA

**Поддерживаемые GPU:**

- GTX 700/900/1000 series и новее: Maxwell, Pascal, Turing, Ampere, Ada Lovelace, Blackwell
- RTX 20/30/40/50 series с полной поддержкой фич

**Совместимость драйвера:**

- **Лучше всего:** NVIDIA 595+ для оптимальной поддержки Wayland и производительности
- **Примечание:** у NVIDIA заметно улучшилась поддержка Wayland в новых драйверах

-----

### Установка драйвера NVIDIA

#### Метод 1: RPM Fusion — строго рекомендуется

RPM Fusion остаётся самым надёжным способом установки NVIDIA-драйверов на Fedora. Он нормально интегрируется с Wayland и системными обновлениями.

```bash
# Включить RPM Fusion, если ещё не включён
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Обновить пакеты
sudo dnf update

# Установить NVIDIA-драйверы с поддержкой Wayland
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# Установить 32-bit compatibility libraries, нужны для Steam/Wine/gaming
sudo dnf install xorg-x11-drv-nvidia-libs.i686

# Установить NVIDIA Settings
sudo dnf install nvidia-settings
```

#### Проверка после установки

Эти команды помогают понять, нормально ли система настроена для максимальной производительности.

```bash
# Проверить установку драйвера и версию
nvidia-smi
# Должны отображаться GPU, версия драйвера 595+ и текущая загрузка

# Проверить поддержку CUDA
nvidia-smi -q | grep "CUDA Version"
# Важно для AI-нагрузок и некоторых игр с GPU compute

# Проверить, что Wayland-сессия используется с NVIDIA GPU
echo $XDG_SESSION_TYPE
# Должно вывести "wayland"

# Проверить, что GBM backend работает
nvidia-smi --query-gpu=name,driver_version --format=csv
# Подтверждает, что драйвер загрузился нормально
```

#### Включение Wayland для NVIDIA — важный шаг

Wayland требует правильной настройки NVIDIA-модулей. Это нужно, чтобы desktop environment нормально использовал hardware acceleration.

```bash
# Включить DRM kernel mode setting, нужно для Wayland
echo 'options nvidia_drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-drm-modeset.conf

# Включить раннюю загрузку NVIDIA-модулей
echo -e 'nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm' | sudo tee /etc/modules-load.d/nvidia.conf

# Пересобрать initramfs
sudo dracut --force

# Перезагрузиться для применения изменений
sudo reboot
```

> **Пользователям Fedora + RPM Fusion:** этот шаг, скорее всего, **не нужен** — `akmod-nvidia` обычно включает modeset автоматически. Делайте вручную только если Wayland не стартует после установки драйвера или если вы ставили драйвер через официальный `.run`, а не RPM Fusion.

-----

### Оптимизация производительности NVIDIA Wayland

#### 1. Переменные окружения для Wayland

Эти переменные настраивают поведение NVIDIA GPU именно под Wayland compositors. В отличие от X11, Wayland многое делает сам, поэтому глобально добавлять всё подряд не стоит.

> **Важное предупреждение про NVIDIA Wayland**
>
> Не добавляйте все NVIDIA-переменные вслепую в `/etc/environment`.
> На современных Fedora + RPM Fusion NVIDIA-сетапах большая часть Wayland-настроек уже обрабатывается автоматически.
>
> Глобальные NVIDIA-переменные могут сломать вход в Wayland, вызвать чёрный экран, статтеры или input lag — зависит от поколения GPU, версии драйвера и DE.
>
> Сначала тестируйте их per-game, например в Steam launch options.

Добавить в `/etc/environment`:

```bash
# Основные NVIDIA Wayland optimizations
#
# Критическое предупреждение для современных NVIDIA GPU, RTX 20 series и новее
#
# По отзывам и тестам, две переменные ниже (`GBM_BACKEND` и `__GLX_VENDOR_LIBRARY_NAME`) могут вызывать сильный системный input lag, статтеры и зависания приложений на NVIDIA RTX 20/30/40/50 series
#
# ! Рекомендация для RTX 20-series и новее: НЕ используйте эти переменные. Современные драйверы NVIDIA и Wayland compositors справляются сами.
# ! Рекомендация для старых GPU, GTX 10-Series и старше: переменные всё ещё могут помочь с Wayland-совместимостью.
#
# Только для старых карт, GTX 10-Series и ниже:
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia

# Включить threaded optimizations, улучшает параллелизм CPU-GPU
__GL_THREADED_OPTIMIZATIONS=1
# Предупреждение: на некоторых RTX-картах может давать чёрный экран. Лучше ставить per-game, см. troubleshooting.
# Если получили чёрный экран после глобального включения:
#    - нажмите Ctrl+Alt+F3, войдите в TTY
#    - sudo nano /etc/environment, удалите строку, сохраните Ctrl+O, Enter, Ctrl+X
#    - sudo reboot
# Спасибо @lemonadeforlife за проблему и решение

# Кэш компиляции шейдеров, снижает статтеры в играх
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_PATH=/tmp/nvidia-shader-cache
__GL_SHADER_DISK_CACHE_SIZE=1073741824

# Отключить VSync для игр, снижает input lag
__GL_SYNC_TO_VBLANK=0

# Включить NVAPI для DLSS и похожих функций в Proton
PROTON_ENABLE_NVAPI=1

# Принудительно использовать native Wayland path в Proton
PROTON_ENABLE_WAYLAND=1

# Фикс статтеров Steam через 25–30+ минут игры
LD_PRELOAD=""

# Не чистить shader cache при запуске, чтобы избежать stutter spikes
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1

# Следующие твики не рекомендуется ставить глобально.
# Только для wl-roots compositors: Sway, Hyprland. На GNOME/KDE не нужно.
# С driver 595+ и Sway 1.11+ / Hyprland с explicit sync
# WLR_NO_HARDWARE_CURSORS может быть уже не нужен. Сначала тестируйте без него.
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1
```

#### 2. Параметры модулей ядра — если понимаете, зачем

Современные NVIDIA-драйверы могут выиграть от некоторых параметров ядра. Тестируйте каждый параметр на своём железе.

Создайте `/etc/modprobe.d/nvidia-power-management.conf`:

```bash
# Включить современные power management features
options nvidia NVreg_DynamicPowerManagement=0x02

# Включить Page Attribute Table, улучшает memory performance
options nvidia NVreg_UsePageAttributeTable=1

# Включить ResizableBAR, RTX 30/40/50 series
options nvidia NVreg_EnableResizableBar=1

# Сохранять видеопамять при suspend
options nvidia NVreg_PreserveVideoMemoryAllocations=1

# Включить stream memory operations, нужно для некоторых workloads
options nvidia NVreg_EnableStreamMemOPs=1
```

#### 3. Настройки GNOME Wayland

GNOME на Wayland требует аккуратной настройки для нормальной NVIDIA-производительности.

**Включить Variable Refresh Rate (VRR):**

```bash
# Fedora 44 с GNOME 50: VRR теперь стабилен — включайте напрямую в Settings → Displays
# Или через команду:
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"
```

> **Примечание для Fedora 44:** в GNOME 50 VRR больше не экспериментальный. Его можно включить напрямую в **Settings → Displays → Variable Refresh Rate** без `gsettings`. Команда выше всё ещё работает, но уже не обязательна.

```bash
# Настроить GNOME под игровую производительность
gsettings set org.gnome.shell.extensions.dash-to-dock click-action 'cycle-windows'
gsettings set org.gnome.desktop.interface enable-animations false

# Масштаб для high-DPI дисплеев, подберите под себя
gsettings set org.gnome.desktop.interface scaling-factor 1
```

#### 4. Настройка KDE Plasma Wayland

KDE Plasma хорошо работает с NVIDIA на Wayland, если нормально настроить compositor.

```bash
# Включить VRR, лучше через настройки KDE/GNOME, но можно и так
kwriteconfig6 --file kwinrc --group Compositing --key VariableRefreshRate true

# Оптимизировать compositor для игр
kwriteconfig6 --file kwinrc --group Compositing --key LatencyPolicy Low
kwriteconfig6 --file kwinrc --group Compositing --key RenderTimeEstimator 1

# Перезапустить KWin для применения
qdbus org.kde.KWin /KWin reconfigure
```

-----

### Игровые оптимизации NVIDIA

#### 1. Steam launch options для Wayland

Для Steam на Wayland нужны правильные launch options, чтобы игры нормально использовали NVIDIA GPU и не теряли производительность.

**Для нативных Linux-игр:**

```bash
# Базовая оптимизация с GameMode
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 %command%

# Улучшенная производительность для competitive games
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 %command%
```

**Для Proton/Wine-игр:**

```bash
# Стандартная Proton-оптимизация
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Продвинутая оптимизация с GPL pipeline compilation, замена старому dxvk-async
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Для игр, где нужен максимум производительности
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%
```

#### 2. Оптимизация Lutris

Lutris хорошо интегрируется с NVIDIA-драйверами на Wayland. Настройте переменные для лучшей производительности.

```bash
# Установить Lutris с NVIDIA support
sudo dnf install lutris wine

# Переменные окружения в настройках Lutris
# Добавить в System Options → Environment variables:
__GL_THREADED_OPTIMIZATIONS=1
__GL_SHADER_DISK_CACHE=1
PROTON_ENABLE_NVAPI=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
```

#### 3. Интеграция GameMode

GameMode автоматически применяет оптимизации во время игры. Можно немного подкрутить его поведение.

```bash
# Установить GameMode
sudo dnf install gamemode

# Настроить GameMode под NVIDIA
sudo nano /etc/gamemode.ini
[general]
renice=10
ioprio=0

[gpu]
# Предупреждение: пытается разгонять GPU через nvidia-settings. Тестируйте осторожно.
apply_gpu_optimisations=accept-responsibility
gpu_device=0
```

-----

### Продвинутые твики NVIDIA Wayland

#### 1. Variable Refresh Rate (VRR)

Современные NVIDIA-драйверы поддерживают VRR на Wayland, что даёт более плавный гейминг на совместимых мониторах.

```bash
# Включить VRR в GNOME
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"
# Или через Display settings, если доступно

# Проверить VRR
sudo dnf install drm_info
drm_info | grep -i vrr
```

#### 2. HDR support — экспериментально

HDR на Wayland с NVIDIA постепенно улучшается. Эти настройки включают экспериментальную HDR-функциональность.

```bash
# Непроверенный параметр — тестируйте перед применением, его нет в официальных NVIDIA docs
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf
```

#### 3. Мониторинг и тюнинг производительности

Мониторинг помогает найти bottlenecks и понять, работают ли оптимизации.

```bash
# Установить инструменты мониторинга
sudo dnf install nvtop mangohud goverlay

# Создать скрипт мониторинга для игровых сессий
sudo nano /usr/local/bin/nvidia-monitor.sh
#!/bin/bash
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
nvidia-smi dmon -s pucvmet

sudo chmod +x /usr/local/bin/nvidia-monitor.sh
```

-----

### Управление питанием и температурами

#### Продвинутое управление питанием — только если понимаете, что делаете

Правильное power management даёт стабильную производительность и не держит лишнее потребление в простое.

```bash
# Настроить advanced power management
echo 'options nvidia NVreg_DynamicPowerManagement=0x02' | sudo tee -a /etc/modprobe.d/nvidia-power.conf

# Включить runtime power management для ноутбуков
sudo nano /etc/udev/rules.d/80-nvidia-pm.rules
# Runtime PM для NVIDIA VGA/3D controller devices
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
```

-----

### Решение проблем NVIDIA Wayland

#### Частые проблемы и современные решения

Диагностика важна: она помогает не гадать и быстро понять, что именно ломает производительность или запуск.

**1. Wayland-сессия не стартует с NVIDIA:**

Это самая частая проблема при переходе с X11 на Wayland с NVIDIA-драйверами.

### Чёрный экран NVIDIA при загрузке

**Проблема:** после загрузки с NVIDIA-драйвером виден чёрный экран.

**Причина:** переменная `__GL_THREADED_OPTIMIZATIONS=1` в `/etc/environment` может ломать инициализацию дисплея на некоторых RTX GPU.

**Решение:**

1. Загрузитесь в recovery mode или переключитесь в TTY через Ctrl+Alt+F3.
2. Откройте `/etc/environment` и удалите или закомментируйте строку:

```bash
# __GL_THREADED_OPTIMIZATIONS=1
```

```bash
# Проверить параметры модуля ядра
cat /etc/modprobe.d/nvidia-drm-modeset.conf
# Должно быть: options nvidia_drm modeset=1 fbdev=1

# Проверить, что DRM modeset включён
cat /sys/module/nvidia_drm/parameters/modeset
# Должно вывести: Y

# Пересобрать initramfs и перезагрузиться, если нужно
sudo dracut --force
sudo reboot

# Проверить Wayland-сессию после перезагрузки
echo $XDG_SESSION_TYPE
# Должно вывести: wayland
```

**2. Плохая игровая производительность при нормальном железе:**

Часто проблема в неправильном использовании GPU или power management.

```bash
# Проверить, что GPU реально используется
nvidia-smi dmon -s pucvmet -c 10

# Проверить power limit
nvidia-smi --query-gpu=power.limit,power.draw --format=csv
# В игре power draw должен приближаться к power limit

# Смотреть частоты GPU во время игры
watch -n 1 'nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv'
# В игре частоты должны доходить до максимальных значений
```

**3. Screen tearing или статтеры:**

Современные Wayland compositors лучше работают с tearing, чем X11, но иногда нужны настройки.

```bash
# Для GNOME убедитесь, что VRR включён
gsettings get org.gnome.mutter experimental-features
# Должно содержать 'variable-refresh-rate'

# В играх отключите VSync внутри игры и используйте compositor VSync
# Добавьте в Steam launch options:
__GL_SYNC_TO_VBLANK=0 %command%
```

**4. Высокое потребление в простое:**

Нужно не держать лишнее питание на GPU, когда он не используется. Это снижает нагрев и помогает батарее на ноутбуках.

```bash
# Включить runtime power management
echo 'auto' | sudo tee /sys/bus/pci/devices/0000:*/power/control

# Проверить, работает ли power management
cat /sys/bus/pci/devices/0000:*/power/runtime_status
# Для простаивающего GPU должно быть 'suspended'

# Смотреть idle power draw
nvidia-smi --query-gpu=power.draw --format=csv --loop=1
```

-----

### Мониторинг производительности и бенчмаркинг

#### Комплексный мониторинг

Мониторинг помогает оптимизировать систему и находить проблемы до того, как они испортят игру или работу.

```bash
# Установить набор инструментов мониторинга
sudo dnf install nvtop btop mangohud goverlay
```

#### Игровой performance overlay

MangoHud показывает метрики производительности прямо в игре.

```bash
# Настроить MangoHud
mkdir -p ~/.config/MangoHud

nano ~/.config/MangoHud/MangoHud.conf
# Информация GPU и CPU
gpu_stats
cpu_stats
gpu_temp
cpu_temp

# FPS и frametime
fps
frametime
frame_timing

# Память
vram
ram

# Позиция и внешний вид
position=top-left
font_size=22
alpha=0.8

# Ограничить логирование, чтобы не бить по производительности
log_duration=60
```

-----

### Дополнительные ресурсы

**Официальная документация NVIDIA:**
- [NVIDIA Linux Driver Installation Guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
- [CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

**Ресурсы по Fedora:**
- [RPM Fusion NVIDIA Guide](https://rpmfusion.org/Howto/NVIDIA)

**Wayland и gaming:**
- [Gaming on Linux with NVIDIA](https://www.gamingonlinux.com/)
- [MangoHud Documentation](https://github.com/flightlessmango/MangoHud)

-----

</details>

-----

## Мониторинг и проверка

### Инструменты мониторинга производительности

```bash
# Установить мониторинг
sudo dnf install btop

# Детальная информация о системе
sudo dnf install hardinfo2
```

### Инструменты бенчмаркинга

```bash
# Игровые бенчмарки
sudo dnf install glmark2 unigine-superposition

# Системные бенчмарки
sudo dnf install sysbench stress-ng
```

-----

## Решение проблем

### Частые проблемы

**1. Проблемы с загрузкой после параметров ядра:**

- Загрузитесь с прошлого ядра через GRUB menu
- Уберите проблемные параметры из `/etc/default/grub`
- Пересоберите GRUB config

**2. Проблемы с графикой:**

- Проверить установку драйвера: `lspci -k | grep -A 2 -E "(VGA|3D)"`
- Проверить, что нужный драйвер загружен: `lsmod | grep -E "(amdgpu|nvidia|i915)"`

**3. Регресс производительности:**

- Смотреть ресурсы системы: `htop`, `iotop`
- Проверить thermal throttling: `watch sensors`
- Проверить сломанные сервисы: `systemctl list-units --failed`

### Команды восстановления

```bash
# Сбросить GRUB к дефолту
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Сбросить SELinux context, если включаете SELinux обратно
sudo restorecon -R /

# Проверить целостность системы
sudo dnf check
sudo rpm -Va
```

-----

## Ожидаемый прирост производительности

По тестам можно ожидать:

- **Время загрузки:** улучшение на 10–20%
- **Игровая производительность:** +5–15% FPS
- **Отзывчивость системы:** заметно ниже input lag
- **Использование памяти:** на 5–15% меньше RAM в простое
- **Производительность накопителя:** лучше работа SSD после TRIM

-----

## Участие в разработке

Нашли улучшения или есть предложения:

- Откройте issue на GitHub
- Отправьте pull request
- Поделитесь результатами оптимизации

-----

## Дополнительные ресурсы

- [Fedora Documentation](https://docs.fedoraproject.org/)
- [RPM Fusion](https://rpmfusion.org/)
- [CachyOS Kernel](https://github.com/CachyOS/linux-cachyos)
- [Ananicy-cpp](https://gitlab.com/ananicy-cpp/ananicy-cpp)
- [Gaming on Linux](https://www.gamingonlinux.com/)

-----

## Дисклеймер

Этот гайд меняет системные настройки, которые могут повлиять на стабильность и безопасность. Перед применением:

- Сделайте резервную копию системы
- Понимайте последствия каждого изменения
- Держите под рукой recovery media

**Результаты по производительности могут отличаться** в зависимости от железа и сценария использования.

-----

<details>
<summary>Список изменений</summary>

- **v1.0** — первый релиз гайда
- **v1.1** — добавлен раздел troubleshooting и русский перевод
- **v1.2** — добавлены инструменты мониторинга и раздел обслуживания
- **v1.3** — добавлен NVIDIA-драйвер, performance guide и другие разделы
- **v1.4** — добавлен раздел AMD GPU tweaks
- **v1.5** — добавлена быстрая навигация
- **v1.6** — переход с UKSMD на KSMD, глубже расписана настройка ядра CachyOS
- **v1.7** — больше NVIDIA-флагов, исправления, подготовка к Fedora 43
- **v1.8** — обновлена почти вся структура гайда, особенно AMD-секция
- **v1.9** — больше деталей, обновление под Fedora 44

</details>

-----

*Последнее обновление: июль 2026*
</details>

-----

## Contributing

Found improvements or have suggestions? Feel free to:

- Open an issue on GitHub
- Submit a pull request
- Share your optimization results

-----

## Additional Resources

- [Fedora Documentation](https://docs.fedoraproject.org/)
- [RPM Fusion](https://rpmfusion.org/)
- [CachyOS Kernel](https://github.com/CachyOS/linux-cachyos)
- [Ananicy-cpp](https://gitlab.com/ananicy-cpp/ananicy-cpp)
- [Gaming on Linux](https://www.gamingonlinux.com/)

-----

## Disclaimer

This guide modifies system settings that may affect stability and security. Always:

- Create system backups before applying changes
- Understand the implications of each modification
- Keep recovery media accessible

**Performance results may vary** based on hardware configuration and specific use cases.

-----

<details>
<summary>Changelog</summary>

- **v1.0** - Initial guide creation
- **v1.1** - Added troubleshooting section and Russian translation
- **v1.2** - Enhanced with monitoring tools and maintenance section
- **v1.3** - Added NVIDIA drivers, performance guide and more
- **v1.4** - Added AMD gpu tweaks section
- **v1.5** - Added a Quick Navigation section for better usability
- **v1.6** - Changed from UKSMD to KSMD, more deeper cachyos kernel setup description
- **v1.7** - Updated with more nvidia flags, corrections, preparing for Fedora 43 release
- **v1.8** - Updated almost every guide section for even more performance gain, especially AMD based
- **v1.9** - More detailed info, also updated for Fedora 44

</details>

-----

*Last updated: July 2026*
