# How I Fixed the Missing Bass on My ASUS ROG G14 2025 Running Ubuntu (And How You Can Too)

## When your $2000 gaming laptop sounds like a $20 speaker

*A complete guide to fixing Cirrus Logic audio amplifier issues on the 2025 ASUS ROG G14 with Ubuntu 24.04*

---

Picture this: You just unboxed your brand new ASUS ROG G14 (2025 model with that gorgeous RTX 5070 Ti), installed Ubuntu 24.04 because you're done with Windows bloat, and everything seems perfect. The display is stunning, the performance is incredible, and then... you play your first song.

Something's wrong. Terribly wrong.

The audio sounds thin, tinny, like someone sucked all the life out of it. You check the volume. It's maxed out. You try different audio apps. Same problem. You boot back into Windows just to verify you're not going crazy, and sure enough — the audio is perfect there. Rich, full, with actual bass.

Welcome to my weekend. And possibly yours too, if you're reading this.

---

## The Mystery of the Missing Woofers

After some digging (and by "some" I mean several hours of terminal commands and log reading), I discovered what was happening. The ROG G14 has a sophisticated audio setup:

- **2 tweeter speakers** (high frequencies) — these were working fine
- **2 woofer speakers** (bass frequencies) — these were completely silent

The woofers are powered by specialized chips called Cirrus Logic CS35L56 smart amplifiers. These little pieces of silicon are basically mini-computers that need firmware to function — think of firmware as the instruction manual that tells the hardware how to do its job.

Here's the kicker: **The firmware for the 2025 model doesn't exist in Ubuntu's official repositories yet.**

The laptop is too new. The Linux community hasn't caught up. Windows has these files pre-installed by ASUS, but Linux users? We're on our own.

---

## Before We Begin: What You'll Need

- ASUS ROG G14 (2025 model) — specifically with the device ID `10431024`
- Ubuntu 24.04 (or similar Linux distribution)
- Kernel 6.11 or newer (check with `uname -r`)
- About 30 minutes
- Basic terminal comfort (I'll explain everything, promise)

---

## Part 1: Confirming You Have the Same Problem

First, let's make sure we're fighting the same battle. Open a terminal (Ctrl+Alt+T) and run:

```bash
sudo dmesg | grep -i cs35l
```

Look for messages that say something like:

```
cs35l56-hda i2c-CSC3556:00-cs35l56-hda.0: .bin file required but not found
cs35l56-hda i2c-CSC3556:00-cs35l56-hda.1: .bin file required but not found
```

If you see these errors, you're in the right place. The system is detecting your amplifiers but can't load the firmware they need to function.

Now check your specific device ID:

```bash
sudo dmesg | grep "DSP system name"
```

You should see:

```
DSP system name: '10431024-spkid0', amp name: 'AMP1'
DSP system name: '10431024-spkid0', amp name: 'AMP2'
```

That `10431024` is your device ID. It's unique to the 2025 ROG G14. If you see a different number, this guide might still work, but adjust the commands accordingly.

---

## Part 2: Installing ASUS Control Software (Optional but Recommended)

The ASUS Linux community has created excellent tools for managing ROG laptops. While not strictly necessary for audio, they're incredibly useful for everything else (fan control, RGB, performance profiles).

Let's set them up:

```bash
# Create a workspace
cd ~/Desktop
mkdir -p git
cd git

# Clone the repository
git clone https://github.com/dariomncs/asus-ubuntu.git
cd asus-ubuntu
```

Now install the dependencies:

```bash
sudo apt update
sudo apt install -y libclang1 libseat1 libclang-dev libudev-dev \
    libfontconfig-dev build-essential cmake libxkbcommon-dev
```

Install the ASUS packages:

```bash
cd target
sudo dpkg -i asusctl*.deb supergfxctl*.deb
sudo systemctl enable --now asusd
```

This gives you proper control over your ASUS hardware. But we're here for the audio, so let's keep moving.

---

## Part 3: The Fix — Getting That Bass Back

Here's where the magic happens. Since the exact firmware for our device doesn't exist in the official repositories, we're going to use firmware from a similar ASUS device (the `10431b13` variant) as a workaround. These are the same amplifier chips with similar configurations — think of it like using a universal remote that's 95% perfect rather than waiting for one that doesn't exist yet.

### Step 1: Download the firmware

```bash
cd /tmp

# Download firmware from the kernel repository
wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cirrus/cs35l56-b0-dsp1-misc-10431b13-spkid0-amp1.bin

wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cirrus/cs35l56-b0-dsp1-misc-10431b13-spkid0-amp2.bin
```

These files are tiny (about 8KB each) but critical. They contain the instructions that tell your amplifiers how to process audio signals.

### Step 2: Install the firmware with your device's naming

```bash
# Copy and rename for your device ID
sudo cp cs35l56-b0-dsp1-misc-10431b13-spkid0-amp1.bin \
    /lib/firmware/cirrus/cs35l56-b0-dsp1-misc-10431024-spkid0-amp1.bin

sudo cp cs35l56-b0-dsp1-misc-10431b13-spkid0-amp2.bin \
    /lib/firmware/cirrus/cs35l56-b0-dsp1-misc-10431024-spkid0-amp2.bin
```

The key here is the renaming. The kernel looks for files with your specific device ID (`10431024`). We're giving it what it expects, even though we're using firmware from a similar device.

### Step 3: Create the DSP firmware symlink

```bash
cd /lib/firmware/cirrus
sudo ln -s cs35l56/CS35L56_Rev3.11.16.wmfw.zst \
    cs35l56-b0-dsp1-misc-10431024-spkid0.wmfw.zst
```

This creates a symbolic link (like a shortcut) to the DSP firmware. The amplifiers need two types of firmware:
- The `.bin` files (amplifier-specific tuning)
- The `.wmfw` file (DSP processing instructions)

### Step 4: Reboot

```bash
sudo reboot
```

I know, I know — this isn't Windows, we usually don't need to reboot for everything. But in this case, we're loading kernel modules and hardware firmware. A reboot ensures everything initializes cleanly.

---

## Part 4: The Moment of Truth

After your system boots back up, open a terminal and verify the fix:

```bash
sudo dmesg | grep -i cs35l | tail -20
```

You should **NOT** see the "file required but not found" errors anymore. Instead, you'll see messages about the amplifiers initializing successfully.

Now play some music. 

Really play it — something with good bass. I recommend Daft Punk's "Contact" or any Hans Zimmer soundtrack. You should immediately notice the difference. The woofers are alive. The bass is back. Your laptop finally sounds like the premium machine it is.

---

## What Just Happened? (The Technical Deep Dive)

For those curious about what we actually did, here's the breakdown:

**The Problem:**
Your laptop's audio system has a main audio chip (Realtek ALC295 codec) that handles basic audio. But for the woofers, ASUS uses specialized amplifiers (Cirrus CS35L56) that require firmware to function. These chips have built-in digital signal processors (DSPs) that need to be programmed for your specific speakers.

**The Solution:**
We gave the system firmware files it could work with. The `10431b13` firmware is from another ASUS ROG laptop with similar audio hardware. While not perfect (ideally we'd have the exact `10431024` firmware), it's close enough that the amplifiers can initialize and function properly.

**Why This Works:**
Both devices use the same Cirrus CS35L56 amplifier chips in similar laptop configurations. The firmware contains:
- Speaker protection algorithms (prevents damage from overdrive)
- Equalization curves (tuned for the speaker characteristics)
- Crossover settings (splits frequencies between tweeters and woofers)
- Power management

The differences between models are minor enough that the firmware provides good functionality.

---

## Troubleshooting

### Audio Still Sounds Wrong

Check if the firmware files are actually there:

```bash
ls -l /lib/firmware/cirrus/ | grep 10431024
```

You should see three files with your device ID.

### Getting Errors When Copying Files

Make sure you're using `sudo`. The `/lib/firmware/` directory is system-protected and requires administrator privileges to modify.

### Want to Reload Drivers Without Rebooting

Try this (though it might fail if audio is in use):

```bash
sudo modprobe -r snd_hda_codec_realtek snd_hda_scodec_cs35l56
sudo modprobe snd_hda_scodec_cs35l56 snd_hda_codec_realtek
```

If you get "module is in use" errors, you'll need to reboot.

---

## The Bigger Picture: Contributing Back

This solution is a workaround. The proper fix is getting the actual `10431024` firmware into the official Linux firmware repository. Here's how you can help:

### 1. Report to ASUS Linux Community

Visit [asus-linux.org](https://asus-linux.org/) and let them know about this issue. The more people report it, the faster it gets prioritized.

You can also join their Discord server for real-time discussion.

### 2. Submit to Linux Firmware Repository

The official repository is at:
- GitHub: [github.com/kernel-linux/linux-firmware](https://github.com/kernel-linux/linux-firmware)
- GitLab: [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)

File an issue requesting firmware for device ID `10431024`.

### 3. Share Your Experience

Post on:
- Reddit ([r/Ubuntu](https://reddit.com/r/Ubuntu), [r/ASUS](https://reddit.com/r/ASUS), [r/linuxhardware](https://reddit.com/r/linuxhardware))
- Ubuntu Forums ([ubuntuforums.org](https://ubuntuforums.org))
- Ask Ubuntu ([askubuntu.com](https://askubuntu.com))

The more visibility this gets, the more people it helps.

### 4. Extract the Real Firmware (Advanced)

If you have Windows installed alongside Linux, the proper firmware files exist somewhere in your Windows partition. Extracting and contributing them would help everyone. (This is beyond the scope of this article, but it's possible!)

---

## My Thoughts on Linux Hardware Support in 2025

This experience highlights both the beauty and challenge of Linux on cutting-edge hardware. 

**The Challenge:** When you buy the latest hardware, you're essentially beta testing Linux compatibility. Manufacturers optimize for Windows, and the Linux community plays catch-up.

**The Beauty:** Within hours of identifying the problem, I had a working solution cobbled together from community tools, open-source firmware, and helpful documentation. Try doing that on Windows or macOS when something breaks.

The ASUS Linux community, in particular, has been phenomenal. Projects like asusctl show what's possible when passionate developers tackle real problems. These tools are often better than ASUS's official Windows software.

---

## Final Thoughts

If you're running Linux on modern hardware, you're going to hit snags like this. But each problem you solve makes you more capable. You learn how your system actually works, not just how to click through someone else's GUI.

The ROG G14 is an excellent Linux laptop — amazing performance, great battery life, and solid build quality. With this audio fix, it's even better.

Got questions? Run into issues? Drop a comment below. I'm happy to help troubleshoot.

And if this guide helped you, consider:
- Sharing it with others facing the same issue
- Contributing to the ASUS Linux project
- Reporting the firmware gap to help future ROG G14 users

Happy computing, and enjoy that bass! 🎵

---

## Quick Reference Card

**Verify Your Device ID:**
```bash
sudo dmesg | grep "DSP system name"
```

**Check for Firmware Errors:**
```bash
sudo dmesg | grep -i cs35l | grep -i error
```

**Install Firmware (replace 10431024 with your device ID if different):**
```bash
cd /tmp
wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cirrus/cs35l56-b0-dsp1-misc-10431b13-spkid0-amp1.bin
wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cirrus/cs35l56-b0-dsp1-misc-10431b13-spkid0-amp2.bin
sudo cp cs35l56-b0-dsp1-misc-10431b13-spkid0-amp1.bin /lib/firmware/cirrus/cs35l56-b0-dsp1-misc-10431024-spkid0-amp1.bin
sudo cp cs35l56-b0-dsp1-misc-10431b13-spkid0-amp2.bin /lib/firmware/cirrus/cs35l56-b0-dsp1-misc-10431024-spkid0-amp2.bin
cd /lib/firmware/cirrus
sudo ln -s cs35l56/CS35L56_Rev3.11.16.wmfw.zst cs35l56-b0-dsp1-misc-10431024-spkid0.wmfw.zst
sudo reboot
```

**Verify Fix:**
```bash
sudo dmesg | grep -i cs35l | grep -E "(error|firmware|loaded)"
```

---

## Tags

#Linux #Ubuntu #ASUS #ROG #G14 #Audio #Firmware #Hardware #Tech #OpenSource #LinuxGaming #Tutorial

---

**About the Author**
*This guide was created through collaborative problem-solving and extensive testing on an ASUS ROG G14 2025 (RTX 5070 Ti) running Ubuntu 24.04. The solution has been verified to work as of December 31, 2024.*

**Tested Configuration:**
- Model: ASUS ROG G14 2025 (GA403UV)
- GPU: NVIDIA RTX 5070 Ti
- OS: Ubuntu 24.04.1 LTS
- Kernel: 6.14.0-37-generic
- Audio: Cirrus Logic CS35L56 amplifiers (Device ID: 10431024)

---

**Disclaimer:** This guide is provided as-is for educational and community benefit. While it has been tested and works, you proceed at your own risk. Always backup important data before making system changes.

---

*If this article helped you, please give it a clap (or 50!) and share it with others who might be struggling with the same issue. Together, we make Linux better for everyone.*

*Have suggestions for improving this guide? Found a better solution? Let me know in the comments!*
