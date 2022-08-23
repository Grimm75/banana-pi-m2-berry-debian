# Banana PI M2 Berry current Debian install
How to install and setup current Debian (Bullseye at the time of writing) base system for BPI-M2-Berry
## Prerequisites
- Banna PI M2 Berry board + USB micro 5V/2A power source
- 8GB (or bigger) SD card
- PC with Debian/Ubuntu system
- Wired network with DHCP
- USB keyboard + HDMI monitor
- (optional) UART/USB converter

## Prepare SD card
(on Debian/Ubuntu PC as `comon_user` with `sudo` to `root` rights)

### Use prebuilt images
Visit https://sd-card-images.johang.se/boards/banana_pi_m2_berry.html and follow instructions.

### Build your own image
```bash
# Install docker and ARM emulation stuff
sudo apt install docker.io qemu-system-arm qemu-user-static binfmt-support

# Enable support for seamless running of foreign architecture binaries
sudo binfmt-support --enable

# Grant current user access to docker
sudo usermod -G docker -a <common_user>

# HACK: make 'docker' group membership active in current shell
newgrp docker && newgrp <common_user_group>

# Prepare builder container and output directory for images
docker build -t sd-images https://github.com/johang/sd-card-images.git
mkdir -p ~/sd-images

# Build Debian rootfs image
docker run --rm -v ~/sd_images:/artifacts sd-images build-debian debian armhf bullseye

# Build U-Boot image
docker run --rm -v ~/sd_images:/artifacts sd-images build-boot \ 
banana_pi_m2_berry allwinner-v40 bananapi_m2_berry_defconfig arm-linux-gnueabihf

# Create SD-card image and copy it to SD
cd ~/sd_images
zcat boot-banana_pi_m2_berry.bin.gz debian-bullseye-armhf-*.bin.gz > sd.img
sudo dd if=sd.img of=/dev/mmcblk0
```

## Prepare end execute 1st boot
- Write down `root` password, see last part of Debian image filename
- Connect board to DHCP enabled wired network
- Connect USB keyboard + HDMI monitor or USB/UART conventor
- Boot using your SD-card.**
- Login as `root` via keyboard+monitor combo / UART / SSH

## After 1st boot
(on Banana PI as `root`)

```bash
# Set new password for root
passwd

# Set current time and date
# To get current date string, run on your PC: 
# LC_ALL=C date -u
date -s "Mon Aug 22 10:05:41 UTC 2022"

# Update & upgrade Debian
apt update && apt dist-upgrade

# Install wlan0 firmware
sed -i 's/^deb(.*)$/deb\1 contrib non-free/g' /etc/apt/sources.list
apt update
api install wireless-regdb firmware-brcm80211 

# Add symlink to similar board modprobe config (this board is missing in upstream)
cd /usr/lib/firmware/brcm/ && \
ln -s brcmfmac43430-sdio.sinovoip,bpi-m2-ultra.txt brcmfmac43430-sdio.sinovoip,bpi-m2-berry.txt

# Install BT firmware (maybe there is better source, but I'm lazy to search more)
apt install curl
curl -L -o /usr/lib/firmware/brcm/BCM43430A1.hcd \
https://github.com/RPi-Distro/bluez-firmware/raw/master/broadcom/BCM43430A1.hcd 

# Enable network time sync (optional but highly recomended)
apt install dbus systemd-timesyncd
timedatectl set-ntp 1  

# Resize root partition (optional)
apt install parted
parted
  > resizepart 2
  > answer 'Yes' to warning about partition in use
  > answer <maximum size> of your SD card
# Resize root filesystem
resize2fs /dev/mmcblk0p2
```
## Reboot.
Now all devices **should** working. 
### Tested devices:
- wlan0 (tested: connection to home wireless network, OK)
- BT (tested: device scan using hcitool, OK)
- eth0 (tested: connection to home wired network, OK)
- LEDs (tested: /sys/class/leds/* works, OK)
### Not tested devices:
- SATA
- audio

