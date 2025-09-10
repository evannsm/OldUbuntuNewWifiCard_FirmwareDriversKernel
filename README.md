# 🐧 Intel Wi-Fi 7 BE201 on Ubuntu 22.04 (Jammy)
I was stuck for 5 hours and [this website](https://www.intel.com/content/www/us/en/support/articles/000005511/wireless.html) finally helped me realize that for the WiFi card my computer has (Intel® Wi-Fi 7 BE201), the Ubuntu Kernel must be 6.11+

This prompted the following (used chatgpt to summarize my steps and reasoning, there may be weirdness but the main point is that you do mainline kernel changes and then it basically works except for some headers that you can then get rid of. Only drawbacks so far are that booting up seems to take forever when computer is off/rebooting/etc)
## Why this was needed
- The Intel Wi-Fi 7 BE201 (PCI ID 8086:7740) is only supported starting with Linux kernel 6.11+.
- Ubuntu 22.04 ships with older kernels (5.15 LTS, 6.5 HWE, 6.8), which do not recognize BE201.
- Even if firmware is present in /lib/firmware, the old kernel’s driver (iwlwifi) cannot use it.
- Solution: install a mainline kernel ≥ 6.11 and ensure firmware blobs are present.

---

## Steps taken

### 1. Install Mainline Kernel Installer
Command:
    sudo add-apt-repository ppa:cappelikan/ppa
    sudo apt update
    sudo apt install -y mainline

Explanation:
- Adds a PPA with the `mainline` tool.
- `mainline` fetches kernels directly from kernel.ubuntu.com/mainline.

---

### 2. Install the latest kernel (≥ 6.11)
Command:
    mainline install-latest

Explanation:
- Downloads and installs the newest available kernel.
- In this case: 6.16.6.
- Packages installed: linux-image, linux-modules, linux-headers.

Note:
- On Jammy, the headers fail to configure because they depend on newer libraries:
  - libc6 (>= 2.38)
  - libelf1t64
  - libssl3t64
- This is expected. The kernel itself still installs and boots fine.

---

### 3. Reboot into the new kernel
Command:
    sudo reboot

Verify:
    uname -r
Expected:
    6.16.6-061606-generic

---

### 4. Remove broken headers (clean up APT)
Command:
    sudo dpkg --remove --force-depends linux-headers-6.16.6-061606-generic linux-headers-6.16.6-061606
    sudo apt -f install

Explanation:
- Removes unconfigurable headers that leave APT in a broken state.
- Keeps kernel image + modules (what you need).

---

### 5. Firmware for Intel BE201
- The BE201 requires firmware files (.ucode + .pnvm) in /lib/firmware.
- On this system, the needed files were already present:
    iwlwifi-bz-b0-fm-c0-92.ucode
    iwlwifi-bz-b0-fm-c0-93.ucode
    iwlwifi-bz-b0-fm-c0-94.ucode
    iwlwifi-bz-b0-fm-c0-96.ucode
    iwlwifi-bz-b0-fm-c0-97.ucode
    iwlwifi-bz-b0-fm-c0-98.ucode
    iwlwifi-bz-b0-fm-c0-100.ucode
    iwlwifi-bz-b0-fm-c0.pnvm
- The 6.16 driver successfully loaded these and brought the Wi-Fi card up.
- Without the new kernel, these blobs alone wouldn’t work.

Check what firmware was loaded:
    sudo dmesg | grep iwlwifi | grep 'loaded firmware'

---

### 6. Verify Wi-Fi is working
Command:
    nmcli device status

Expected:
- One line with `wifi` type.
- State = connected or disconnected (not unavailable).

---

## 📌 Summary
1. Installed mainline kernel 6.16.6 using the `mainline` tool.
2. Removed broken headers that depended on newer Ubuntu libraries.
3. Rebooted into the new kernel (`uname -r` → 6.16.6).
4. Kernel driver (iwlwifi) now supports Intel Wi-Fi 7 BE201.
5. Firmware (bz-b0-fm-c0-*) was already in /lib/firmware, so Wi-Fi came up immediately.
6. Verified Wi-Fi works via `nmcli`.

---

## 🛠 Optional: Lock default kernel
Command:
    grep "GRUB_DEFAULT" /etc/default/grub
Then change the line to:
    GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.16.6-061606-generic"

Finally update GRUB:
    sudo update-grub

Explanation:
- Ensures GRUB always boots into the new kernel by default.
- Prevents fallback to 6.8 unless manually selected.

---

## 🔍 Troubleshooting
- If Wi-Fi disappears after an update:
      uname -r                 # check if you booted into 6.16
      sudo dmesg | grep iwlwifi | tail -n 40   # check firmware load
      nmcli device status      # confirm device state
- If APT breaks again:
      sudo apt -f install
- If GRUB reverts to old kernel, repeat the GRUB_DEFAULT step.
