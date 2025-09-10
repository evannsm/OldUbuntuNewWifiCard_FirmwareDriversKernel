# 🐧 Intel Wi-Fi 7 BE201 on Ubuntu 22.04 (Jammy)

## Why this was needed
- The Intel Wi-Fi 7 BE201 (PCI ID 8086:7740) is only supported starting with Linux kernel 6.11+.
- Ubuntu 22.04 ships with older kernels (5.15 LTS, 6.5 HWE, 6.8), which do not recognize BE201.
- Even if firmware is present in /lib/firmware, the old kernel’s driver (iwlwifi) cannot use it.
- Solution: install a mainline kernel ≥ 6.11 and ensure firmware blobs are present.

## My steps:
Install of Ubuntu 22.04 on new DellPro16Plus didn't have access to Wifi chip
First solution was to update drivers. Failed because kernel wasn't compatible with them
I was stuck for 5 hours and [this website](https://www.intel.com/content/www/us/en/support/articles/000005511/wireless.html) finally helped me realize that for the WiFi card my computer has (Intel® Wi-Fi 7 BE201), the Ubuntu Kernel must be 6.11+
Then update kernel to kernel 16.xx.xx (latest) caused massive slowdown in the boot time (~2mins)

## Punchline:
1. Update kernel to the last 6.11.xx version (meets the minimum requirements needed while not risking being too modern and unstable/broken drivers/etc. it also has all the patches for the 6.11 kernel family and should be more stable than all other 6.11 versions)
2. This will cause kernel header issues so delete them
3. Last step is updating the drivers
4. Reboot to make changes show up

---

## Steps taken

### 1. Install Mainline Kernel Installer
You won't have Wi-Fi so connect to internet via ethernet or connect your iphone via USB and it'll automatically hotspot
```bash
    sudo add-apt-repository ppa:cappelikan/ppa
    sudo apt update
    sudo apt install -y mainline
```
Explanation:
- Adds a PPA with the `mainline` tool.
- `mainline` fetches kernels directly from kernel.ubuntu.com/mainline.
---

### 2. Install the latest kernel (≥ 6.11)
To see the full list of available kernels:
```bash
    mainline list
```

The most stable kernel in the 6.11 family will the the one at the top (aka the most recent). It should be 6.11.11
```bash
    mainline install 6.11.11
```
Explanation:
- Downloads and installs the desired kernel
- Packages installed: linux-image, linux-modules, linux-headers.

Note:
- On Jammy, the headers fail to configure because they depend on newer libraries:
  - libc6 (>= 2.38)
  - libelf1t64
  - libssl3t64
- This is expected. The kernel itself still installs and boots fine.

---

### 3. Remove headers
Check which broken packages are hanging around
```bash
    dpkg -l | grep 6.11.11
```
You’ll probably see things like:
- linux-headers-6.11.11-061111-generic
- linux-headers-6.11.11-061111
- Maybe linux-modules-… too

Remove broken headers
```bash
    sudo dpkg --remove --force-depends linux-headers-6.11.11-061111-generic linux-headers-6.11.11-061111
    sudo apt --fix-broken install
    sudo apt autoremove -y
```
Explanation:
- Removes unconfigurable headers that leave APT in a broken state.
- Keeps kernel image + modules (what you need).

### 4. Update linux firmware/drivers according to the new kernel

This command will give you info on what isn't working / what's missing and where to find what is missing:
```bash
    sudo dmesg | grep iwlwifi | tail -n 20
```

If not working, it'll say what the minimum / maximum version of the drivers are that you will need and also where to find them potentially. The next command will tell you how to get the drivers/firmware automatially wihout doing it all by hand by git cloning/downloading from here

This will install the new drivers for the kernel and make wifi work
```
    sudo apt install --reinstall linux-firmware
```

### 5. Reboot into the new kernel and verify
```bash
    sudo reboot
```

Verify:
```bash
    uname -r
    nmcli device status
    sudo dmesg | grep iwlwifi | tail -n 20
```
Expected:
- make sure the kernel is 6.11.11
- nmcli should be: one line with `wifi` type.
- nmcli status: state = connected or disconnected (not unavailable).
- the last command is the one that told you what was missing and where to get it. it should return everything good now

### Firmware for Intel BE201
- The BE201 requires firmware files (.ucode + .pnvm) in /lib/firmware.
- On this system, the needed files were already present:
    - iwlwifi-bz-b0-fm-c0-92.ucode
    - iwlwifi-bz-b0-fm-c0-93.ucode
    - iwlwifi-bz-b0-fm-c0-94.ucode
    - iwlwifi-bz-b0-fm-c0-96.ucode
    - iwlwifi-bz-b0-fm-c0-97.ucode
    - iwlwifi-bz-b0-fm-c0-98.ucode
    - iwlwifi-bz-b0-fm-c0-100.ucode
    - iwlwifi-bz-b0-fm-c0.pnvm

