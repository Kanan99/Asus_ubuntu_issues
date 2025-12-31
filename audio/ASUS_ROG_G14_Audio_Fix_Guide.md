# ASUS ROG G14 (2025 Model with RTX 5070 Ti) Audio Fix for Ubuntu 24.04

## Problem Description

The ASUS ROG G14 (2025 model with NVIDIA RTX 5070 Ti) running Ubuntu 24.04 experiences poor audio quality where only the tweeter speakers work, while the woofer speakers remain silent. This results in tinny, bass-less audio despite perfect audio quality on Windows.

### Symptoms
- Only high-frequency (tweeter) speakers produce sound
- Woofer speakers are completely silent
- Audio sounds thin and lacks bass
- Windows audio works perfectly fine
- Issue affects Ubuntu 24.04 with kernel 6.14.0-37-generic

### Root Cause

The Cirrus Logic CS35L56 audio amplifiers (which power the woofer speakers) are detected by the Linux kernel but fail to initialize due to missing device-specific firmware files. The kernel logs show:

```
cs35l56-hda i2c-CSC3556:00-cs35l56-hda.0: .bin file required but not found
cs35l56-hda i2c-CSC3556:00-cs35l56-hda.1: .bin file required but not found
```

The specific device ID `10431024-spkid0` is not yet included in the official Linux firmware repository.

## Hardware Configuration

- **Model**: ASUS ROG G14 (2025)
- **GPU**: NVIDIA RTX 5070 Ti
- **Audio Architecture**: 
  - 2x Tweeter speakers (working)
  - 2x Woofer speakers powered by Cirrus Logic CS35L56 amplifiers (not working)
- **Device ID**: 10431024-spkid0
- **Amplifiers**: AMP1 and AMP2

## Solution

### Prerequisites

- Ubuntu 24.04 (or similar Linux distribution)
- Kernel 6.11 or newer (confirmed working on 6.14.0-37)
- Root/sudo access

### Step 1: Verify the Issue

Check if your system detects the Cirrus amplifiers but can't load firmware:

```bash
sudo dmesg | grep -i cs35l
```

Look for messages about "DSP system name" and ".bin file required but not found".

Verify your device ID:

```bash
sudo dmesg | grep -i "DSP system name"
```

You should see something like:
```
cs35l56-hda i2c-CSC3556:00-cs35l56-hda.0: DSP system name: '10431024-spkid0', amp name: 'AMP1'
cs35l56-hda i2c-CSC3556:00-cs35l56-hda.1: DSP system name: '10431024-spkid0', amp name: 'AMP2'
```

### Step 2: Install ASUS Control Software (Optional but Recommended)

Clone and install the ASUS Linux utilities:

```bash
cd ~/Desktop
mkdir -p git
cd git
git clone https://github.com/dariomncs/asus-ubuntu.git
cd asus-ubuntu
```

Install required dependencies:

```bash
sudo apt update
sudo apt install -y libclang1 libseat1 libclang-dev libudev-dev \
    libfontconfig-dev build-essential cmake libxkbcommon-dev
```

Install the ASUS control packages:

```bash
cd target
sudo dpkg -i asusctl*.deb supergfxctl*.deb
sudo systemctl enable --now asusd
```

### Step 3: Download and Install Firmware

Since the exact firmware for device ID `10431024` doesn't exist in the official repository yet, we use firmware from a similar ASUS device as a workaround:

```bash
cd /tmp

# Download firmware from a similar ASUS device (10431b13)
wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cirrus/cs35l56-b0-dsp1-misc-10431b13-spkid0-amp1.bin
wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cirrus/cs35l56-b0-dsp1-misc-10431b13-spkid0-amp2.bin

# Copy firmware files with your device's naming convention
sudo cp cs35l56-b0-dsp1-misc-10431b13-spkid0-amp1.bin \
    /lib/firmware/cirrus/cs35l56-b0-dsp1-misc-10431024-spkid0-amp1.bin
sudo cp cs35l56-b0-dsp1-misc-10431b13-spkid0-amp2.bin \
    /lib/firmware/cirrus/cs35l56-b0-dsp1-misc-10431024-spkid0-amp2.bin

# Create symlink for the .wmfw file
cd /lib/firmware/cirrus
sudo ln -s cs35l56/CS35L56_Rev3.11.16.wmfw.zst \
    cs35l56-b0-dsp1-misc-10431024-spkid0.wmfw.zst
```

### Step 4: Reboot

```bash
sudo reboot
```

### Step 5: Verify the Fix

After reboot, check if firmware loaded successfully:

```bash
sudo dmesg | grep -i cs35l | grep -E "(error|firmware|loaded)"
```

You should no longer see the ".bin file required but not found" error.

Test your audio - the woofers should now be working, giving you full, rich sound with bass!

## Verification Commands

To check audio devices:
```bash
pactl list sinks short
aplay -l
```

To monitor for errors:
```bash
sudo dmesg | grep -i cs35l | tail -20
```

## Important Notes

1. **Firmware Workaround**: This solution uses firmware from a similar ASUS device (10431b13) as a workaround. The proper firmware for device ID `10431024` should be added to the official Linux firmware repository.

2. **Kernel Version**: Ensure you're running kernel 6.11 or newer. Cirrus Logic CS35L56 support was added in recent kernels.

3. **Future Updates**: When updating your system, check if the official firmware for `10431024` becomes available. You can monitor the linux-firmware git repository.

## Troubleshooting

### Audio Still Not Working
- Double-check that the firmware files are in `/lib/firmware/cirrus/`
- Verify the filenames match your device ID exactly
- Check kernel logs for new error messages
- Try unloading and reloading audio modules:
  ```bash
  sudo modprobe -r snd_hda_codec_realtek snd_hda_scodec_cs35l56
  sudo modprobe snd_hda_scodec_cs35l56 snd_hda_codec_realtek
  ```

### Distortion or Audio Issues
- The firmware mismatch (using 10431b13 instead of 10431024) might cause slight issues
- Report to ASUS Linux community for proper firmware
- Consider extracting actual firmware from Windows drivers if issues persist

## Where to Report/Share This Solution

### 1. ASUS Linux Community
- **Website**: https://asus-linux.org/
- **GitLab**: https://gitlab.com/asus-linux/
- **Discord**: Join the ASUS Linux Discord server
- **Action**: Report that ROG G14 2025 (device ID 10431024) needs firmware added to the repository

### 2. Linux Firmware Repository
- **GitHub**: https://github.com/kernel-linux/linux-firmware
- **GitLab**: https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
- **Action**: Submit an issue or pull request to add firmware for device ID 10431024

### 3. Ubuntu Forums/Communities
- **Ubuntu Forums**: https://ubuntuforums.org/
- **Ask Ubuntu**: https://askubuntu.com/
- **r/Ubuntu**: https://www.reddit.com/r/Ubuntu/
- **Action**: Share this guide to help others with the same laptop

### 4. ASUS ROG Forums
- **ROG Forum**: https://rog-forum.asus.com/
- **Action**: Help other ROG G14 owners who want to run Linux

### 5. Linux Audio Communities
- **r/linuxaudio**: https://www.reddit.com/r/linuxaudio/
- **Linux Audio Users Mailing List**
- **Action**: Document this as a reference for Cirrus Logic audio issues

### 6. GitHub Issue Tracker
- Create an issue in the asus-ubuntu repository: https://github.com/dariomncs/asus-ubuntu/issues
- Tag it as "audio" and "ROG G14 2025"

## Contributing

If you have the ROG G14 2025 model with working audio on Windows, you can help by:

1. **Extracting Firmware from Windows**: The proper firmware files exist in your Windows installation. Extracting and contributing them would help everyone.

2. **Testing**: If you try this solution, report back whether it works perfectly or if there are any audio quality issues.

3. **Hardware Variants**: If you have a different ROG G14 2025 variant, check your device ID and report it.

## Credits

Solution developed through collaborative troubleshooting, combining:
- ASUS Linux community tools and documentation
- Linux kernel firmware repository
- Understanding of ALSA/HDA audio subsystem
- Trial and error with Cirrus Logic CS35L56 drivers

## License

This guide is provided as-is for community benefit. Feel free to share, modify, and distribute.

---

**Last Updated**: December 31, 2024  
**Tested On**: ASUS ROG G14 2025 (RTX 5070 Ti), Ubuntu 24.04, Kernel 6.14.0-37-generic  
**Status**: Working with firmware workaround
