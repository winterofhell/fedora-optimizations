# Fedora Linux 43 Performance & Gaming Optimization Guide

> **Complete guide for optimizing Fedora 43 for maximum performance, gaming, general use, etc | by winterofhell**

## Quick Navigation

| Setup & Kernel                                                              | System & Gaming                                                            | Resources                                                                 |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [**System Information**](#system-information)                              | [**Advanced System Tweaks**](#advanced-system-tweaks)                     | [**Monitoring & Verification**](#monitoring--verification)               |
| [**Initial Setup & Preparation**](#initial-setup--preparation)             | [**Gaming Optimizations**](#gaming-optimizations)                         | [**Troubleshooting**](#troubleshooting)                                  |
| [**Kernel Optimization**](#kernel-optimization)                            | [**Maintenance & Cleanup**](#maintenance--cleanup)                        | [**Russian Translation**](#русская-версия--russian-translation)     |
| [**GRUB Kernel Parameters**](#grub-kernel-parameters)                       | [**Graphics Driver Optimization**](#graphics-driver-optimization)          |                                                                           |

## System Information

**Testing Environment:**

- **Period:** October 14, 2024 - December 31, 2025
- **Distribution:** Fedora 43
- **Additional Testing:** NVIDIA and AMD gpu systems
- **These optimizations may also work on any other distro, but i cannot guarantee it. It is always necessary to test everything. About 90% of these tweaks work on Arch and NixOS :)**

**Hardware Configurations (tested on):**

- **First:** Ryzen 5 5500U, 20GB DDR4, RX550X/RX Vega 7, NVMe
- **Second:** Ryzen 5 5600, 16GB DDR4, GTX 1060, SATA SSD
- **Third:** Ryzen 5 7500f, 32Gb DDR5, RX 9070 XT, Nvme

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

**More Info:** [CachyOS Kernel Installation](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/)

-----

## System Services Optimization

### Ananicy-cpp Installation

Ananicy-cpp automatically manages process priorities and reduces system latency:

```bash
# Install build dependencies
sudo dnf group install "Development Tools"
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel make automake gcc gcc-c++

# Clone and build
git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install

# Enable service
sudo systemctl enable --now ananicy-cpp

### Leveraging Automated CachyOS Tweaks

# The CachyOS team provides powerful packages that can automate many of the advanced tweaks. This is a simpler and safer approach than manually setting dozens of system variables :)

# 1. Install CachyOS Optimization Packages
sudo dnf install cachyos-settings cachyos-ksm-settings scx-scheds

# 2. Advanced CPU Scheduler Optimization (SCX)
# This is one of the most impactful tweaks for system responsiveness and gaming performance. We will replace the default Linux CPU scheduler with a specialized one from the `scx-scheds` package we installed earlier.
# Note: This is an advanced tweak. While it provides significant gains, it changes a core component of the system !!

# Step 1: Configure the Default Scheduler
# We will set `scx_lavd` as our default scheduler. It is currently the best scheduler for gaming latency
# on modern cpus, as it prioritizes critical game threads over background noise.

sudo nano /etc/scx_loader/config.toml

# Set the lavd scheduler as default and configure it for gaming mode.
default_sched = "scx_lavd"
default_mode = "Gaming"
 
[scheds.scx_lavd]
auto_mode = ["--performance"]
gaming_mode = ["-m", "performance"]

### Important: Disable IRQBalance
# If you are using `scx_scheds`, you MUST disable `irqbalance`. It fights with the scheduler causing micro-stutters.

sudo systemctl disable --now irqbalance

# Step 2: Enable and Start the Scheduler Service
sudo systemctl enable --now scx_loader

# Step 3: Verify the Change
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# the output should show string "scx_lavd".

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
### Parameter Breakdown:

amd_pstate=active: **CRITICAL for Zen 3/4/5. Enables the EPP driver for millisecond-speed clock boosting. (Intel users: remove this).**

amdgpu.ppfeaturemask=0xffffffff:  **CRITICAL for AMD GPU. Unlocks voltage control and overclocking limits (essential for LACT/CoreCtrl).**

split_lock_detect=off:  **Prevents SIGBUS crashes and stutters in games that use "split locks" (anti-cheat often does this).**

nowatchdog & nmi_watchdog=0:  **Disables CPU cycle-wasting watchdog timers. Reduces micro-stutter.**

zswap.enabled=0:  **Disables ZSWAP to lower latency. (Only do this if you have 32GB+ RAM. If you have 4-16GB, remove this).**

pci=pcie_bus_perf:  **Forces PCIe bus to max payload size (Maximum GPU/NVMe bandwidth).**

pcie_aspm.policy=performance:  **Disables PCIe power saving states. Fixes idle latency spikes. (its basically better than pcie_aspm=off)!**

usbcore.autosuspend=-1:  **Prevents USB devices from sleeping. Fixes "wake up" lag.**

## Experimental (Use at your own risk):
**mitigations=off: Disables CPU security patches.**

Good for: **Older CPUs (Zen 1/2/3, Intel 9th gen and older).** ~3% fps gain.

Bad for: **Zen 4 / Zen 5 (Ryzen 7000/9000).** Can actually lower 1% low FPS due to branch prediction logic. **Zen 4 users should NOT use this!**

### Update GRUB Configuration

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

-----

## Advanced System Tweaks

### Memory Management

**Enable systemd-oomd (Out-of-Memory Daemon):**

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

Do NOT enable this if you already enabled scx_lavd!
If experiencing performance issues with Intel integrated graphics:

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

**For Native HDR Games (Cyberpunk, Elden Ring, etc):**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --force-grab-cursor --adaptive-sync --sharpness 2 -f -- %command%
```
## Environment Variables & Wrappers

MANGOHUD_CONFIG="fps_limit=277,no_display": **This configures the MangoHud layer silently.**

fps_limit=277: **This is your manual "low latency" mode(like amd antilag). Take your monitor's max Refresh Rate and subtract 3 (e.g., 280Hz - 3 = 277 FPS). This keeps the framerate strictly inside your FreeSync range, preventing V-Sync backpressure and minimizing input lag.**

no_display: **Hides the overlay so you don't see the stats, but the limiter still works in the background.**

mangohud: **Actually injects the HUD layer. Without this, the config above does nothing.**

LD_PRELOAD="": **this fixes Steam stuttering issues after playing for like 25-30+ minutes**

AMD_USERQ=1: **(RDNA 3/4 Only) Enables User Queues. Lets the game talk directly to the GPU hardware queues, bypassing some kernel driver overhead. Basically free CPU performance.**

PROTON_NTSYNC=1: **(Requires CachyOS Kernel) Replaces the old fsync/esync emulation with a proper kernel-level driver for Windows threading. Massively reduces CPU overhead in complex games.**

gamemoderun: **Just a feral gamemode option**

## Gamescope Arguments (The Container)

gamescope: **Valve's micro-compositor. It isolates the game window from your desktop, fixing Alt-Tab crashes and handling resolution/HDR perfectly.**

-W 2560 -H 1440: **Sets the internal resolution the game thinks it is running at. Set this to your monitor's native resolution (like 1920x1080 etc)**

-r 280: **Sets the refresh rate cap for the Gamescope container. Match this exactly to your monitor's max Hz (144hz, 180, 60, 100 etc).**

--hdr-enabled: **Unlocks HDR output. Essential for oled/ips to actually trigger HDR mode.**

--force-grab-cursor: **Prevents your mouse from accidentally clicking outside the game window onto a second monitor.**

--adaptive-sync: **Explicitly enables VRR (FreeSync/G-Sync) inside the container so you don't get screen tearing**

--sharpness 2: **Applies a high-quality CAS sharpening filter.**
How it works: **The scale is 0 to 20, where 0 is Maximum Sharpening and 20 is Least Sharpening. A value of 2 provides a very crisp, detailed image.**

-f: **Forces the container to display in Fullscreen mode.**

--: **The separator. It tells Gamescope "My settings stop here, the game command starts next."**

%command%: **Steam automatically replaces this with the actual game executable.**


**For games without HDR:**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display” mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --force-grab-cursor --adaptive-sync —sharpness 2 -f -- %command%
```


**For games without hdr, BUT if you want auto SDR to HDR (like Auto HDR on windows)**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_NTSYNC=1 gamemoderun gamescope -W 2560 -H 1440 -r 280 --hdr-enabled --hdr-itm-enabled --hdr-itm-target-nits 1000 --hdr-sdr-content-nits 203 --force-grab-cursor --adaptive-sync --sharpness 2 -f -- %command%
```

## Auto sdr to hdr convertation
--hdr-enabled: **Initializes the HDR pipeline. Without this, the other flags do nothing.**

--hdr-itm-enabled: **Inverse Tone Mapping. This is the "Auto HDR" switch. It expands the color range of SDR games to use your OLED/ips full capabilities.**

--hdr-itm-target-nits 1000: **Peak Brightness. Tells the algorithm your monitor hits 1000 nits. This ensures bright effects (sun, fire, magic) actually hit max brightness without clipping details.**

--hdr-sdr-content-nits 203: **Base Brightness (paper white). Sets the brightness for standard textures and UI. 203 is the industry standard. If you set this too high, menus will blind you; too low, and the game looks dim...**

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

## As for rx9070xt users, i recommend to start with "Performance Level: Manual", -70mv voltage offset, lock your memory clock on 2714Mhz, set Power Limit to 355W. For undervolt, set PL to 260-270W.
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

- **Best:** NVIDIA 590+ drivers for optimal Wayland support and performance
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
# This should show your GPU, driver version (570+), and current utilization

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

-----

### NVIDIA Wayland Performance Optimizations

#### 1. Environment Variables for Wayland

These environment variables optimize NVIDIA GPU behavior specifically for Wayland compositors. Unlike X11, Wayland handles many optimizations automatically, but these variables fine-tune performance.

Add to `/etc/environment`:

```bash
# Core NVIDIA Wayland optimizations
#
# Critical Warning for Modern NVIDIA GPUs (RTX 20 series and newer)
#
# Based on user feedback and testing, the following two env variables (`GBM_BACKEND` and `__GLX_VENDOR_LIBRARY_NAME`) can cause severe system-wide input lag, stuttering, and application unresponsiveness on NVIDIA RTX 20, 30, 40, and 50 series gpus
#
# ! Recommendation for RTX 20-series and newer: DO NOT use these variables. Modern nvidia drivers and wayland compositors generally handle this configuration automatically. Enabling them manually can create conflicts
# ! Recommendation for older gpus (GTX 10-Series and older): These variables can still be beneficial for ensuring wayland compatibility on older hardware. you can try them. any issue report with a specific gpu problems will be very valuable! :)
#
#
# For older cards ONLY (GTX 10-Series and below), you might still need:
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia
#
# Enable threaded optimizations (improves CPU-GPU parallelism)
__GL_THREADED_OPTIMIZATIONS=1
# Warning: __GL_THREADED_OPTIMIZATIONS option can cause black screens on some RTX cards (See NVIDIA Wayland Troubleshooting)
# Set this per-game instead (see troubleshooting section)
# Or test and set it for environment, if you encounter a black screen - log in through tty, remove __GL_THREADED_OPTIMIZATION=1 from /etc/environment, save and reboot.
# short tty guide
#    - press Ctrl+Alt+F3 (or F2–F6) to switch to a TTY login screen.
#    - log in with your username and password.
#
# 2. edit /etc/environment and remove the problematic line:
# sudo nano /etc/environment
#    - look for the line:
#        __GL_THREADED_OPTIMIZATIONS=1
#    - delete it, then save (Ctrl+O, Enter) and exit (Ctrl+X).
#
# 3. reboot your system:
# sudo reboot
# Btw it's still recommended to set this option only per game
# Thanks to @lemonadeforlife for pointing out this problem and solution

# Shader compilation caching (reduces stutter in games)
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_PATH=/tmp/nvidia-shader-cache
__GL_SHADER_DISK_CACHE_SIZE=1073741824

# Disable VSync for gaming (reduces input lag)
__GL_SYNC_TO_VBLANK=0

# Enable unofficial protocol extensions for Wayland compatibility
__GL_ALLOW_UNOFFICIAL_PROTOCOL=1

# Gaming-specific optimizations
# Enables NVAPI for features like DLSS in Proton, rtx gpu users test this please
PROTON_ENABLE_NVAPI=1
NVIDIA_DRIVER_CAPABILITIES=all

# Good flags for every game on Steam/PortProton/Lutris etc found on reddit and used by community
PROTON_ENABLE_WAYLAND=1
LD_PRELOAD=""
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
%command%
-fullscreen;

# The following tweaks are not recommended globally. Use them only if you know you need them.

# For wl-roots compositors ONLY (Sway, Hyprland)
# These may help with graphical glitches but are not needed and recommended on GNOME/KDE.
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
# Enable NVIDIA acceleration for GNOME Wayland session
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"

# Configure GNOME for gaming performance
gsettings set org.gnome.shell.extensions.dash-to-dock click-action 'cycle-windows'
gsettings set org.gnome.desktop.interface enable-animations false

# Set scaling factor for high-DPI displays (adjust as needed)
gsettings set org.gnome.desktop.interface scaling-factor 1
```

#### 4. KDE Plasma Wayland Configuration

KDE Plasma has excellent Wayland support and works particularly well with NVIDIA drivers when properly configured.

```bash
# Enable variable refresh rate support (or using kde settings which is better)
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

# Advanced optimization with dxvk async/gpl compilation
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 DXVK_ASYNC=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# For games requiring maximum performance
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 DXVK_ASYNC=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%
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
ioprio=1

[gpu]
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
# Enable HDR support (requires compatible display and recent drivers)
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf

# GNOME HDR support (experimental!)
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate','hdr']"
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
    
# Гайд по оптимизации производительности и гейминга Fedora Linux 43

> **Полное руководство по оптимизации Fedora 43 для максимальной производительности, гейминга, повседневного использования и т.д. | от winterofhell**

## Быстрая навигация

| Настройка и ядро                                                              | Система и гейминг                                                            | Ресурсы                                                                 |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [**Информация о системе**](#информация-о-системе)                              | [**Продвинутые системные твики**](#продвинутые-системные-твики)                     | [**Мониторинг и проверка**](#мониторинг-и-проверка)               |
| [**Начальная настройка**](#начальная-настройка-и-подготовка)             | [**Оптимизация для гейминга**](#оптимизация-для-гейминга)                         | [**Решение проблем**](#решение-проблем)                                  |
| [**Оптимизация ядра**](#оптимизация-ядра)                            | [**Обслуживание и очистка**](#обслуживание-и-очистка)                        | [**Английская версия**](#english-version)     |
| [**Параметры ядра GRUB**](#параметры-ядра-grub)                       | [**Оптимизация графических драйверов**](#оптимизация-графических-драйверов)          |                                                                           |

## Информация о системе

**Тестовое окружение:**

- **Период:** 14 октября 2024 - 31 декабря 2025
- **Дистрибутив:** Fedora 43
- **Дополнительное тестирование:** Системы с NVIDIA и AMD видеокартами
- **Эти оптимизации могут работать и на других дистрибутивах, но я не могу этого гарантировать. Всегда необходимо всё тестировать. Около 90% этих твиков работают на Arch и NixOS :)**

**Конфигурации железа (протестировано на):**

- **Первая:** Ryzen 5 5500U, 20GB DDR4, RX550X/RX Vega 7, NVMe
- **Вторая:** Ryzen 5 5600, 16GB DDR4, GTX 1060, SATA SSD
- **Третья:** Ryzen 5 7500f, 32Gb DDR5, RX 9070 XT, Nvme

-----

## Начальная настройка и подготовка

### 1. Минимальная установка

Для оптимальной производительности всегда начинайте с **Fedora Minimal ISO**. Этот подход исключает ненужные пакеты и сервисы, которые могут влиять на ресурсы системы. Но это необязательно.

### 2. Включение репозиториев RPM Fusion

RPM Fusion предоставляет необходимые мультимедийные кодеки и проприетарные драйверы:

```bash
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

**Официальный гайд:** [Настройка RPM Fusion](https://rpmfusion.org/Configuration)

### 3. Настройка SELinux (опционально)

**Предупреждение о безопасности:** Отключение SELinux снижает безопасность системы, но также делает вашу систему немного быстрее. Продолжайте только если понимаете последствия. (Лично мне плевать на SELinux и я всегда его отключаю)

**Временное отключение (до перезагрузки):**

```bash
sudo setenforce 0
```

**Постоянное отключение (требуется перезагрузка):**

```bash
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# Требуется перезагрузка для применения изменений
```

### 4. Обновление системы

Всегда начинайте с полностью обновлённой системы:

```bash
sudo dnf upgrade --refresh

# Установка обновлений прошивки (включая микрокод CPU)
sudo dnf install linux-firmware intel-ucode amd-ucode
```

-----

## Оптимизация ядра

### Установка ядра CachyOS

Ядро CachyOS обеспечивает значительное улучшение производительности для гейминга и общей отзывчивости системы.

**Требования:** Процессор должен поддерживать набор инструкций x86_64_v3 !!

```bash
# Проверка поддержки CPU
# Проверьте поддержку следующей командой

/lib64/ld-linux-x86-64.so.2 --help | grep "(supported, searched)"

# Если не определяется поддержка x86_64_v3, НЕ устанавливайте это ядро. Если определяется только x86_64_v2, можете использовать LTS-ядро.
```

```bash
# Добавление COPR-репозитория CachyOS
sudo dnf copr enable bieszczaders/kernel-cachyos
sudo dnf copr enable bieszczaders/kernel-cachyos-addons

# Установка ядра CachyOS
sudo dnf install kernel-cachyos kernel-cachyos-devel

# Только для x86_64_v2 (старые процессоры):
sudo dnf install kernel-cachyos-lts kernel-cachyos-lts-devel-matched
```

**Подробнее:** [Установка ядра CachyOS](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/)

-----

## Оптимизация системных сервисов

### Установка Ananicy-cpp

Ananicy-cpp автоматически управляет приоритетами процессов и снижает системную задержку:

```bash
# Установка зависимостей для сборки
sudo dnf group install "Development Tools"
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel make automake gcc gcc-c++

# Клонирование и сборка
git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install

# Включение сервиса
sudo systemctl enable --now ananicy-cpp

### Использование автоматизированных твиков CachyOS

# Команда CachyOS предоставляет мощные пакеты, которые могут автоматизировать многие продвинутые твики. Это более простой и безопасный подход, чем ручная настройка десятков системных параметров :)

# 1. Установка пакетов оптимизации CachyOS
sudo dnf install cachyos-settings cachyos-ksm-settings scx-scheds

# 2. Продвинутая оптимизация планировщика CPU (SCX)
# Это один из самых значимых твиков для отзывчивости системы и производительности в играх. Мы заменим стандартный планировщик Linux на специализированный из пакета `scx-scheds`, который мы установили ранее.
# Примечание: Это продвинутый твик. Хотя он даёт значительный прирост, он меняет ключевой компонент системы !!

# Шаг 1: Настройка планировщика по умолчанию
# Мы установим `scx_lavd` в качестве планировщика по умолчанию. Это на данный момент лучший планировщик для игровых задержек
# на современных процессорах, так как он приоритизирует критичные игровые потоки над фоновым шумом.

sudo nano /etc/scx_loader/config.toml

# Установите планировщик lavd по умолчанию и настройте его для игрового режима.
default_sched = "scx_bpfland"
default_mode = "Gaming"
 
[scheds.scx_lavd]
auto_mode = ["--performance"]
gaming_mode = ["-m", "performance"]

### Важно: Отключите IRQBalance
# Если вы используете `scx_scheds`, вы ДОЛЖНЫ отключить `irqbalance`. Он конфликтует с планировщиком, вызывая микрофризы.

sudo systemctl disable --now irqbalance

# Шаг 2: Включите и запустите сервис планировщика
sudo systemctl enable --now scx_loader

# Шаг 3: Проверьте изменение
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# вывод должен показать string "scx_bpfland".

```

### Управление сервисами

Отключите ненужные сервисы для освобождения системных ресурсов:

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

**Совет:** Отключайте только те сервисы, которые вам не нужны. Изучите каждый сервис перед отключением, чтобы не сломать функционал, на который вы полагаетесь. Также вы можете искать сервисы вручную в интернете/некоторых приложениях

-----

## Параметры ядра GRUB

### Настройка

Отредактируйте `/etc/default/grub` и измените командную строку ядра:

```bash
sudo nano /etc/default/grub
```

Добавьте эти параметры в `GRUB_CMDLINE_LINUX`:

```bash
GRUB_CMDLINE_LINUX="quiet splash amdgpu.ppfeaturemask=0xffffffff split_lock_detect=off amd_pstate=active page_alloc.shuffle=1 pci=pcie_bus_perf pcie_aspm.policy=performance usbcore.autosuspend=-1 nowatchdog nmi_watchdog=0 zswap.enabled=0"
```
### Расшифровка параметров:

amd_pstate=active: **КРИТИЧНО для Zen 3/4/5. Включает драйвер EPP для разгона с миллисекундной скоростью. (Пользователи Intel: удалите этот параметр).**

amdgpu.ppfeaturemask=0xffffffff: **КРИТИЧНО для AMD GPU. Разблокирует контроль напряжения и лимиты разгона (необходимо для LACT/CoreCtrl).**

split_lock_detect=off: **Предотвращает крэши SIGBUS и фризы в играх, использующих "split locks" (античит часто это делает).**

nowatchdog & nmi_watchdog=0: **Отключает watchdog-таймеры, тратящие циклы CPU. Уменьшает микрофризы.**

zswap.enabled=0: **Отключает ZSWAP для снижения задержек. (Делайте это только если у вас 32GB+ RAM. Если у вас 4-16GB, удалите этот параметр).**

pci=pcie_bus_perf: **Принудительно устанавливает максимальный размер payload для шины PCIe (Максимальная пропускная способность GPU/NVMe).**

pcie_aspm.policy=performance: **Отключает состояния энергосбережения PCIe. Исправляет скачки задержек в idle. (это лучше чем pcie_aspm=off)!**

usbcore.autosuspend=-1: **Предотвращает переход USB-устройств в спящий режим. Исправляет задержку "пробуждения".**

## Экспериментальные (Используйте на свой риск):
**mitigations=off: Отключает патчи безопасности CPU.**

Хорошо для: **Старых CPU (Zen 1/2/3, Intel 9-го поколения и старше).** ~3% прирост fps.

Плохо для: **Zen 4 / Zen 5 (Ryzen 7000/9000).** Может фактически снизить 1% low FPS из-за логики предсказания переходов. **Пользователям Zen 4 НЕ следует использовать это!**

### Обновление конфигурации GRUB

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

-----

## Продвинутые системные твики

### Управление памятью

**Включите systemd-oomd (демон Out-of-Memory):**

```bash
sudo systemctl enable --now systemd-oomd
```

### Оптимизация накопителя

**Включите TRIM для SSD:**

```bash
# Включение автоматического TRIM
sudo systemctl enable --now fstrim.timer

# Запуск ручного TRIM
sudo fstrim -v /
```

### Настройка масштабирования CPU

**Для систем AMD:**

```bash
echo "active" | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

**Для систем Intel:**

```bash
echo "passive" | sudo tee /sys/devices/system/cpu/intel_pstate/status
```

### Системные лимиты

**Увеличьте лимит файловых дескрипторов** в `/etc/security/limits.conf`:

```bash
# Замените 'yourusername' на ваше реальное имя пользователя
yourusername hard nofile 1048576
yourusername soft nofile 1048576
```

### IRQ Balance (пользователи Intel iGPU)

Если возникают проблемы с производительностью при использовании встроенной графики Intel:

```bash
# Проверка статуса
sudo systemctl status irqbalance

# Отключение при необходимости
sudo systemctl disable --now irqbalance
```

### Настройка планировщика I/O

Современные системы Linux используют правила udev для настройки планировщиков I/O для каждого типа устройства. Параметр ядра `elevator=` устарел и больше не работает на новых версиях ядра

**Создайте правило udev для оптимального планирования I/O:**
```bash
sudo tee /etc/udev/rules.d/60-ioschedulers.rules
# HDD (вращающиеся диски) - используйте mq-deadline для лучшей производительности
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"

# SSD (невращающиеся диски) - используйте mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"

# NVMe SSD - используйте 'none' для лучшей производительности
# NVMe-диски имеют собственное продвинутое управление очередями и не получают выгоды от дополнительного планирования
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

спасибо netarchy за упоминание этого нового метода!
```

**Применить изменения немедленно:**
```bash
# Перезагрузить правила udev
sudo udevadm control --reload-rules
sudo udevadm trigger

# Проверить текущие планировщики I/O
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

# Проверка установки
gamemoded -t
```

**Использование:** Запускайте игры с префиксом `gamemoderun` или настройте в параметрах запуска Steam.

### Совместимость с Windows-играми

**PortProton** предлагает отличную совместимость для исполняемых файлов Windows (я использую portproton вместо Lutris / Bottles и это моё любимое приложение для запуска proton!):

```bash
sudo dnf copr enable boria138/portproton
sudo dnf install portproton
```

### Оптимизация Steam

Если вы установили **ядро CachyOS**, у вас есть доступ к функциям NTSync и FSR4 (видеокарты amd rdna 4 (rx90**)).

**Для нативных HDR-игр (Cyberpunk, Elden Ring и т.д.):**
```bash
MANGOHUD_CONFIG="fps_limit=277,no_display" mangohud LD_PRELOAD="" AMD_USERQ=1 PROTON_NTSYNC=1 game-performance gamescope -W 2560 -H 1440 -r 277 --hdr-enabled --force-grab-cursor --adaptive-sync --sharpness 2 -f -- %command%
```
## Переменные окружения и обёртки

MANGOHUD_CONFIG="fps_limit=277,no_display": **Настраивает слой MangoHud в тихом режиме.**

fps_limit=277: **Это ваш ручной режим "низкой задержки" (как amd antilag). Возьмите максимальную частоту обновления вашего монитора и вычтите 3 (например, 280Hz - 3 = 277 FPS). Это удерживает частоту кадров строго в диапазоне FreeSync, предотвращая давление V-Sync и минимизируя задержку ввода.**

no_display: **Скрывает оверлей, чтобы вы не видели статистику, но ограничитель всё равно работает в фоне.**

mangohud: **Фактически внедряет слой HUD. Без этого вышеуказанная конфигурация ничего не делает.**

LD_PRELOAD="": **это исправляет проблемы с фризами Steam после игры примерно 25-30+ минут**

AMD_USERQ=1: **(Только RDNA 3/4) Включает пользовательские очереди. Позволяет игре напрямую общаться с аппаратными очередями GPU, обходя часть накладных расходов драйвера ядра. В основном бесплатная производительность CPU.**

PROTON_NTSYNC=1: **(Требуется ядро CachyOS) Заменяет старую эмуляцию fsync/esync на правильный драйвер на уровне ядра для потоков Windows. Значительно снижает накладные расходы CPU в сложных играх.**

gamemoderun: **Просто опция feral gamemode**

## Аргументы Gamescope (контейнер)

gamescope: **Микро-композитор от Valve. Изолирует игровое окно от рабочего стола, исправляя крэши при Alt-Tab и идеально обрабатывая разрешение/HDR.**

-W 2560 -H 1440: **Устанавливает внутреннее разрешение, которое игра считает текущим. Установите это на нативное разрешение вашего монитора (например 1920x1080 и т.д.)**

-r 280: **Устанавливает ограничение частоты обновления для контейнера Gamescope. Установите это точно на максимальные Hz вашего монитора (144hz, 180, 60, 100 и т.д.).**

--hdr-enabled: **Разблокирует вывод HDR. Необходимо для oled/ips для фактической активации режима HDR.**

--force-grab-cursor: **Предотвращает случайные клики мышью за пределами игрового окна на второй монитор.**

--adaptive-sync: **Явно включает VRR (FreeSync/G-Sync) внутри контейнера, чтобы не было разрывов изображения**

--sharpness 2: **Применяет высококачественный фильтр повышения резкости CAS.**
Как это работает: **Шкала от 0 до 20, где 0 - Максимальная резкость, а 20 - Минимальная резкость. Значение 2 обеспечивает очень чёткое, детализированное изображение.**

-f: **Заставляет контейнер отображаться в полноэкранном режиме.**

--: **Разделитель. Сообщает Gamescope "Мои настройки заканчиваются здесь, дальше начинается команда игры".**

%command%: **Steam автоматически заменяет это на фактический исполняемый файл игры.**

-----

## Обслуживание и очистка

### Управление кэшем пакетов

**Очистка кэша DNF:**

```bash
sudo dnf clean all
```

**Очистка системных журналов:**

```bash
# Сохранять только последние 7 дней логов
sudo journalctl --vacuum-time=7d

# Или ограничить по размеру (сохранять только 100MB)
sudo journalctl --vacuum-size=100M
```

### Автоматизированное обслуживание

Создайте простой скрипт обслуживания:

```bash
#!/bin/bash
# Сохраните как ~/maintenance.sh и сделайте исполняемым

echo "Запуск обслуживания системы..."

# Обновление системы
sudo dnf upgrade --refresh

# Очистка кэшей
sudo dnf clean all

# Очистка старых записей журнала
sudo journalctl --vacuum-time=7d

# Запуск TRIM на SSD
sudo fstrim -v /

echo "Обслуживание завершено!"
```

-----

## Рекомендации по окружению рабочего стола

### Лёгкие альтернативы

Для максимальной производительности рассмотрите эти лёгкие окружения рабочего стола (при установке с минимального iso):

- **Sway** - Тайловый композитор на базе Wayland (у меня 700mb в idle)
- **i3** - Тайловый оконный менеджер X11 (600mb в idle)
- **Hyprland** - Современный Wayland-композитор с анимациями (900mb в idle)
- **XFCE** - Лёгкий традиционный рабочий стол
- **LXQt** - Лёгкий рабочий стол на базе Qt

**Редакция KDE Plasma:**
KDE Plasma теперь официальная редакция Fedora наряду с Workstation (GNOME). 
Это означает лучшую интеграцию, поддержку и оптимизацию из коробки.

### Управление питанием ноутбука

**Установка инструментов оптимизации питания:**
```bash
sudo dnf install powertop tlp tlp-rdw

sudo systemctl enable --now tlp

# Настройка TLP для режима игр/производительности

sudo nano /etc/tlp.conf

# Установить: TLP_DEFAULT_MODE=performance (при подключении к сети)
```

### Продвинутое управление памятью

**Настройка поведения свопа:**
```bash
# Снижение swappiness для лучшей производительности
echo 'vm.swappiness=150' | sudo tee -a /etc/sysctl.conf

# Улучшение распределения памяти для игр и т.д.
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf
```

**Раздел гейминга в контейнерах/Flatpak**


### Оптимизация гейминга в контейнерах

**Оптимизация Steam Flatpak:**
```bash
# Установка Steam как Flatpak для изоляции
flatpak install com.valvesoftware.Steam

# Предоставление необходимых разрешений для игр
flatpak override --user --filesystem=~/.local/share/Steam com.valvesoftware.Steam
```


### Оптимизация GNOME

Если остаётесь с GNOME:

```bash
# Установка GNOME tweaks
sudo dnf install gnome-tweaks gnome-extensions-app

# Можете отключить анимации для лучшей производительности
gsettings set org.gnome.desktop.interface enable-animations false

# Снижение использования ресурсов
gsettings set org.gnome.shell.overrides workspaces-only-on-primary false
```

-----

## Оптимизация графических драйверов

<details>
<summary>🔴 Оптимизация графики AMD</summary>

### Установка драйвера AMD GPU

```bash
# Драйверы Mesa включены по умолчанию, убедитесь в последней версии
sudo dnf install mesa-vulkan-drivers mesa-vdpau-drivers mesa-va-drivers

# Установка ROCm для вычислений (опционально)
sudo dnf install rocm-opencl rocm-smi
```

### Твики производительности AMD

```bash
## Опционально! Некоторые из этих твиков могут снизить ваш fps или сделать игры нестабильными!
# Проверяйте производительность и стабильность игр после внесения любых изменений

sudo nano /etc/environment.d/99-amd-gaming.conf
# Включение потоковости GPU (действительно хорошая опция для minecraft, но может снизить fps в большинстве игр)
mesa_glthread=true

# Оптимизации RadeonSI (иногда драйвер radv работает хуже mesa. тестируйте всё!)
RADV_PERFTEST=gpl,nggc,sam,rt
AMD_VULKAN_ICD=RADV

# Аппаратное ускорение видео
VDPAU_DRIVER=radeonsi
LIBVA_DRIVER_NAME=radeonsi

# Включение Resizable BAR (если по какой-то причине rebar у вас не работает)
AMD_GPU_ALLOW_RESIZE_BAR=1
```

### Управление питанием AMD GPU

```bash
# Установка режима производительности (в терминале):
echo "performance" | sudo tee /sys/class/drm/card*/device/power_dpm_state
echo "high" | sudo tee /sys/class/drm/card*/device/power_profile

# Создание постоянного сервиса
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

    
### Включение AMD User Queues (специфично для RDNA 3/4)
Позволяет игре напрямую общаться с GPU, обходя накладные расходы ядра
Или можете просто добавить это в параметры запуска игры в Steam

```bash
AMD_USERQ=1
```


### AMD Lact (как AMD Adrenalin в Windows, позволяет настраивать смещение mv, частоту памяти, управление мощностью, кривую вентиляторов и т.д.)

```bash
### Поскольку мы добавили `amdgpu.ppfeaturemask=0xffffffff` в GRUB, теперь можете использовать LACT для андервольта/разгона карты для стабильности и более высоких boost-частот!

# Включение copr-репозитория:
sudo dnf copr enable ilyaz/LACT

# Установка пакета (альтернативные пакеты: lact-headless, lact-libadwaita):
sudo dnf install lact

# Включение сервиса:
sudo systemctl enable --now lactd

## Что касается пользователей rx9070xt, рекомендую начать с "Performance Level: Manual", смещение напряжения -70mv, заблокировать частоту памяти на 2714Mhz, установить Power Limit на 355W. Для андервольта установите PL на 260-270W.
```

</details>

<details>
<summary>🟢 Оптимизация графики NVIDIA</summary>

## Оптимизация графики NVIDIA для Fedora

> **Комплексное руководство по оптимизации для GPU NVIDIA на Fedora с дисплейным сервером Wayland**

### Системные требования NVIDIA

**Поддерживаемые GPU:**

- GTX 700/900/1000 серии и новее (Maxwell, Pascal, Turing, Ampere, Ada Lovelace, Blackwell)
- RTX 20/30/40/50 серии с полной поддержкой функций

**Совместимость драйверов:**

- **Лучше всего:** Драйверы NVIDIA 590+ для оптимальной поддержки Wayland и производительности
- **Примечание:** Стек драйверов NVIDIA значительно улучшил поддержку Wayland в последних версиях

-----

### Установка драйвера NVIDIA

#### Метод 1: RPM Fusion (Настоятельно рекомендуется)

RPM Fusion остаётся наиболее надёжным методом для драйверов NVIDIA на Fedora. Этот подход обеспечивает правильную интеграцию с дисплейным сервером Wayland и системными обновлениями.

```bash
# Включение репозиториев RPM Fusion (если ещё не включены)
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Обновление системных пакетов
sudo dnf update

# Установка драйверов NVIDIA с поддержкой Wayland
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# Установка 32-битных библиотек совместимости (необходимо для Steam, Wine, игр)
sudo dnf install xorg-x11-drv-nvidia-libs.i686

# Установка утилиты настроек NVIDIA
sudo dnf install nvidia-settings
```

#### Проверка после установки

Понимание того, что говорит каждая команда, помогает убедиться, что ваша система правильно настроена для оптимальной производительности.

```bash
# Проверка установки драйвера и версии
nvidia-smi
# Должна показать ваш GPU, версию драйвера (570+) и текущую загрузку

# Подтверждение доступности поддержки CUDA
nvidia-smi -q | grep "CUDA Version"
# Необходимо для AI-нагрузок и некоторых игр, использующих GPU-вычисления

# Проверка использования NVIDIA GPU в Wayland
echo $XDG_SESSION_TYPE
# Должно вывести "wayland"

# Проверка работы GBM backend
nvidia-smi --query-gpu=name,driver_version --format=csv
# Подтверждает правильную загрузку драйвера
```

#### Включение Wayland для NVIDIA (Важный шаг)

Wayland требует специальной настройки для правильной работы с драйверами NVIDIA. Этот шаг обеспечивает возможность вашего окружения рабочего стола использовать аппаратное ускорение.

```bash
# Включение режима настройки ядра DRM (требуется для Wayland)
echo 'options nvidia_drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-drm-modeset.conf

# Включение ранней загрузки модулей NVIDIA
echo -e 'nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm' | sudo tee /etc/modules-load.d/nvidia.conf

# Пересборка initramfs для включения изменений
sudo dracut --force

# Перезагрузка для применения изменений модулей ядра
sudo reboot
```

-----

### Оптимизация производительности NVIDIA Wayland

#### 1. Переменные окружения для Wayland

Эти переменные окружения оптимизируют поведение NVIDIA GPU специально для композиторов Wayland. В отличие от X11, Wayland автоматически обрабатывает многие оптимизации, но эти переменные тонко настраивают производительность.

Добавьте в `/etc/environment`:

```bash
# Основные оптимизации NVIDIA Wayland
#
# Критическое предупреждение для современных NVIDIA GPU (RTX 20 серии и новее)
#
# На основе отзывов пользователей и тестирования, следующие две переменные окружения (`GBM_BACKEND` и `__GLX_VENDOR_LIBRARY_NAME`) могут вызывать серьёзные системные задержки ввода, фризы и неотзывчивость приложений на NVIDIA RTX 20, 30, 40 и 50 серий
#
# ! Рекомендация для RTX 20-серии и новее: НЕ используйте эти переменные. Современные драйверы nvidia и wayland-композиторы обычно обрабатывают эту настройку автоматически. Их ручное включение может создать конфликты
# ! Рекомендация для старых gpu (GTX 10-серии и старше): Эти переменные всё ещё могут быть полезны для обеспечения совместимости wayland на старом железе. можете попробовать их. любой отчёт о проблеме с конкретной gpu будет очень ценным! :)
#
#
# Только для старых карт (GTX 10-серии и ниже), вам может понадобиться:
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia
#
# Включение потоковых оптимизаций (улучшает параллелизм CPU-GPU)
__GL_THREADED_OPTIMIZATIONS=1
# Предупреждение: Опция __GL_THREADED_OPTIMIZATIONS может вызывать чёрные экраны на некоторых RTX-картах (См. раздел решения проблем NVIDIA Wayland)
# Установите это для каждой игры отдельно (см. раздел решения проблем)
# Или протестируйте и установите для окружения, если столкнётесь с чёрным экраном - войдите через tty, удалите __GL_THREADED_OPTIMIZATION=1 из /etc/environment, сохраните и перезагрузитесь.
# краткий гайд по tty
#    - нажмите Ctrl+Alt+F3 (или F2–F6) для переключения на экран входа TTY.
#    - войдите с вашим именем пользователя и паролем.
#
# 2. отредактируйте /etc/environment и удалите проблемную строку:
# sudo nano /etc/environment
#    - найдите строку:
#        __GL_THREADED_OPTIMIZATIONS=1
#    - удалите её, затем сохраните (Ctrl+O, Enter) и выйдите (Ctrl+X).
#
# 3. перезагрузите систему:
# sudo reboot
# Кстати, всё равно рекомендуется устанавливать эту опцию только для каждой игры отдельно
# Спасибо @lemonadeforlife за указание на эту проблему и решение

# Кэширование компиляции шейдеров (уменьшает фризы в играх)
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_PATH=/tmp/nvidia-shader-cache
__GL_SHADER_DISK_CACHE_SIZE=1073741824

# Отключение VSync для игр (уменьшает задержку ввода)
__GL_SYNC_TO_VBLANK=0

# Включение неофициальных расширений протокола для совместимости с Wayland
__GL_ALLOW_UNOFFICIAL_PROTOCOL=1

# Оптимизации для игр
# Включает NVAPI для функций типа DLSS в Proton, пользователи rtx gpu протестируйте это пожалуйста
PROTON_ENABLE_NVAPI=1
NVIDIA_DRIVER_CAPABILITIES=all

# Хорошие флаги для каждой игры в Steam/PortProton/Lutris и т.д., найденные на reddit и используемые сообществом
PROTON_ENABLE_WAYLAND=1
LD_PRELOAD=""
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
%command%
-fullscreen;

# Следующие твики не рекомендуются глобально. Используйте их только если знаете, что они вам нужны.

# ТОЛЬКО для композиторов wl-roots (Sway, Hyprland)
# Это может помочь с графическими глюками, но не нужно и не рекомендуется на GNOME/KDE.
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1
```

#### 2. Параметры модулей ядра (используйте если знаете что делаете)

Современные драйверы NVIDIA выигрывают от специфических параметров ядра, которые могут улучшить производительность
Тестируйте каждую опцию, работает ли она в вашем случае и с вашим железом

Создайте `/etc/modprobe.d/nvidia-power-management.conf`:

```bash
# Включение современных функций управления питанием
options nvidia NVreg_DynamicPowerManagement=0x02

# Включение таблицы атрибутов страниц (улучшает производительность памяти)
options nvidia NVreg_UsePageAttributeTable=1

# Включение поддержки ResizableBAR (RTX 30/40/50 серии)
options nvidia NVreg_EnableResizableBar=1

# Сохранение видеопамяти во время suspend
options nvidia NVreg_PreserveVideoMemoryAllocations=1

# Включение операций потоковой памяти (требуется для некоторых нагрузок)
options nvidia NVreg_EnableStreamMemOPs=1
```

#### 3. Специфические настройки GNOME Wayland

GNOME на Wayland требует особого внимания для достижения оптимальной производительности NVIDIA. Эти настройки решают распространённые проблемы с композитором GNOME.

```bash
# Включение ускорения NVIDIA для сессии GNOME Wayland
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"

# Настройка GNOME для игровой производительности
gsettings set org.gnome.shell.extensions.dash-to-dock click-action 'cycle-windows'
gsettings set org.gnome.desktop.interface enable-animations false

# Установка масштаба для high-DPI дисплеев (настройте по необходимости)
gsettings set org.gnome.desktop.interface scaling-factor 1
```

#### 4. Настройка KDE Plasma Wayland

KDE Plasma имеет отличную поддержку Wayland и особенно хорошо работает с драйверами NVIDIA при правильной настройке.

```bash
# Включение поддержки переменной частоты обновления (или используя настройки kde, что лучше)
kwriteconfig6 --file kwinrc --group Compositing --key VariableRefreshRate true

# Оптимизация настроек композитора для игр
kwriteconfig6 --file kwinrc --group Compositing --key LatencyPolicy Low
kwriteconfig6 --file kwinrc --group Compositing --key RenderTimeEstimator 1

# Перезапуск KWin для применения изменений
qdbus org.kde.KWin /KWin reconfigure
```

-----

### Специфические оптимизации NVIDIA для игр

#### 1. Параметры запуска Steam для Wayland

Игры в Steam на Wayland требуют специфических параметров запуска для обеспечения правильного использования NVIDIA GPU и достижения оптимальной производительности.

**Для нативных Linux-игр:**

```bash
# Базовая оптимизация с GameMode
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 %command%

# Улучшенная производительность для соревновательных игр
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 %command%
```

**Для Proton/Wine игр:**

```bash
# Стандартная оптимизация Proton
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Продвинутая оптимизация с асинхронной компиляцией dxvk/gpl
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 DXVK_ASYNC=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%

# Для игр, требующих максимальной производительности
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 DXVK_ASYNC=1 PROTON_ENABLE_NVAPI=1 LD_PRELOAD="" __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 %command%
```

#### 2. Оптимизация игр в Lutris

Lutris обеспечивает отличную интеграцию с драйверами NVIDIA на Wayland. Настройте эти параметры для оптимальной игровой производительности.

```bash
# Установка Lutris с поддержкой NVIDIA
sudo dnf install lutris wine

# Настройка переменных окружения Lutris (в настройках Lutris)
# Добавьте это в Системные опции → Переменные окружения:
__GL_THREADED_OPTIMIZATIONS=1
__GL_SHADER_DISK_CACHE=1
PROTON_ENABLE_NVAPI=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
```

#### 3. Интеграция GameMode

GameMode автоматически применяет системные оптимизации во время игры. Хотя он хорошо работает из коробки, вы можете тонко настроить его поведение.

```bash
# Установка GameMode
sudo dnf install gamemode

# Настройка GameMode для оптимизации NVIDIA
sudo nano /etc/gamemode.ini
[general]
renice=10
ioprio=1

[gpu]
apply_gpu_optimisations=accept-responsibility
gpu_device=0
```

-----

### Продвинутые твики NVIDIA Wayland

#### 1. Поддержка переменной частоты обновления (VRR)

Современные драйверы NVIDIA поддерживают переменную частоту обновления на Wayland, обеспечивая более плавный игровой опыт с совместимыми мониторами.

```bash
# Включение VRR в GNOME
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"
# Или в настройках дисплея (если поддерживается)

# Проверка работы VRR
sudo dnf install drm_info
drm_info | grep -i vrr
```

#### 2. Поддержка HDR (Экспериментальная)

Поддержка расширенного динамического диапазона постепенно улучшается на Wayland с драйверами NVIDIA. Эти настройки включают экспериментальную функциональность HDR.

```bash
# Включение поддержки HDR (требуется совместимый дисплей и последние драйверы)
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf

# Поддержка HDR в GNOME (экспериментально!)
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate','hdr']"
```

#### 3. Мониторинг и настройка производительности

Эффективный мониторинг производительности помогает выявлять узкие места и проверять, что оптимизации работают правильно.

```bash
# Установка инструментов мониторинга
sudo dnf install nvtop mangohud goverlay

# Создание скрипта мониторинга для игровых сессий
sudo nano /usr/local/bin/nvidia-monitor.sh
#!/bin/bash
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
nvidia-smi dmon -s pucvmet

sudo chmod +x /usr/local/bin/nvidia-monitor.sh
```

-----

### Управление питанием и термооптимизация

#### Продвинутое управление питанием (применяйте только если знаете что делаете)

Правильное управление питанием обеспечивает стабильную производительность при предотвращении ненужного потребления энергии в простое.

```bash
# Настройка продвинутого управления питанием
echo 'options nvidia NVreg_DynamicPowerManagement=0x02' | sudo tee -a /etc/modprobe.d/nvidia-power.conf

# Включение runtime power management для ноутбуков
sudo nano /etc/udev/rules.d/80-nvidia-pm.rules
# Включение runtime PM для устройств NVIDIA VGA/3D контроллера
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
```

-----

### Решение проблем NVIDIA Wayland

#### Распространённые проблемы и современные решения

Понимание того, как диагностировать и решать проблемы, обеспечивает оптимальную производительность и стабильность системы.

**1. Сессия Wayland не запускается с NVIDIA:**

Это самая распространённая проблема при переходе с X11 на Wayland с драйверами NVIDIA.

```bash
### Чёрный экран NVIDIA при загрузке

**Проблема:** Система показывает чёрный экран после загрузки с драйверами NVIDIA

**Причина:** Переменная окружения `__GL_THREADED_OPTIMIZATIONS=1` в `/etc/environment` может вызывать проблемы с инициализацией дисплея на некоторых RTX GPU

**Решение:**
1. Загрузитесь в режим восстановления (удерживайте Shift во время загрузки для доступа к меню GRUB)
2. Отредактируйте `/etc/environment` и удалите или закомментируйте:
   ```bash
   # __GL_THREADED_OPTIMIZATIONS=1

# Проверьте правильность параметров модуля ядра
cat /etc/modprobe.d/nvidia-drm-modeset.conf
# Должно содержать: options nvidia_drm modeset=1 fbdev=1

# Проверьте включён ли DRM modeset
cat /sys/module/nvidia_drm/parameters/modeset
# Должно вывести: Y

# Пересоберите initramfs и перезагрузитесь при необходимости
sudo dracut --force
sudo reboot

# Проверьте сессию Wayland после перезагрузки
echo $XDG_SESSION_TYPE
# Должно вывести: wayland
```

**2. Плохая игровая производительность несмотря на хорошее железо:**

Проблемы с производительностью часто связаны с неправильным использованием GPU или настройками управления питанием.

```bash
# Проверьте используется ли GPU
nvidia-smi dmon -s pucvmet -c 10

# Проверьте ограничение питания
nvidia-smi --query-gpu=power.limit,power.draw --format=csv
# Потребление энергии должно приближаться к лимиту во время игры

# Мониторьте частоты GPU во время игры
watch -n 1 'nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv'
# Частоты должны достигать максимальных значений во время игры
```

**3. Разрывы изображения или фризы:**

Современные композиторы Wayland лучше обрабатывают разрывы чем X11, но может потребоваться некоторая настройка.

```bash
# Для GNOME убедитесь что VRR включён
gsettings get org.gnome.mutter experimental-features
# Должно включать 'variable-refresh-rate'

# Для игр отключите VSync в игре и используйте VSync композитора
# Добавьте в параметры запуска Steam:
__GL_SYNC_TO_VBLANK=0 %command%
```

**4. Высокое потребление энергии в простое:**

Предотвращение ненужного потребления энергии в простое улучшает время работы от батареи и снижает нагрев.

```bash
# Включение runtime power management
echo 'auto' | sudo tee /sys/bus/pci/devices/0000:*/power/control

# Проверка работы управления питанием
cat /sys/bus/pci/devices/0000:*/power/runtime_status
# Должно показывать 'suspended' для простаивающего GPU

# Мониторинг потребления энергии в простое
nvidia-smi --query-gpu=power.draw --format=csv --loop=1
```

-----

### Мониторинг производительности и бенчмаркинг

#### Комплексная настройка мониторинга

Эффективный мониторинг помогает оптимизировать производительность и выявлять потенциальные проблемы до того, как они повлияют на игры или работу.

```bash
# Установка комплексного пакета мониторинга
sudo dnf install nvtop btop mangohud goverlay
```

#### Игровой оверлей производительности

MangoHud предоставляет метрики производительности в реальном времени во время игровых сессий.

```bash
# Настройка MangoHud для оптимального отображения
mkdir -p ~/.config/MangoHud

nano ~/.config/MangoHud/MangoHud.conf
# Информация о GPU и CPU
gpu_stats
cpu_stats
gpu_temp
cpu_temp

# Частота кадров и тайминги
fps
frametime
frame_timing

# Использование памяти
vram
ram

# Позиция и внешний вид
position=top-left
font_size=22
alpha=0.8

# Ограничение логирования для предотвращения влияния на производительность
log_duration=60
```

-----

### Дополнительные ресурсы

**Официальная документация NVIDIA:**
- [Руководство по установке драйвера NVIDIA Linux](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
- [Руководство по установке CUDA для Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

**Ресурсы для Fedora:**
- [Гайд по NVIDIA от RPM Fusion](https://rpmfusion.org/Howto/NVIDIA)

**Wayland и гейминг:**
- [Гейминг на Linux с NVIDIA](https://www.gamingonlinux.com/)
- [Документация MangoHud](https://github.com/flightlessmango/MangoHud)

-----

</details>

-----

## Мониторинг и проверка

### Инструменты мониторинга производительности

```bash
# Установка инструмента мониторинга
sudo dnf install btop

# Для детальной информации о системе
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

### Распространённые проблемы

**1. Проблемы с загрузкой после параметров ядра:**

- Загрузитесь с предыдущим ядром из меню GRUB
- Удалите проблемные параметры из `/etc/default/grub`
- Пересгенерируйте конфигурацию GRUB

**2. Проблемы с графикой:**

- Проверьте установку драйвера: `lspci -k | grep -A 2 -E "(VGA|3D)"`
- Проверьте правильную загрузку драйвера: `lsmod | grep -E "(amdgpu|nvidia|i915)"`

**3. Регрессия производительности:**

- Мониторьте системные ресурсы: `htop`, `iotop`
- Проверьте термотроттлинг: `watch sensors`
- Проверьте статус сервисов: `systemctl list-units --failed`

### Команды восстановления

```bash
# Сброс GRUB к значениям по умолчанию
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Сброс контекста SELinux (при повторном включении SELinux)
sudo restorecon -R /

# Проверка целостности системы
sudo dnf check
sudo rpm -Va
```

-----

## Ожидаемый прирост производительности

На основе тестирования, пользователи могут ожидать:

- **Время загрузки:** Улучшение на 10-20%
- **Игровая производительность:** Прирост FPS на 5-15%
- **Отзывчивость системы:** Значительное снижение задержки ввода
- **Использование памяти:** Снижение использования RAM в простое на 5-15%
- **Производительность накопителя:** Улучшенная производительность SSD с trim

-----

-----

## Участие в разработке

Нашли улучшения или есть предложения? Не стесняйтесь:

- Открыть issue на GitHub
- Отправить pull request
- Поделиться результатами оптимизации

-----

## Дополнительные ресурсы

- [Документация Fedora](https://docs.fedoraproject.org/)
- [RPM Fusion](https://rpmfusion.org/)
- [Ядро CachyOS](https://github.com/CachyOS/linux-cachyos)
- [Ananicy-cpp](https://gitlab.com/ananicy-cpp/ananicy-cpp)
- [Гейминг на Linux](https://www.gamingonlinux.com/)

-----

## Дисклеймер

Это руководство изменяет системные настройки, которые могут повлиять на стабильность и безопасность. Всегда:

- Создавайте резервные копии системы перед применением изменений
- Понимайте последствия каждого изменения
- Держите под рукой носители для восстановления

**Результаты производительности могут варьироваться** в зависимости от конфигурации железа и конкретных случаев использования.

-----

<details>
<summary>Список изменений</summary>
 
- **v1.0** - Создание первоначального гайда
- **v1.1** - Добавлен раздел решения проблем и русский перевод
- **v1.2** - Улучшен инструментами мониторинга и разделом обслуживания
- **v1.3** - Добавлены драйверы NVIDIA, гайд по производительности и многое другое
- **v1.4** - Добавлен раздел твиков для AMD gpu
- **v1.5** - Добавлен раздел быстрой навигации для лучшей юзабилити
- **v1.6** - Изменено с UKSMD на KSMD, более глубокое описание настройки ядра cachyos
- **v1.7** - Обновлено с большим количеством nvidia флагов, исправлениями, подготовка к релизу Fedora 43
- **v1.8** - Обновлены почти все разделы гайда, особенно на базе AMD для ещё большего прироста производительности
</details>

-----

*Последнее обновление: Январь 2026*
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
</details>

-----

*Last updated: January 2026*
