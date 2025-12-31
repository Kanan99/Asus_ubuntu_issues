# Complete Guide to Linux Hybrid Graphics: Concepts, Terms, and Technologies

## Table of Contents
1. [GPU Architecture Fundamentals](#gpu-architecture-fundamentals)
2. [Linux Graphics Stack](#linux-graphics-stack)
3. [NVIDIA Driver Architecture](#nvidia-driver-architecture)
4. [Linux Kernel Concepts](#linux-kernel-concepts)
5. [Hybrid Graphics Technologies](#hybrid-graphics-technologies)
6. [Power Management](#power-management)
7. [Diagnostic Tools and Commands](#diagnostic-tools-and-commands)
8. [What We Did - Step by Step Explanation](#what-we-did-step-by-step-explanation)

---

## GPU Architecture Fundamentals

### What is a GPU?
**GPU (Graphics Processing Unit)** is a specialized processor designed for parallel computation. Unlike CPUs which have few powerful cores, GPUs have thousands of smaller cores optimized for processing many operations simultaneously.

### Types of GPUs in Laptops

#### iGPU (Integrated GPU)
- **Definition:** GPU integrated into the CPU die or chip package
- **Location:** Part of the CPU, shares system RAM
- **Power:** Low power consumption (5-15W)
- **Performance:** Adequate for desktop UI, video playback, light gaming
- **Examples:** AMD Radeon 890M (in Ryzen 9 8945HS), Intel Iris Xe
- **Advantages:** 
  - Always available
  - Excellent battery life
  - No additional power overhead
- **Disadvantages:**
  - Limited performance
  - Shares system RAM (no dedicated VRAM)

#### dGPU (Discrete GPU)
- **Definition:** Separate, dedicated graphics card
- **Location:** Separate chip on motherboard with dedicated VRAM
- **Power:** High power consumption (60-150W+)
- **Performance:** High performance for gaming, ML, rendering
- **Examples:** NVIDIA RTX 5070 Ti, AMD Radeon RX 7800M
- **Advantages:**
  - High performance
  - Dedicated VRAM (12GB in your case)
  - Better for compute workloads (CUDA, ML)
- **Disadvantages:**
  - Higher power consumption
  - Generates more heat
  - Reduces battery life when active

### Hybrid Graphics
**Definition:** System with both iGPU and dGPU that can switch between them or use them simultaneously.

**Purpose:** Balance performance and battery life
- Use iGPU for desktop/light tasks → Save battery
- Use dGPU for demanding tasks → Get performance

---

## Display Routing & MUX Switch

### Physical Display Connections

In your ASUS G14 2025, displays are physically wired to specific GPUs:

```
Internal Display (eDP-1-1) ← Wired to → AMD Radeon 890M (iGPU)
HDMI Port                  ← Wired to → NVIDIA RTX 5070 Ti (dGPU)
USB-C/Thunderbolt         ← Wired to → NVIDIA RTX 5070 Ti (dGPU)
```

### MUX Switch
**MUX (Multiplexer)** is a hardware switch that can change which GPU a display is connected to.

**Without MUX (Traditional Hybrid):**
```
Rendering GPU → Frame Copy → Display GPU → Monitor
(Slower, overhead from copying)
```

**With MUX (Switchable Graphics):**
```
Rendering GPU → Direct → Monitor
(Faster, no overhead)
```

Your G14 has a MUX switch, allowing it to:
1. **Integrated mode:** All displays → AMD
2. **Hybrid mode:** Internal → AMD, External → NVIDIA  
3. **Discrete mode:** All displays → NVIDIA (MUX switches internal display)

### Cross-GPU Routing Problem

**What Caused Your Freezes:**
```
1. Desktop renders with NVIDIA (wrong GPU)
2. Frame must be copied to AMD (cross-GPU routing)
3. AMD sends frame to internal display
4. Routing is unstable → System freezes
```

**The Fix:**
```
1. Desktop renders with AMD (correct GPU)
2. Frame goes directly to display (no routing)
3. Stable and efficient
```

---

## Linux Graphics Stack

### Display Server Protocols

#### X11 (X Window System)
- **Age:** 1984 - legacy system
- **Architecture:** Client-server model
- **Features:**
  - Network transparency (can display remotely)
  - Mature, stable, widely supported
  - Complex, monolithic design
- **Security:** Limited isolation between windows
- **Performance:** Additional overhead, some tearing

#### Wayland
- **Age:** 2008 - modern replacement for X11
- **Architecture:** Compositor-centric
- **Features:**
  - Better security (window isolation)
  - Smoother graphics (no tearing)
  - Simpler protocol
  - Better touch/gesture support
- **Hybrid Graphics:** Native support, handles GPU switching elegantly
- **Current Status:** Default on Ubuntu 24.04

**Your System Uses:** Wayland (check with `echo $XDG_SESSION_TYPE`)

### Graphics Rendering APIs

#### OpenGL (Open Graphics Library)
- **Purpose:** Cross-platform API for 2D/3D graphics
- **Use Cases:** Desktop UI, games, CAD, visualization
- **How it Works:** Application → OpenGL API → GPU Driver → GPU
- **Test Tool:** `glxinfo`, `glxgears`

#### Vulkan
- **Purpose:** Modern, low-level graphics API (successor to OpenGL)
- **Use Cases:** High-performance games, scientific visualization
- **Advantages:** Lower overhead, better multi-threading

#### CUDA (Compute Unified Device Architecture)
- **Purpose:** NVIDIA's parallel computing platform for general computation
- **Use Cases:** ML/AI, scientific computing, video processing
- **Important:** CUDA bypasses OpenGL/Vulkan entirely
  ```
  PyTorch/TensorFlow → CUDA Runtime → NVIDIA Driver → GPU
  (No display system involvement)
  ```

### Linux Graphics Subsystems

#### DRM (Direct Rendering Manager)
- **Location:** Linux kernel
- **Purpose:** Interface between userspace and GPU hardware
- **Functions:**
  - Memory management
  - Command submission to GPU
  - Mode setting (resolution, refresh rate)

#### KMS (Kernel Mode Setting)
- **Part of:** DRM subsystem
- **Purpose:** Set display modes in kernel space (not userspace)
- **Benefits:** 
  - Faster boot (graphics mode set earlier)
  - Smoother VT switching
  - Better error handling

**Kernel Parameter:** `nvidia-drm.modeset=1` enables KMS for NVIDIA

---

## NVIDIA Driver Architecture

### Driver Types

#### Proprietary Driver
- **Package:** `nvidia-driver-XXX`
- **Source:** Closed-source, binary blob from NVIDIA
- **Advantages:**
  - Full features (CUDA, RTX, NVENC/NVDEC)
  - Best performance
  - Official support
- **Disadvantages:**
  - Not in mainline kernel
  - Updates lag behind kernel
  - Some compatibility issues

#### Open Kernel Modules
- **Package:** `nvidia-driver-XXX-open`
- **Source:** NVIDIA's open-source kernel modules (released 2022)
- **Status:** For RTX 20+ series, becoming default
- **Note:** Core GPU firmware still proprietary
- **Your System:** Using `nvidia-driver-580-open`

#### Nouveau (Open Source Driver)
- **Source:** Reverse-engineered, open-source
- **Status:** Basic support, no CUDA, poor performance
- **Use:** Fallback only
- **Your System:** Blacklisted (disabled)

### NVIDIA Driver Components

```
User Space:
├── libcuda.so              (CUDA Runtime)
├── libGL.so.1              (OpenGL implementation)
├── nvidia-smi              (Management tool)
└── /usr/bin/nvidia-*       (Utilities)

Kernel Space:
├── nvidia.ko               (Core driver module)
├── nvidia-modeset.ko       (KMS support)
├── nvidia-drm.ko           (DRM/KMS integration)
└── nvidia-uvm.ko           (Unified Memory)
```

### NVIDIA Module Loading

**Normal Load Order:**
1. `nvidia` - Core driver
2. `nvidia-modeset` - Mode setting support  
3. `nvidia-drm` - Display interface
4. `nvidia-uvm` - Unified memory (for CUDA)

**Check Loaded Modules:**
```bash
lsmod | grep nvidia
```

Output shows:
- Module name
- Size
- Use count (how many processes using it)
- Dependencies

---

## Linux Kernel Concepts

### What is the Linux Kernel?

**Kernel** is the core of the operating system that:
- Manages hardware resources
- Provides system calls for programs
- Handles memory, processes, devices
- Runs in privileged mode (ring 0)

**Your Kernel:** `6.14.0-37-generic`
- **6.14** - Major version
- **0** - Minor version  
- **37** - Ubuntu patch level
- **generic** - Standard desktop/server kernel

### Kernel Modules

**Definition:** Pieces of code that can be loaded/unloaded into the kernel at runtime without rebooting.

**Purpose:** 
- Add hardware support (drivers)
- Add filesystems
- Add networking protocols
- Keep kernel size manageable

**Example:** NVIDIA driver is a set of kernel modules

#### Module Operations

**List Loaded Modules:**
```bash
lsmod
```

**Load Module:**
```bash
sudo modprobe nvidia
```

**Remove Module:**
```bash
sudo modprobe -r nvidia
```

**Module Information:**
```bash
modinfo nvidia
```

### Modprobe & Module Configuration

#### modprobe
**Purpose:** Intelligent module loader that handles dependencies

**Example:** Loading `nvidia-drm` automatically loads `nvidia` and `nvidia-modeset` first

**Configuration Directories:**
- `/etc/modprobe.d/` - User/package configurations (priority)
- `/lib/modprobe.d/` - System defaults

#### Configuration Types

**Blacklist:** Prevent module from loading
```bash
blacklist nouveau
```

**Alias:** Alternative name for module
```bash
alias nvidia off  # Maps "nvidia" to "off" (disabled)
```

**Options:** Pass parameters to module
```bash
options nvidia-drm modeset=1
```

**Install/Remove Directives:** Custom loading behavior
```bash
install nvidia /sbin/modprobe --ignore-install nvidia
```

#### The "alias nvidia off" Problem

**What Happened:**
- File: `/lib/modprobe.d/blacklist-nvidia.conf`
- Content: `alias nvidia off`
- Effect: When system tries `modprobe nvidia`, it looks for module named "off"
- Result: Error "could not find module by name='off'"

**Why It Existed:**
- Created by `nvidia-prime` package
- Intended to keep NVIDIA disabled by default
- Outdated syntax causing conflicts

**The Fix:**
- Removed `alias nvidia off` lines
- Kept `blacklist nouveau` to prevent conflicts

### Kernel Parameters

**Definition:** Arguments passed to kernel at boot time to modify behavior

**Where Defined:** `/etc/default/grub`
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1"
```

**Your Parameters:**
- `nvidia-drm.modeset=1` - Enable KMS for NVIDIA
- `nvidia_drm.fbdev=1` - Enable framebuffer device
- `amdgpu.dcdebugmask=0x10` - AMD GPU debug settings
- `nouveau.modeset=0` - Disable nouveau KMS
- `ibt=off` - Disable Indirect Branch Tracking (compatibility)

**Apply Changes:**
```bash
sudo update-grub  # Regenerates /boot/grub/grub.cfg
sudo reboot       # New parameters take effect
```

**View Current Parameters:**
```bash
cat /proc/cmdline
```

---

## Hybrid Graphics Technologies

### NVIDIA PRIME

**PRIME (Platform Runtime Integrated Mesa Extensions)**

**Purpose:** Framework for hybrid graphics on Linux

**Components:**
- **PRIME Render Offload:** Render on one GPU, display on another
- **PRIME Synchronization:** Sync frames between GPUs

**Ubuntu's Implementation:**
- `nvidia-prime` package
- `prime-select` tool
- Modes: intel (iGPU only), nvidia (dGPU only), on-demand (hybrid)

**Problem with prime-select:**
- Doesn't understand ASUS MUX switches
- Can configure incorrect display routing
- Generic solution, not hardware-specific

### ASUS-Specific: supergfxctl & asusctl

#### asusctl
**What:** Comprehensive utility for ASUS ROG laptops on Linux

**Features:**
- Keyboard LED control
- Fan curve management
- Power profiles
- Battery charge limits
- **Graphics management (supergfxctl)**

#### supergfxctl
**What:** GPU switching daemon specifically for ASUS ROG laptops

**Why Better than prime-select:**
- Understands ASUS MUX switch hardware
- Proper display routing configuration
- Integrated with ASUS firmware
- Mode-specific optimizations

**Configuration File:** `/etc/supergfxd.conf`

**Available Modes:**

1. **Integrated**
   - Only AMD iGPU active
   - NVIDIA completely powered off
   - Best battery life (6-8 hours)
   - No CUDA available

2. **Hybrid**
   - Both GPUs available
   - AMD handles desktop/internal display
   - NVIDIA available for compute/external displays
   - Balanced (5-7 hours battery)
   - Full CUDA support

3. **AsusMuxDgpu**
   - MUX switch routes all displays to NVIDIA
   - Like dedicated GPU laptop
   - Best gaming performance
   - Worst battery (2-3 hours)

**Check Current Mode:**
```bash
supergfxctl -g
```

**Daemon Service:**
```bash
sudo systemctl status supergfxd
```

### PRIME Render Offload (Environment Variables)

**Purpose:** Tell applications to render with NVIDIA instead of AMD

**Variables:**
```bash
__NV_PRIME_RENDER_OFFLOAD=1        # Enable NVIDIA offload
__GLX_VENDOR_LIBRARY_NAME=nvidia   # Use NVIDIA OpenGL
```

**Usage:**
```bash
# Single command
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxgears

# Session-wide
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
```

**Important:** CUDA applications don't need these variables - they use NVIDIA automatically.

---

## Power Management

### PCIe Runtime Power Management

**Concept:** PCIe devices can enter low-power states when idle

**Power States:**
- **D0:** Fully operational (~60-80W for GPU)
- **D1/D2:** Intermediate states
- **D3hot:** Suspended, quick wake (~2-5W)
- **D3cold:** Powered off completely (~0.5-1W)

**Check GPU Power State:**
```bash
cat /sys/bus/pci/devices/0000:64:00.0/power/runtime_status
```

Outputs:
- `active` - GPU is running
- `suspended` - GPU is in low-power state

**Control Settings:**
```bash
cat /sys/bus/pci/devices/0000:64:00.0/power/control
```

Values:
- `auto` - Automatic power management (recommended)
- `on` - Keep GPU always powered

**Time Metrics:**
```bash
# Milliseconds spent suspended
cat /sys/bus/pci/devices/0000:64:00.0/power/runtime_suspended_time

# Milliseconds spent active  
cat /sys/bus/pci/devices/0000:64:00.0/power/runtime_active_time
```

### Why nvidia-smi is Slow

**In Hybrid Mode with auto PM:**
1. GPU is suspended (D3cold state)
2. You run `nvidia-smi`
3. System wakes GPU:
   - Power on GPU
   - Initialize PCIe link
   - Load GPU firmware
   - Driver communication established
4. Takes 1-3 seconds
5. `nvidia-smi` displays info

**Subsequent calls:** Instant (GPU still awake)

**For ML workloads:** GPU stays awake during training, no delays

---

## Diagnostic Tools and Commands

### GPU Information

#### lspci
**Purpose:** List PCI devices

```bash
lspci | grep -E 'VGA|3D'
```

Shows:
```
03:00.0 VGA compatible controller: AMD [Radeon 890M]
64:00.0 3D controller: NVIDIA [RTX 5070 Ti Laptop GPU]
```

**Detailed Info:**
```bash
lspci -vnn -s 64:00.0
```

#### nvidia-smi
**Purpose:** NVIDIA System Management Interface

**Basic:**
```bash
nvidia-smi
```

Shows:
- Driver version
- CUDA version
- GPU utilization
- Memory usage
- Temperature
- Power draw
- Running processes

**Continuous Monitoring:**
```bash
watch -n 1 nvidia-smi
```

**Query Specific Fields:**
```bash
nvidia-smi --query-gpu=power.draw,temperature.gpu --format=csv
```

#### glxinfo
**Purpose:** OpenGL/GLX information

**Renderer:**
```bash
glxinfo | grep "OpenGL renderer"
```

Shows which GPU is rendering:
- `AMD Radeon Graphics` - iGPU
- `NVIDIA GeForce RTX` - dGPU

**Full Details:**
```bash
glxinfo | less
```

### System Information

#### uname
**Purpose:** System information

```bash
uname -r  # Kernel version: 6.14.0-37-generic
uname -a  # Full system info
```

#### dmesg
**Purpose:** Kernel ring buffer messages

**Recent Messages:**
```bash
sudo dmesg | tail -50
```

**Filter for NVIDIA:**
```bash
sudo dmesg | grep -i nvidia
```

**Follow Live:**
```bash
sudo dmesg -w
```

#### journalctl
**Purpose:** Systemd journal (service logs)

**Service Logs:**
```bash
sudo journalctl -u supergfxd
```

**Recent Entries:**
```bash
sudo journalctl -u supergfxd -n 50
```

**Follow Live:**
```bash
sudo journalctl -u supergfxd -f
```

**Time Range:**
```bash
sudo journalctl -u supergfxd --since "10 minutes ago"
```

### Module Management

#### lsmod
**Purpose:** List loaded kernel modules

```bash
lsmod | grep nvidia
```

Output:
```
nvidia_drm            135168  1
nvidia_modeset       1814528  1 nvidia_drm
nvidia              14376960  3 nvidia_modeset
```

Shows:
- Module name
- Size (bytes)
- Use count
- Dependencies

#### modprobe
**Purpose:** Load/unload kernel modules

**Load:**
```bash
sudo modprobe nvidia
```

**Unload:**
```bash
sudo modprobe -r nvidia
```

**Show Dependencies:**
```bash
modprobe --show-depends nvidia-drm
```

#### modinfo
**Purpose:** Show module information

```bash
modinfo nvidia | head -20
```

Shows:
- Module path
- Description
- Author
- License
- Parameters
- Dependencies
- Aliases

### File System Inspection

#### cat
**Purpose:** Display file contents

```bash
cat /proc/cmdline  # Kernel parameters
cat /etc/os-release  # OS information
cat /sys/class/power_supply/BAT0/capacity  # Battery %
```

#### grep
**Purpose:** Search text patterns

**Recursive Search:**
```bash
grep -r "nvidia" /etc/modprobe.d/
```

**With Context:**
```bash
grep -A 5 -B 5 "error" /var/log/syslog
```

**Case Insensitive:**
```bash
grep -i "ERROR" /var/log/syslog
```

---

## What We Did - Step by Step Explanation

### Initial Problem Diagnosis

#### 1. Symptom Observation
**What:** System freezes randomly, external monitor works
**Significance:** External monitor stability indicated display routing issue, not hardware failure

#### 2. Checked Driver Version
```bash
nvidia-smi
```
**Found:** nvidia-driver-580-open (correct for RTX 50-series)
**Meaning:** Driver itself was appropriate for the hardware

#### 3. Checked GPU Mode
```bash
prime-select query
```
**Found:** "on-demand" mode
**Problem:** This configured NVIDIA as primary renderer with AMD display, causing cross-GPU routing

#### 4. Identified Display Architecture
```bash
xrandr  # (Would work on X11)
```
**Learned:** Internal display wired to AMD, external ports to NVIDIA
**Implication:** Desktop using NVIDIA + internal display = cross-GPU routing

### Solution Implementation

#### 5. Installed ASUS-Specific Tools
```bash
sudo add-apt-repository ppa:supergfxd/supergfxd
sudo apt update
sudo apt install supergfxctl asusctl
```

**Why:** Hardware-specific tool that understands ASUS MUX switch and display routing

**What Happened:**
- Downloaded supergfxctl packages from PPA
- Installed supergfxd daemon
- Installed asusctl utilities
- Created systemd service for supergfxd

#### 6. Attempted Mode Switch
```bash
sudo nano /etc/supergfxd.conf
# Changed "mode": "Integrated" to "mode": "Hybrid"
sudo systemctl restart supergfxd
```

**Problem:** NVIDIA modules failed to load
**Error:** "could not find module by name='off'"

**Why This Failed:**
- supergfxd tried to load NVIDIA modules
- modprobe encountered `alias nvidia off` in blacklist file
- System interpreted this as loading a module called "off"
- No such module exists → error

#### 7. Found Root Cause
```bash
grep -r "alias.*off" /etc/modprobe.d/ /lib/modprobe.d/
```

**Discovery:** `/lib/modprobe.d/blacklist-nvidia.conf` contained:
```
alias nvidia off
alias nvidia-drm off  
alias nvidia-modeset off
```

**Understanding:**
- File created by nvidia-prime package
- Intended to prevent NVIDIA loading in "intel" mode
- Outdated syntax (modern: use `install` directives)
- Conflicted with supergfxd's module management

#### 8. Fixed Module Blacklist
```bash
sudo sed -i '/^alias nvidia/d' /lib/modprobe.d/blacklist-nvidia.conf
```

**What This Command Does:**
- `sed` - Stream editor for text manipulation
- `-i` - Edit file in-place
- `/^alias nvidia/d` - Delete lines starting with "alias nvidia"
- Result: Removed problematic aliases, kept actual blacklists

**After:**
```
blacklist nvidia
blacklist nvidia-drm
blacklist nvidia-modeset
```

These `blacklist` directives are fine - they prevent auto-loading but allow manual loading.

#### 9. Loaded NVIDIA Modules Manually
```bash
sudo modprobe nvidia
sudo modprobe nvidia-modeset
sudo modprobe nvidia-drm
```

**What Happened:**
1. `modprobe nvidia`:
   - Loaded core NVIDIA driver
   - Initialized GPU hardware
   - Created `/dev/nvidia0` device

2. `modprobe nvidia-modeset`:
   - Loaded mode-setting module
   - Enabled KMS for NVIDIA
   - Prepared for display management

3. `modprobe nvidia-drm`:
   - Loaded DRM integration
   - Connected to kernel graphics subsystem
   - Enabled proper Linux graphics stack integration

**Verification:**
```bash
lsmod | grep nvidia
```
All modules loaded successfully with proper dependencies

#### 10. Verified NVIDIA Functionality
```bash
nvidia-smi
```

**Result:** RTX 5070 Ti detected and working
**Meaning:** Driver loaded correctly, GPU accessible

#### 11. Restarted supergfxd
```bash
sudo systemctl restart supergfxd
```

**What Happened:**
- Daemon detected loaded NVIDIA modules
- Applied Hybrid mode configuration  
- Set up power management (auto suspend)
- No more module loading errors

#### 12. Verified Hybrid Mode
```bash
supergfxctl -g
```
**Output:** `Hybrid`
**Meaning:** System correctly in Hybrid mode

```bash
glxinfo | grep "OpenGL renderer"
```
**Output:** `AMD Radeon Graphics`
**Meaning:** Desktop rendering with AMD (correct!)

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```
**Output:** `NVIDIA GeForce RTX 5070 Ti`
**Meaning:** NVIDIA available on-demand (correct!)

### Why the Fix Works

**Before (Broken):**
```
1. prime-select on-demand → NVIDIA renders desktop
2. Internal display wired to AMD
3. Frames: NVIDIA → copy → AMD → Display
4. Cross-GPU routing unstable → Freezes
```

**After (Fixed):**
```
1. supergfxctl Hybrid → AMD renders desktop
2. Internal display wired to AMD  
3. Frames: AMD → Display (direct)
4. No routing, stable operation
5. NVIDIA: Available for CUDA, suspends when idle
```

### Additional Optimizations

#### 13. Added Kernel Parameters
```bash
sudo nano /etc/default/grub
# Added: nvidia_drm.fbdev=1 ibt=off
sudo update-grub
```

**nvidia_drm.fbdev=1:**
- Enables framebuffer device support
- Better console graphics
- Improved boot splash

**ibt=off:**
- Disables Indirect Branch Tracking
- Compatibility workaround for some GPU firmware
- Prevents certain rare crashes

#### 14. Configured Module Options
```bash
sudo nano /etc/modprobe.d/nvidia.conf
```

**NVreg_PreserveVideoMemoryAllocations=1:**
- Preserves GPU memory across suspend
- Faster resume from suspend
- Better power management

**NVreg_TemporaryFilePath=/var/tmp:**
- Specifies temp file location
- Better than /tmp for large allocations

---

## Key Takeaways

### Understanding Hybrid Graphics
1. **Physical Wiring Matters:** Displays are hardwired to specific GPUs
2. **Cross-GPU Routing is Fragile:** Copying frames between GPUs causes instability
3. **Proper Configuration is Critical:** Desktop must render on the GPU connected to internal display

### Linux Graphics Stack
1. **Multiple Layers:** Kernel (DRM/KMS) → Driver → Display Server (Wayland/X11) → Applications
2. **Wayland vs X11:** Wayland handles hybrid graphics more elegantly
3. **CUDA is Separate:** ML/compute workloads bypass display system entirely

### Driver Management
1. **Modules are Not Automatic:** Kernel modules need explicit loading or configuration
2. **Configuration Conflicts:** Multiple tools (prime-select, supergfxctl) can conflict
3. **Hardware-Specific Tools Better:** supergfxctl > prime-select for ASUS laptops

### Power Management
1. **Runtime PM is Good:** GPU suspension saves 10-20W when idle
2. **Wake Delays are Normal:** 2-3 second delay on first nvidia-smi is expected and healthy
3. **Compute Workloads Unaffected:** GPU stays awake during ML training

### Diagnostic Methodology
1. **Isolate the Symptom:** External monitor working indicated display routing issue
2. **Check Each Layer:** Driver → modules → configuration → display routing
3. **Read Logs:** journalctl and dmesg reveal the real errors
4. **Understand the Hardware:** MUX switch and display wiring explained the behavior

---

## Further Learning Resources

### Official Documentation
- **NVIDIA Linux Driver:** [NVIDIA Unix Graphics Docs](https://download.nvidia.com/XFree86/Linux-x86_64/580.95.05/README/)
- **Kernel Graphics:** [DRM/KMS Documentation](https://www.kernel.org/doc/html/latest/gpu/index.html)
- **Wayland:** [Wayland Documentation](https://wayland.freedesktop.org/docs/html/)
- **ASUS Linux:** [asus-linux.org](https://asus-linux.org/)

### Command Man Pages
```bash
man nvidia-smi
man modprobe
man journalctl
man lspci
```

### Online Communities
- **r/linux_gaming** - Hybrid graphics discussions
- **r/ASUSROG** - ASUS-specific issues
- **Arch Wiki** - Excellent technical reference (applies to Ubuntu too)
- **Stack Overflow/Unix SE** - Specific troubleshooting questions

### Books
- "How Linux Works" by Brian Ward
- "Understanding the Linux Kernel" by Daniel P. Bovet
- "Linux Device Drivers" by Jonathan Corbet

---

## Glossary

**API (Application Programming Interface):** Set of functions/protocols for software interaction

**CUDA (Compute Unified Device Architecture):** NVIDIA's parallel computing platform

**Daemon:** Background service running continuously

**DRM (Direct Rendering Manager):** Linux kernel subsystem for GPU management

**eDP (Embedded DisplayPort):** Display interface for internal laptop screens

**Framebuffer:** Memory buffer containing frame data to display

**iGPU (Integrated GPU):** GPU built into CPU

**KMS (Kernel Mode Setting):** Display mode configuration in kernel space

**Module:** Loadable kernel code for adding functionality

**MUX Switch:** Hardware multiplexer for routing displays

**OpenGL:** Cross-platform graphics API

**PCIe (Peripheral Component Interconnect Express):** High-speed hardware interface

**PPA (Personal Package Archive):** Ubuntu third-party software repository

**PRIME:** Platform Runtime Integrated Mesa Extensions (hybrid graphics framework)

**Runtime PM:** Dynamic power management for devices

**Wayland:** Modern display server protocol

**X11:** Legacy display server protocol

---

## Summary

To troubleshoot Linux hybrid graphics issues, you need understanding of:

1. **Hardware:** GPU types, display wiring, MUX switches
2. **Linux Kernel:** Modules, parameters, DRM/KMS subsystems  
3. **Drivers:** NVIDIA architecture, module loading, configuration
4. **Display Stack:** Wayland/X11, OpenGL, rendering pipeline
5. **Power Management:** PCIe states, runtime PM, suspension
6. **Diagnostic Tools:** lspci, nvidia-smi, journalctl, modprobe

**The Core Principle:** Desktop rendering must use the GPU that's physically connected to the internal display. Cross-GPU routing causes instability.

**The Solution:** Use hardware-aware tools (supergfxctl for ASUS) that properly configure display routing, not generic solutions (prime-select) that may misconfigure based on assumptions.

---

**This guide gives you the foundation to understand not just what we did, but WHY each step mattered and how the entire system works together.**
