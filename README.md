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

- **Period:** October 14, 2024 - December 21, 2025
- **Distribution:** Fedora 43
- **Additional Testing:** NVIDIA and AMD gpu systems
- **These optimizations may also work on any other distro, but i cannot guarantee that all these tweaks will be good on other distro / your system. It is always necessary to test everything. 80% of tweaks work on Arch and NixOS :)**

**Hardware Configurations (tested on):**

- **First:** Ryzen 5 5500U, 20GB DDR4, RX550X discrete/RX Vega 7 iGPU, NVMe
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
# We will set `bpfland` as our default scheduler, as it provides an excellent balance for gaming and desktop usage. Create the configuration file with this command:

sudo nano /etc/scx_loader/config.toml
# Set the bpfland scheduler as default and configure it for gaming mode.
default_sched = "scx_bpfland"
default_mode = "Gaming"
 
[scheds.scx_bpfland]
auto_mode = []
gaming_mode = ["-m", "performance"]
lowlatency_mode = ["-s", "5000", "-S", "500", "-l", "5000", "-m", "performance"]
powersave_mode = ["-m", "powersave"]


# Step 2: Enable and Start the Scheduler Service
sudo systemctl enable --now scx_loader

# Step 3: Verify the Change
dbus-send --system --print-reply --dest=org.scx.Loader /org/scx/Loader org.freedesktop.DBus.Properties.Get string:org.scx.Loader string:CurrentScheduler
# the output should show string "scx_bpfland".

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
GRUB_CMDLINE_LINUX="quiet lpj=XXXXXXX mitigations=off skew_tick=1 nowatchdog page_alloc.shuffle=1 pci=pcie_bus_perf intel_idle.max_cstate=1 processor.max_cstate=1 libahci.ignore_sss=1 noautogroup amd_pstate=active"
```

**Get the LPJ value:**

```bash
sudo dmesg | grep -o "lpj=[0-9]*"
# Then replace XXXXXXX in GRUB_CMDLINE_LINUX with the value shown in the output
```

### Update GRUB Configuration

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Parameter Explanations:**

- `mitigations=off` - Disables CPU vulnerability mitigations for better performance
- `nowatchdog` - Disables hardware watchdog
- `intel_idle.max_cstate=1` - Limits CPU idle states for lower latency (intel only)
- `amd_pstate=active` - Enables AMD P-State driver for better power management

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

### Graphics Optimization (AMD Users)

Add to `/etc/environment`:

```bash
# AMD GPU optimizations
RADV_PERFTEST=gpl
mesa_glthread=true
AMD_VULKAN_ICD=RADV
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

Add to Steam launch options for games:

```bash
gamemoderun %command%
```

Or for Proton games:

```bash
gamemoderun DXVK_ASYNC=1 %command%
```

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
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

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
# Create AMD GPU optimization config
sudo nano /etc/environment.d/99-amd-gaming.conf
# Enable GPU Threading
mesa_glthread=true

# RadeonSI optimizations
RADV_PERFTEST=gpl,nggc,sam,rt
AMD_VULKAN_ICD=RADV

# Video acceleration
VDPAU_DRIVER=radeonsi
LIBVA_DRIVER_NAME=radeonsi

# Enable Resizable BAR
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

### AMD Memory Overclocking (Advanced, dont attempt to do this without experience)

```bash
# Check current memory clock
cat /sys/class/drm/card*/device/pp_dpm_mclk

# Overclock VRAM (example, adjust values carefully)
echo "manual" | sudo tee /sys/class/drm/card*/device/power_dpm_force_performance_level
echo "3" | sudo tee /sys/class/drm/card*/device/pp_dpm_mclk
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

# Install additional tools for monitoring
sudo dnf install nvidia-ml-py3
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

### ⚡ NVIDIA Wayland Performance Optimizations

#### 1. Environment Variables for Wayland

These environment variables optimize NVIDIA GPU behavior specifically for Wayland compositors. Unlike X11, Wayland handles many optimizations automatically, but these variables fine-tune performance.

Add to `/etc/environment`:

```bash
# Core NVIDIA Wayland optimizations
#
# Critical Warning for Modern NVIDIA GPUs (RTX 20-Series and Newer)
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

#### 2. Kernel Module Parameters

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
# Enable variable refresh rate support
kwriteconfig5 --file kwinrc --group Compositing --key VariableRefreshRate true

# Optimize compositor settings for gaming
kwriteconfig5 --file kwinrc --group Compositing --key LatencyPolicy Low
kwriteconfig5 --file kwinrc --group Compositing --key RenderTimeEstimator 1

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

# Advanced optimization with DXVK async compilation
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

# For KDE Plasma, enable in system settings or via command:
kwriteconfig5 --file kwinrc --group Compositing --key VariableRefreshRate true

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
sudo nano /usr/local/bin/nvidia-gaming-monitor.sh
#!/bin/bash
echo "=== NVIDIA Performance Monitor ==="
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
echo "=== Real-time Stats ==="
nvidia-smi dmon -s pucvmet

sudo chmod +x /usr/local/bin/nvidia-gaming-monitor.sh
```

-----

### Power Management and Thermal Optimization

#### 1. Advanced Power Management

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

#### 2. Thermal Management

Effective thermal management prevents throttling and maintains optimal performance during extended gaming sessions.

```bash
# Install thermal monitoring tools
sudo dnf install lm_sensors

# Configure sensor detection
sudo sensors-detect --auto

# Create thermal monitoring script
sudo nano /usr/local/bin/nvidia-thermal.sh
#!/bin/bash
TEMP=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits)
POWER=$(nvidia-smi --query-gpu=power.draw --format=csv,noheader,nounits)

echo "GPU Temperature: ${TEMP}°C"
echo "Power Draw: ${POWER}W"

# Alert if temperature is high
if [ $TEMP -gt 83 ]; then
    echo "WARNING: GPU temperature is high!"
    notify-send "GPU Temperature Warning" "GPU is running at ${TEMP}°C"
fi


sudo chmod +x /usr/local/bin/nvidia-thermal.sh
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

### NVIDIA Developer and AI Tools

#### CUDA Development Environment

Setting up CUDA properly ensures compatibility with AI frameworks and development tools.

```bash
# Install CUDA toolkit
sudo dnf install cuda-toolkit cuda-devel cuda-runtime

# Configure environment variables
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export CUDA_HOME=/usr/local/cuda' >> ~/.bashrc

# Reload environment
source ~/.bashrc

# Verify CUDA installation
nvcc --version
nvidia-smi --query-gpu=compute_cap --format=csv
```

#### Container Support for AI/ML Workloads

Container support enables easy deployment of AI and machine learning applications.

```bash
# Install NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

sudo dnf install nvidia-container-toolkit

# Configure Docker/Podman
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Test container support
sudo docker run --rm --gpus all nvidia/cuda:12.7-runtime-ubuntu25.04 nvidia-smi
```

-----

### Performance Monitoring and Benchmarking

#### Comprehensive Monitoring Setup

Effective monitoring helps optimize performance and identify potential issues before they impact gaming or work performance.

```bash
# Install comprehensive monitoring suite
sudo dnf install nvtop btop mangohud goverlay

# Create performance monitoring script
sudo nano /usr/local/bin/nvidia-perf-monitor.sh
#!/bin/bash
clear
echo "=== NVIDIA Performance Monitor ==="
echo "System: $(hostnamectl --static) | $(date)"
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo ""

# GPU utilization and memory
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw,clocks.gr,clocks.mem --format=csv

echo ""
echo "=== Active GPU Processes ==="
nvidia-smi pmon -c 1

echo ""
echo "Press Ctrl+C to exit continuous monitoring..."
watch -n 2 nvidia-smi


sudo chmod +x /usr/local/bin/nvidia-perf-monitor.sh
```

#### Gaming Performance Overlay

MangoHud provides real-time performance metrics during gaming sessions.

```bash
# Configure MangoHud for optimal display
mkdir -p ~/.config/MangoHud

nano > ~/.config/MangoHud/MangoHud.conf
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
# Install useful monitoring tools
sudo dnf install htop iotop powertop fastfetch

# For detailed system information
sudo dnf install hardinfo
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
    
# 🚀 Руководство по оптимизации Fedora для игр и производительности

# ОБНОВЛЯЕТСЯ С ЗАДЕРЖКОЙ, если хотите видеть самые последние и свежие изменения в этом гайде, то читайте английскую версию

## 🧭 Быстрая навигация

| Настройка и ядро                                                                  | Система и игры                                                                 | Ресурсы и другое                                                                       |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| [**Информация о системе**](#-информация-о-системе)                                | [**Продвинутые настройки**](#-продвинутые-системные-настройки)                  | [**Мониторинг и проверка**](#-мониторинг-и-проверка)                                    |
| [**Первоначальная настройка**](#-первоначальная-настройка-и-подготовка)            | [**Игровые оптимизации**](#-игровые-оптимизации)                               | [**Устранение неполадок**](#-устранение-неполадок)                                      |
| [**Оптимизация ядра**](#-оптимизация-ядра)                                        | [**Обслуживание и очистка**](#-обслуживание-и-очистка)                         | [**Полная русская версия**](#-русская-версия--russian-translation)                      |
| [**Параметры ядра GRUB**](#️-параметры-ядра-grub)                                    | [**Оптимизация драйверов**](#️-оптимизация-графических-драйверов)              |                                                                                        |

> **Полное руководство по оптимизации Fedora 42 для игр и максимальной производительности | от winterofhell**

## 📋 Информация о системе

**Среда тестирования:**

- **Период проверки:** 14 октября 2024 - 7 сентября 2025
- **Дистрибутив:** Fedora 42 (Minimal ISO + Sway WM | Второй ПК: Fedora GNOME Edition | Fedora KDE Edition)
- **Дополнительное тестирование:** GNOME DE на системах с NVIDIA и AMD
- **Это может также работать на любом другом дистрибутиве, но я не могу гарантировать, что все эти настройки будут работать на вашей системе. Но 80% твиков работают на всех системах, включая Arch / NixOS :)**

**Конфигурации оборудования (протестировано на):**

- **Основная:** Ryzen 5 5500U, 20ГБ DDR4, RX550X дискретная/RX Vega 7 встроенная, NVMe диск
- **Вторая:** Ryzen 5 5600, 16ГБ DDR4, GTX 1060, SATA SSD
- **Новая:** Ryzen 5 7500f, 32Гб DDR5, RX 9070 XT, Nvme M2.

-----

## 🛠 Первоначальная настройка и подготовка

### 1. Минимальная установка

Для оптимальной производительности всегда начинайте с **Fedora Minimal ISO**. Этот iso исключает ненужные пакеты и службы, которые могут влиять на системные ресурсы. Но, это не обязательно делать.

### 2. Включение репозиториев RPM Fusion

RPM Fusion предоставляет необходимые мультимедийные кодеки и проприетарные драйверы:

```bash
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

📖 **Официальное руководство:** [Конфигурация RPM Fusion](https://rpmfusion.org/Configuration)

### 3. Конфигурация SELinux (опционально)

⚠️ **Предупреждение о безопасности:** Отключение SELinux снижает безопасность системы, но также делает вашу систему немного быстрее. Продолжайте только если понимаете последствия. (Лично я не забочусь о SELinux и всегда отключаю его.)

**Временное отключение (до перезагрузки):**

```bash
sudo setenforce 0
```

**Постоянное отключение (требует перезагрузки):**

```bash
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# Для применения изменений требуется перезагрузка
```

### 4. Обновление системы

Всегда начинайте с полностью обновленной системы:

```bash
sudo dnf upgrade --refresh

# Установите обновления прошивки (включая микрокод ЦП)
sudo dnf install linux-firmware intel-ucode amd-ucode
```

-----

## ⚡ Оптимизация ядра

### Установка ядра CachyOS

Ядро CachyOS обеспечивает значительное улучшение производительности для игр и общей отзывчивости системы.

**Предварительные требования:** ЦП должен поддерживать набор инструкций x86_64_v3 !!

```bash
# Проверка поддержки ЦП
# Проверьте поддержку следующей командой

/lib64/ld-linux-x86-64.so.2 --help | grep "(supported, searched)"

# Если это не обнаружит поддержку x86_64_v3, НЕ устанавливайте это ядро. Если обнаружит только x86_64_v2, вы можете использовать LTS ядро.
```

```bash
# Добавить репозиторий COPR CachyOS
sudo dnf copr enable bieszczaders/kernel-cachyos

# Установить ядро CachyOS
sudo dnf install kernel-cachyos kernel-cachyos-devel

# Для x86_64_v2 (старые CPUs):
sudo dnf install kernel-cachyos-lts kernel-cachyos-lts-devel-matched
```

📖 **Дополнительная информация:** [Установка ядра CachyOS](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/)

### Установка UKSMD

UKSMD (Userspace Kernel Same-page Merging Daemon) снижает использование памяти и улучшает отзывчивость системы:

```bash
# Добавить репозиторий дополнений UKSMD
sudo dnf copr enable bieszczaders/kernel-cachyos-addons

# Установить UKSMD
sudo dnf install uksmd

# Включить и запустить службу UKSMD
sudo systemctl enable --now uksmd
```

📖 **Дополнительная информация:** [Дополнения UKSMD](https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos-addons/)

-----

## 🔧 Оптимизация системных служб

### Установка Ananicy-cpp

Ananicy-cpp автоматически управляет приоритетами процессов и снижает задержки системы:

```bash
# Установить зависимости для сборки
sudo dnf group install "Development Tools"
sudo dnf install cmake systemd-devel spdlog-devel fmt-devel nlohmann-json-devel make automake gcc gcc-c++

# Клонировать и собрать
git clone https://gitlab.com/ananicy-cpp/ananicy-cpp.git
cd ananicy-cpp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install

# Включить службу
sudo systemctl enable --now ananicy-cpp
```

### Управление службами

Отключите ненужные службы для освобождения системных ресурсов:

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

💡 **Совет:** Отключайте только службы, которые вам не нужны. Просмотрите каждую службу перед отключением, чтобы не сломать функциональность, на которую вы полагаетесь.

-----

## ⚙️ Параметры ядра GRUB

### Конфигурация

Отредактируйте `/etc/default/grub` и измените командную строку ядра:

```bash
sudo nano /etc/default/grub
```

Добавьте эти параметры в `GRUB_CMDLINE_LINUX`:

```bash
GRUB_CMDLINE_LINUX="quiet lpj=XXXXXXX mitigations=off elevator=mq-deadline nowatchdog page_alloc.shuffle=1 pci=pcie_bus_perf intel_idle.max_cstate=1 processor.max_cstate=1 libahci.ignore_sss=1 noautogroup amd_pstate=active"
```

**Получить значение LPJ:**

```bash
sudo dmesg | grep -o "lpj=[0-9]*"
```

### Обновление конфигурации GRUB

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Объяснение параметров:**

- `mitigations=off` - Отключает смягчения уязвимостей ЦП для лучшей производительности
- `elevator=mq-deadline` - Использует планировщик ввода-вывода deadline (лучше для SSD, чем noop)
- `nowatchdog` - Отключает аппаратный watchdog
- `intel_idle.max_cstate=1` - Ограничивает состояния простоя ЦП для меньшей задержки (только Intel)
- `amd_pstate=active` - Включает драйвер AMD P-State для лучшего управления питанием

-----

## 🎯 Продвинутые системные настройки

### Управление памятью

**Включить systemd-oomd (демон нехватки памяти):**

```bash
sudo systemctl enable --now systemd-oomd
```

### Оптимизация хранилища

**Включить TRIM для SSD:**

```bash
# Включить автоматический TRIM
sudo systemctl enable --now fstrim.timer

# Запустить ручной TRIM
sudo fstrim -v /
```

### Оптимизация графики (пользователи AMD)

Добавить в `/etc/environment`:

```bash
# Оптимизации AMD GPU
RADV_PERFTEST=gpl
mesa_glthread=true
AMD_VULKAN_ICD=RADV
```

### Конфигурация масштабирования ЦП

**Для систем AMD:**

```bash
echo "active" | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

**Для систем Intel:**

```bash
echo "passive" | sudo tee /sys/devices/system/cpu/intel_pstate/status
```

### Системные лимиты

**Увеличить лимит дескрипторов файлов** в `/etc/security/limits.conf`:

```bash
# Замените 'yourusername' на ваше фактическое имя пользователя
yourusername hard nofile 1048576
yourusername soft nofile 1048576
```

### Балансировка IRQ (пользователи Intel iGPU)

Если возникают проблемы с производительностью с интегрированной графикой Intel:

```bash
# Проверить статус
sudo systemctl status irqbalance

# Отключить при необходимости
sudo systemctl disable --now irqbalance
```

-----

## 🎮 Игровые оптимизации

### Установка GameMode

GameMode применяет системные оптимизации во время игры:

```bash
sudo dnf install gamemode gamemode-devel

# Проверить установку
gamemoded -t
```

**Использование:** Запускайте игры с префиксом `gamemoderun` или настройте в параметрах запуска Steam.

### Совместимость игр Windows

**PortProton** предлагает отличную совместимость для исполняемых файлов Windows (я использую PortProton вместо Lutris/Bottles и это мой любимый исполнитель proton!):

```bash
sudo dnf copr enable boria138/portproton
sudo dnf install portproton
```

### Оптимизации Steam

Добавьте в параметры запуска Steam для игр:

```bash
gamemoderun %command%
```

Или для игр Proton:

```bash
gamemoderun DXVK_ASYNC=1 %command%
```

-----

## 🧹 Обслуживание и очистка

### Управление кэшем пакетов

**Очистить кэш DNF:**

```bash
sudo dnf clean all
```

**Очистить системные журналы:**

```bash
# Сохранить только последние 7 дней логов
sudo journalctl --vacuum-time=7d

# Или ограничить по размеру (сохранить только 100МБ)
sudo journalctl --vacuum-size=100M
```

### Автоматизированное обслуживание

Создайте простой скрипт обслуживания:

```bash
#!/bin/bash
# Сохранить как ~/maintenance.sh и сделать исполняемым

echo "🧹 Запуск обслуживания системы..."

# Обновить систему
sudo dnf upgrade --refresh

# Очистить кэш
sudo dnf clean all

# Очистить старые записи журнала
sudo journalctl --vacuum-time=7d

# Запустить TRIM на SSD
sudo fstrim -v /

echo "✅ Обслуживание завершено!"
```

-----

## 🖥️ Рекомендации по окружению рабочего стола

### Легкие альтернативы

Для максимальной производительности рассмотрите эти легкие окружения рабочего стола (при установке с Minimal ISO):

- **Sway** - Композитор тайлинга на основе Wayland (у меня 700МБ в простое)
- **i3** - Тайлинговый оконный менеджер X11 (600МБ в простое)
- **Hyprland** - Современный композитор Wayland с анимациями (900МБ в простое)
- **XFCE** - Легкий традиционный рабочий стол
- **LXQt** - Легкий рабочий стол на основе Qt

**KDE Plasma Edition (НОВОЕ в Fedora 42):**
KDE Plasma теперь официальная редакция Fedora наряду с Workstation (GNOME). 
Это означает лучшую интеграцию, поддержку и оптимизацию из коробки.

### Специфические улучшения Fedora 42

**Что нового для производительности:**
- Обновленные драйверы Mesa для лучшей графики AMD/Intel
- Улучшенная производительность композитора Wayland
- Лучшие настройки управления питанием по умолчанию
- Улучшенная поддержка контейнеров

**Игровые улучшения Fedora 42:**
- Лучшая совместимость с режимом Steam Deck
- Улучшенная интеграция Proton
- Подготовка улучшенной поддержки HDR

### Управление питанием ноутбука

**Установить инструменты оптимизации питания:**
```bash
sudo dnf install powertop tlp tlp-rdw

sudo systemctl enable --now tlp

# Настроить TLP для игрового/производительного режима

sudo nano /etc/tlp.conf

# Установить: TLP_DEFAULT_MODE=performance (при подключении к сети)
```

### Продвинутое управление памятью

**Настроить поведение подкачки:**
```bash
# Уменьшить swappiness для лучшей игровой производительности
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Улучшить выделение памяти для игр
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf
```

**Раздел контейнерных/Flatpak игр**

### Оптимизация контейнерных игр

**Оптимизация Steam Flatpak:**
```bash
# Установить Steam как Flatpak для лучшей изоляции
flatpak install com.valvesoftware.Steam

# Предоставить необходимые разрешения для игр
flatpak override --user --filesystem=~/.local/share/Steam com.valvesoftware.Steam
```

### Оптимизации GNOME

Если остаетесь с GNOME:

```bash
# Установить инструменты GNOME
sudo dnf install gnome-tweaks gnome-extensions-app

# Отключить анимации для лучшей производительности
gsettings set org.gnome.desktop.interface enable-animations false

# Уменьшить использование ресурсов
gsettings set org.gnome.shell.overrides workspaces-only-on-primary false
```

-----

## 🖥️ Оптимизация графических драйверов

<details>
<summary>🔴 Оптимизация графики AMD</summary>

### Установка драйвера AMD GPU

```bash
# Драйверы Mesa включены по умолчанию, убедитесь в последней версии
sudo dnf install mesa-vulkan-drivers mesa-vdpau-drivers mesa-va-drivers

# Установить ROCm для вычислений (опционально)
sudo dnf install rocm-opencl rocm-smi
```

### Настройки производительности AMD

```bash
# Создать конфигурацию оптимизации AMD GPU
cat << 'EOF' | sudo tee /etc/environment.d/99-amd-gaming.conf
# Включить потоки GPU
mesa_glthread=true

# Оптимизации RadeonSI
RADV_PERFTEST=gpl,nggc,sam,rt
AMD_VULKAN_ICD=RADV

# Ускорение видео
VDPAU_DRIVER=radeonsi
LIBVA_DRIVER_NAME=radeonsi

# Включить изменяемый BAR
AMD_GPU_ALLOW_RESIZE_BAR=1
EOF
```

### Управление питанием AMD GPU

```bash
# Установить режим производительности (в терминале):
echo "performance" | sudo tee /sys/class/drm/card*/device/power_dpm_state
echo "high" | sudo tee /sys/class/drm/card*/device/power_profile

# Создать постоянную службу
cat << 'EOF' | sudo tee /etc/systemd/system/amd-gpu-performance.service
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
EOF

sudo systemctl enable --now amd-gpu-performance.service
```

### Разгон памяти AMD (продвинутый, не пытайтесь делать это без опыта)

```bash
# Проверить текущую частоту памяти
cat /sys/class/drm/card*/device/pp_dpm_mclk

# Разогнать VRAM (пример, осторожно настраивайте значения)
echo "manual" | sudo tee /sys/class/drm/card*/device/power_dpm_force_performance_level
echo "3" | sudo tee /sys/class/drm/card*/device/pp_dpm_mclk
```

</details>

<details>
<summary>🟢 Оптимизация графики NVIDIA</summary>

## 🎮 Оптимизация графики NVIDIA для Fedora 42 (Wayland)

> **Комплексное руководство по оптимизации для GPU NVIDIA на Fedora 42 с дисплейным сервером Wayland**

### 📋 Системные требования NVIDIA

**Поддерживаемые GPU:**

- GTX 700/900/1000 серий и новее (Maxwell, Pascal, Turing, Ampere, Ada Lovelace, Blackwell)
- RTX 20/30/40/50 серий с полной поддержкой функций
- Карты Quadro и Tesla (профессиональные рабочие нагрузки)

**Совместимость драйверов:**

- **Рекомендуется:** Драйверы NVIDIA 575+ для оптимальной поддержки Wayland
- **Минимум:** NVIDIA 570+ для стабильной функциональности Wayland
- **Примечание:** Стек драйверов NVIDIA демонстрирует намного лучшую поддержку Wayland с последними драйверами

-----

### 🔧 Установка драйвера NVIDIA (Fedora 42 Wayland)

#### Метод 1: RPM Fusion (настоятельно рекомендуется)

RPM Fusion остается самым надежным методом для драйверов NVIDIA на Fedora 42. Этот подход обеспечивает правильную интеграцию с дисплейным сервером Wayland и системными обновлениями.

```bash
# Включить репозитории RPM Fusion (если еще не включены)
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Обновить системные пакеты
sudo dnf update

# Установить драйверы NVIDIA с поддержкой Wayland
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# Установить 32-битные библиотеки совместимости (необходимо для Steam, Wine, игр)
sudo dnf install xorg-x11-drv-nvidia-libs.i686

# Установить утилиту настроек NVIDIA
sudo dnf install nvidia-settings

# Установить дополнительные инструменты для мониторинга
sudo dnf install nvidia-ml-py3
```

#### Проверка после установки

Понимание того, что говорит каждая команда, помогает убедиться, что ваша система правильно настроена для оптимальной производительности.

```bash
# Проверить установку драйвера и версию
nvidia-smi
# Это должно показать ваш GPU, версию драйвера (575+) и текущее использование

# Подтвердить доступность поддержки CUDA
nvidia-smi -q | grep "CUDA Version"
# Необходимо для ИИ нагрузок и некоторых игр, использующих вычисления GPU

# Проверить, что Wayland использует NVIDIA GPU
echo $XDG_SESSION_TYPE
# Должно выводить "wayland" на Fedora 42

# Проверить, что бэкенд GBM работает
nvidia-smi --query-gpu=name,driver_version --format=csv
# Подтверждает правильную загрузку драйвера
```

#### Включение Wayland для NVIDIA (важный шаг)

Wayland требует специальной конфигурации для правильной работы с драйверами NVIDIA. Этот шаг обеспечивает возможность вашего окружения рабочего стола использовать аппаратное ускорение.

```bash
# Включить DRM kernel mode setting (требуется для Wayland)
echo 'options nvidia_drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-drm-modeset.conf

# Включить раннюю загрузку модулей NVIDIA
echo -e 'nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm' | sudo tee /etc/modules-load.d/nvidia.conf

# Пересобрать initramfs для включения изменений
sudo dracut --force

# Перезагрузить для применения изменений модулей ядра
sudo reboot
```

-----

### ⚡ Оптимизации производительности NVIDIA Wayland

#### 1. Переменные окружения для Wayland

Эти переменные окружения оптимизируют поведение GPU NVIDIA специально для композиторов Wayland. В отличие от X11, Wayland автоматически обрабатывает многие оптимизации, но эти переменные тонко настраивают производительность.

Добавить в `/etc/environment`:

```bash
# Основные оптимизации NVIDIA Wayland
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia

# ВАЖНО: __GL_THREADED_OPTIMIZATIONS может вызвать чёрный экран на некоторых RTX-картах
# Рекомендуется выставлять эту опцию только для отдельных игр, а не глобально.
# Если вы всё же добавили её в /etc/environment и получили чёрный экран — выполните следующие шаги.

# 1. Переключитесь в TTY:
#    - Нажмите Ctrl+Alt+F3 (можно F2–F6) — появится экран для входа.
#    - Введите свой логин и пароль.

# 2. Откройте файл /etc/environment для редактирования и удалите строку:
sudo nano /etc/environment

#    Найдите строку:
#        __GL_THREADED_OPTIMIZATIONS=1
#    Удалите её, затем сохраните (Ctrl+O, Enter) и выйдите (Ctrl+X).

# 3. Перезагрузите систему:
sudo reboot

# Кэширование компиляции шейдеров (сокращает время загрузки)
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_PATH=/tmp/nvidia-shader-cache
__GL_SHADER_DISK_CACHE_SIZE=1073741824

# Отключить VSync для игр (снижает задержку ввода)
__GL_SYNC_TO_VBLANK=0

# Включить неофициальные расширения протокола (совместимость)
__GL_ALLOW_UNOFFICIAL_PROTOCOL=1

# Специфические оптимизации Wayland
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1

# Игровые оптимизации
NVIDIA_DRIVER_CAPABILITIES=all
PROTON_ENABLE_NVAPI=1
```

#### 2. Параметры модулей ядра

Современные драйверы NVIDIA получают преимущества от специфических параметров ядра, которые улучшают совместимость с Wayland и производительность.

Создать `/etc/modprobe.d/nvidia-power-management.conf`:

```bash
# Включить современные функции управления питанием
options nvidia NVreg_DynamicPowerManagement=0x02

# Включить Page Attribute Table (улучшает производительность памяти)
options nvidia NVreg_UsePageAttributeTable=1

# Включить поддержку ResizableBAR (серии RTX 30/40/50)
options nvidia NVreg_EnableResizableBar=1

# Сохранить видеопамять во время приостановки
options nvidia NVreg_PreserveVideoMemoryAllocations=1

# Включить операции потоковой памяти (требуется для некоторых нагрузок)
options nvidia NVreg_EnableStreamMemOPs=1
```

#### 3. Специфические настройки GNOME Wayland

GNOME на Wayland требует особого внимания для достижения оптимальной производительности NVIDIA. Эти настройки решают общие проблемы с композитором GNOME.

```bash
# Включить ускорение NVIDIA для сессии GNOME Wayland
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"

# Настроить GNOME для игровой производительности
gsettings set org.gnome.shell.extensions.dash-to-dock click-action 'cycle-windows'
gsettings set org.gnome.desktop.interface enable-animations false

# Установить коэффициент масштабирования для дисплеев высокой плотности (настройте по необходимости)
gsettings set org.gnome.desktop.interface scaling-factor 1
```

#### 4. Конфигурация KDE Plasma Wayland

KDE Plasma имеет отличную поддержку Wayland и особенно хорошо работает с драйверами NVIDIA при правильной настройке.

```bash
# Включить поддержку переменной частоты обновления
kwriteconfig5 --file kwinrc --group Compositing --key VariableRefreshRate true

# Оптимизировать настройки композитора для игр
kwriteconfig5 --file kwinrc --group Compositing --key LatencyPolicy Low
kwriteconfig5 --file kwinrc --group Compositing --key RenderTimeEstimator 1

# Перезапустить KWin для применения изменений
qdbus org.kde.KWin /KWin reconfigure
```

-----

### 🏎️ Специфические игровые оптимизации NVIDIA

#### 1. Параметры запуска Steam для Wayland

Игры Steam на Wayland требуют специфических параметров запуска, чтобы обеспечить правильное использование GPU NVIDIA и достичь оптимальной производительности.

**Для нативных Linux игр:**

```bash
# Базовая оптимизация с GameMode
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 %command%

# Улучшенная производительность для соревновательных игр
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 %command%
```

**Для игр Proton/Wine:**

```bash
# Стандартная оптимизация Proton
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 PROTON_ENABLE_NVAPI=1 %command%

# Продвинутая оптимизация с асинхронной компиляцией DXVK
gamemoderun __GL_THREADED_OPTIMIZATIONS=1 DXVK_ASYNC=1 PROTON_ENABLE_NVAPI=1 %command%

# Для игр, требующих максимальной производительности
gamemoderun __GL_SYNC_TO_VBLANK=0 __GL_THREADED_OPTIMIZATIONS=1 DXVK_ASYNC=1 PROTON_ENABLE_NVAPI=1 %command%
```

#### 2. Игровая оптимизация Lutris

Lutris обеспечивает отличную интеграцию с драйверами NVIDIA на Wayland. Настройте эти параметры для оптимальной игровой производительности.

```bash
# Установить Lutris с поддержкой NVIDIA
sudo dnf install lutris wine

# Настроить переменные окружения Lutris (в настройках Lutris)
# Добавить в System Options → Environment variables:
__GL_THREADED_OPTIMIZATIONS=1
__GL_SHADER_DISK_CACHE=1
PROTON_ENABLE_NVAPI=1
DXVK_HUD=fps,memory,gpuload
```

#### 3. Интеграция GameMode

GameMode автоматически оптимизирует производительность системы во время игровых сессий, обеспечивая лучшее распределение ресурсов и сниженную задержку.

```bash
# Установить GameMode
sudo dnf install gamemode

# Настроить GameMode для оптимизации NVIDIA
sudo tee /etc/gamemode.ini << 'EOF'
[general]
renice=10
ioprio=1

[gpu]
apply_gpu_optimisations=accept-responsibility
gpu_device=0
amd_performance_level=high

[custom]
start=nvidia-smi -pm 1 && nvidia-smi -pl 300
end=nvidia-smi -pm 0 && nvidia-smi -ac -r
EOF
```

-----

### 🔥 Продвинутые настройки NVIDIA Wayland

#### 1. Поддержка переменной частоты обновления (VRR)

Современные драйверы NVIDIA поддерживают переменную частоту обновления на Wayland, обеспечивая более плавный игровой опыт с совместимыми мониторами.

```bash
# Включить VRR в GNOME (требует GNOME 45+)
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate']"

# Для KDE Plasma включить в системных настройках или через команду:
kwriteconfig5 --file kwinrc --group Compositing --key VariableRefreshRate true

# Проверить, что VRR работает
sudo dnf install drm_info
drm_info | grep -i vrr
```

#### 2. Поддержка HDR (экспериментальная)

Поддержка расширенного динамического диапазона постепенно улучшается на Wayland с драйверами NVIDIA. Эти настройки включают экспериментальную функциональность HDR.

```bash
# Включить поддержку HDR (требует совместимый дисплей и свежие драйверы)
echo 'options nvidia NVreg_EnableHDR=1' | sudo tee /etc/modprobe.d/nvidia-hdr.conf

# Поддержка HDR в GNOME (экспериментальная, GNOME 46+)
gsettings set org.gnome.mutter experimental-features "['variable-refresh-rate','hdr']"
```

#### 3. Мониторинг и настройка производительности

Эффективный мониторинг производительности помогает выявить узкие места и проверить, что оптимизации работают правильно.

```bash
# Установить инструменты мониторинга
sudo dnf install nvtop mangohud goverlay

# Создать скрипт мониторинга для игровых сессий
sudo tee /usr/local/bin/nvidia-gaming-monitor.sh << 'EOF'
#!/bin/bash
echo "=== Монитор игровой производительности NVIDIA ==="
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo "Драйвер: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
echo "=== Статистика в реальном времени ==="
nvidia-smi dmon -s pucvmet
EOF

sudo chmod +x /usr/local/bin/nvidia-gaming-monitor.sh
```

-----

### 🛡️ Управление питанием и тепловая оптимизация

#### 1. Продвинутое управление питанием

Правильное управление питанием обеспечивает стабильную производительность, предотвращая ненужное потребление энергии в периоды простоя.

```bash
# Настроить продвинутое управление питанием
echo 'options nvidia NVreg_DynamicPowerManagement=0x02' | sudo tee -a /etc/modprobe.d/nvidia-power.conf

# Включить runtime управление питанием для ноутбуков
sudo tee /etc/udev/rules.d/80-nvidia-pm.rules << 'EOF'
# Включить runtime PM для устройств NVIDIA VGA/3D контроллера
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
EOF
```

#### 2. Тепловое управление

Эффективное тепловое управление предотвращает троттлинг и поддерживает оптимальную производительность во время продолжительных игровых сессий.

```bash
# Установить инструменты теплового мониторинга
sudo dnf install lm_sensors

# Настроить обнаружение датчиков
sudo sensors-detect --auto

# Создать скрипт теплового мониторинга
sudo tee /usr/local/bin/nvidia-thermal.sh << 'EOF'
#!/bin/bash
TEMP=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits)
POWER=$(nvidia-smi --query-gpu=power.draw --format=csv,noheader,nounits)

echo "Температура GPU: ${TEMP}°C"
echo "Потребляемая мощность: ${POWER}Вт"

# Предупредить, если температура высокая
if [ $TEMP -gt 83 ]; then
    echo "ВНИМАНИЕ: Высокая температура GPU!"
    notify-send "Предупреждение о температуре GPU" "GPU работает при ${TEMP}°C"
fi
EOF

sudo chmod +x /usr/local/bin/nvidia-thermal.sh
```

-----

### 🐛 Устранение неполадок NVIDIA Wayland

#### Общие проблемы и современные решения

Понимание того, как диагностировать и решать проблемы, обеспечивает оптимальную производительность и стабильность системы.

**1. Сессия Wayland не запускается с NVIDIA:**

Это наиболее распространенная проблема при переходе с X11 на Wayland с драйверами NVIDIA.

```bash
# Проверить правильность параметров модуля ядра
cat /etc/modprobe.d/nvidia-drm-modeset.conf
# Должно содержать: options nvidia_drm modeset=1 fbdev=1

# Проверить, включен ли DRM modeset
cat /sys/module/nvidia_drm/parameters/modeset
# Должно выводить: Y

# Пересобрать initramfs и перезагрузиться при необходимости
sudo dracut --force
sudo reboot

# Проверить сессию Wayland после перезагрузки
echo $XDG_SESSION_TYPE
# Должно выводить: wayland
```

**2. Плохая игровая производительность несмотря на хорошее оборудование:**

Проблемы производительности часто связаны с неправильным использованием GPU или настройками управления питанием.

```bash
# Проверить использование GPU
nvidia-smi dmon -s pucvmet -c 10

# Проверить ограничения по питанию
nvidia-smi --query-gpu=power.limit,power.draw --format=csv
# Потребляемая мощность должна приближаться к лимиту мощности во время игр

# Мониторить частоты GPU во время игр
watch -n 1 'nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv'
# Частоты должны достигать максимальных значений во время игр
```

**3. Разрывы экрана или заикания:**

Современные композиторы Wayland лучше справляются с разрывами, чем X11, но может потребоваться некоторая настройка.

```bash
# Для GNOME убедиться, что VRR включен
gsettings get org.gnome.mutter experimental-features
# Должно включать 'variable-refresh-rate'

# Для игр отключить VSync в игре и использовать VSync композитора
# Добавить в параметры запуска Steam:
__GL_SYNC_TO_VBLANK=0 %command%
```

**4. Высокое потребление энергии в простое:**

Предотвращение ненужного потребления энергии во время простоя улучшает время работы батареи и снижает тепловыделение.

```bash
# Включить runtime управление питанием
echo 'auto' | sudo tee /sys/bus/pci/devices/0000:*/power/control

# Проверить, что управление питанием работает
cat /sys/bus/pci/devices/0000:*/power/runtime_status
# Должно показывать 'suspended' для простаивающего GPU

# Мониторить потребление энергии в простое
nvidia-smi --query-gpu=power.draw --format=csv --loop=1
```

-----

### 🔧 Инструменты разработчика и ИИ NVIDIA

#### Среда разработки CUDA

Правильная настройка CUDA обеспечивает совместимость с фреймворками ИИ и инструментами разработки.

```bash
# Установить CUDA toolkit
sudo dnf install cuda-toolkit cuda-devel cuda-runtime

# Настроить переменные окружения
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export CUDA_HOME=/usr/local/cuda' >> ~/.bashrc

# Перезагрузить окружение
source ~/.bashrc

# Проверить установку CUDA
nvcc --version
nvidia-smi --query-gpu=compute_cap --format=csv
```

#### Поддержка контейнеров для рабочих нагрузок ИИ/МО

Поддержка контейнеров обеспечивает легкое развертывание приложений искусственного интеллекта и машинного обучения.

```bash
# Установить NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

sudo dnf install nvidia-container-toolkit

# Настроить Docker/Podman
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Протестировать поддержку контейнеров
sudo docker run --rm --gpus all nvidia/cuda:12.3-runtime-ubuntu20.04 nvidia-smi
```

-----

### 📊 Мониторинг производительности и бенчмаркинг

#### Комплексная настройка мониторинга

Эффективный мониторинг помогает оптимизировать производительность и выявлять потенциальные проблемы до того, как они повлияют на игровую или рабочую производительность.

```bash
# Установить комплексный набор для мониторинга
sudo dnf install nvtop btop mangohud goverlay

# Создать скрипт мониторинга производительности
sudo tee /usr/local/bin/nvidia-perf-monitor.sh << 'EOF'
#!/bin/bash
clear
echo "=== Монитор производительности NVIDIA ==="
echo "Система: $(hostnamectl --static) | $(date)"
echo "Драйвер: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)"
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)"
echo ""

# Использование GPU и памяти
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw,clocks.gr,clocks.mem --format=csv

echo ""
echo "=== Активные процессы GPU ==="
nvidia-smi pmon -c 1

echo ""
echo "Нажмите Ctrl+C для выхода из непрерывного мониторинга..."
watch -n 2 nvidia-smi
EOF

sudo chmod +x /usr/local/bin/nvidia-perf-monitor.sh
```

#### Игровой оверлей производительности

MangoHud предоставляет метрики производительности в реальном времени во время игровых сессий.

```bash
# Настроить MangoHud для оптимального отображения
mkdir -p ~/.config/MangoHud

cat > ~/.config/MangoHud/MangoHud.conf << 'EOF'
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

# Ограничить логирование для предотвращения влияния на производительность
log_duration=60
EOF
```

-----

### 📚 Дополнительные ресурсы

**Официальная документация NVIDIA:**
- [Руководство по установке драйвера NVIDIA для Linux](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html)
- [Руководство по установке CUDA для Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

**Ресурсы специфичные для Fedora:**
- [Руководство RPM Fusion по NVIDIA](https://rpmfusion.org/Howto/NVIDIA)
- [Заметки о выпуске Fedora 42](https://fedoraproject.org/wiki/Releases/42/ChangeSet)

**Wayland и игры:**
- [Игры на Linux с NVIDIA](https://www.gamingonlinux.com/)
- [Документация MangoHud](https://github.com/flightlessmango/MangoHud)

-----

</details>

-----

## 🔍 Мониторинг и проверка

### Инструменты мониторинга производительности

```bash
# Установить полезные инструменты мониторинга
sudo dnf install htop iotop powertop fastfetch

# Для подробной информации о системе
sudo dnf install hardinfo
```

### Инструменты бенчмаркинга

```bash
# Игровые бенчмарки
sudo dnf install glmark2 unigine-superposition

# Системные бенчмарки
sudo dnf install sysbench stress-ng
```

-----

## 🚨 Устранение неполадок

### Распространенные проблемы

**1. Проблемы загрузки после параметров ядра:**

- Загрузитесь с предыдущего ядра из меню GRUB
- Удалите проблемные параметры из `/etc/default/grub`
- Пересгенерируйте конфигурацию GRUB

**2. Проблемы с графикой:**

- Проверьте установку драйвера: `lspci -k | grep -A 2 -E "(VGA|3D)"`
- Проверьте правильную загрузку драйвера: `lsmod | grep -E "(amdgpu|nvidia|i915)"`

**3. Регрессия производительности:**

- Мониторьте системные ресурсы: `htop`, `iotop`
- Проверьте тепловое троттлинг: `watch sensors`
- Проверьте статус служб: `systemctl list-units --failed`

### Команды восстановления

```bash
# Сбросить GRUB к настройкам по умолчанию
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Сбросить контекст SELinux (при повторном включении SELinux)
sudo restorecon -R /

# Проверить целостность системы
sudo dnf check
sudo rpm -Va
```

-----

## 📊 Ожидаемый прирост производительности

На основе тестирования пользователи могут ожидать:

- **Время загрузки:** Улучшение на 15-30%
- **Игровая производительность:** Увеличение FPS на 5-15%
- **Отзывчивость системы:** Значительно сниженная задержка ввода
- **Использование памяти:** Снижение использования ОЗУ в простое на 10-20%
- **Производительность хранилища:** Улучшенная производительность SSD с trim

-----

## 🤝 Участие в проекте

Нашли улучшения или есть предложения? Не стесняйтесь:
- Открыть issue на GitHub
- Отправить pull request
- Поделиться результатами ваших оптимизаций

-----

## 📚 Дополнительные ресурсы

- [Документация Fedora](https://docs.fedoraproject.org/)
- [RPM Fusion](https://rpmfusion.org/)
- [Ядро CachyOS](https://github.com/CachyOS/linux-cachyos)
- [Ananicy-cpp](https://gitlab.com/ananicy-cpp/ananicy-cpp)
- [Gaming on Linux](https://www.gamingonlinux.com/)

-----

## ⚖️ Отказ от ответственности

Данное руководство изменяет системные настройки, которые могут повлиять на стабильность и безопасность. Всегда:
- Создавайте резервные копии системы перед применением изменений
- Тестируйте изменения сначала на некритических системах
- Понимайте последствия каждого изменения
- Держите носители восстановления под рукой

**Результаты производительности могут варьироваться** в зависимости от конфигурации оборудования и конкретных случаев использования.

-----

## 📝 История изменений

- **v1.0** - Первоначальное комплексное руководство
- **v1.1** - Добавлен раздел устранения неполадок и русский перевод
- **v1.2** - Улучшено инструментами мониторинга и скриптами обслуживания
- **v1.3** - Добавлены драйверы NVIDIA и руководство по производительности и многое другое
- **v1.4** - Добавлен раздел настроек GPU AMD, исправлен некоторый текст и обновлен полный русский перевод
- **v1.5** - Добавлен раздел "Быстрая навигация" для удобства использования.

-----

*Последнее обновление: Сентябрь 2025*
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
- Test changes on non-critical systems first
- Understand the implications of each modification
- Keep recovery media accessible

**Performance results may vary** based on hardware configuration and specific use cases.

-----

## 📝 Changelog

- **v1.0** - Initial comprehensive guide
- **v1.1** - Added troubleshooting section and Russian translation
- **v1.2** - Enhanced with monitoring tools and maintenance scripts
- **v1.3** - Added NVIDIA drivers and performance guide and more
- **v1.4** - Added AMD gpu tweaks section, corrected some text and updated full Russian translation
- **v1.5** - Added a Quick Navigation section for better usability
- **v1.6** - Changed from UKSMD to KSMD, more deeper cachyos kernel settings, etc
- **v1.7** - Updated with more nvidia flags, corrections, preparing for Fedora 43 release

-----

*Last updated: December 2025*
