# Nextcloud setup on rock64


## Assumptions

* The config was written for a rock64 4GB, if you have less RAM, probably worth fixing the config (php-fpm and postgresql)
* I use cloudflare for DNS and thus request the LE certificates through caddy's plugin for it. That allows me to request a certificate for a server not publicly exposed to the internet.
* I have a dns record set for my rock64
* This is for a local network so I didn't go crazy on firewalls and fail2ban and stuff


## Before starting

### Flash the board's SPI to be able to boot from USB

```bash
curl -L "https://github.com/ayufan-rock64/linux-u-boot/releases/latest/download/u-boot-flash-spi-rock64.img.xz" \
    | xzcat -v \
    | sudo dd of=/dev/mmcblk0 bs=1M
```

### Install debian on a usb drive

```bash
curl -L "https://github.com/ayufan-rock64/linux-build/releases/download/0.9.14/buster-minimal-rock64-0.9.14-1159-arm64.img.xz" \
    | xzcat -v \
    | sudo dd of=/dev/mmcblk0 bs=1M
```

### On the board:

* Setup your user
* Allow running sudo without password:
```bash
sudo tee /etc/sudoers.d/sudo-nopass <<< '%sudo ALL=(ALL:ALL) NOPASSWD: ALL'
```

### In this repo

* install depencies:
```bash
pipenv install
pipenv shell

cp defaults.yml _vars.yml # and fill them in

echo "hostname.local\n[auyfan_buster]\nhostname.local" > inventory.ini

ansible-playbook -e _vars.yml playbooks/global.yml
```


### Formatting disks with LVM

```bash
# on all disks
fdisk /dev/sdX # g, n (keep all defaults), t (1, e8), w
pvcreate /dev/sdX

vgcreate vg.nextcloud /dev/sd{b,c,d}
lvcreate -n nc-apps -L 1G vg.nextcloud # no mirror needed, easy to re-install
lvcreate -m 1 -n nc-postgresql -L 5G vg.nextcloud
lvcreate -m 1 -n nc-data -L 20G vg.nextcloud

mkfs.ext4 /dev/vg.nextcloud/nc-X

# adding a disk
vgextend vg.nextcloud /dev/sdX
# adding part of a disk
vgextend vg.nextcloud /dev/sda8
```

Once the disks are formatted, you can mount them either
* before installing software (easier, must know the directories and permissions)
* after installing software:
  * make sure the services are stopped
  * mount the partition to /mnt
  * copy files
  * remount the partition over the existing target folder
  * restart the services

## Still to do

* [ ] Fix task changed when updating nc config
* [ ] Make a playbook to setup LVM?
* [x] Backups
* [x] Default apps to install
* [ ] Static IP
* [ ] Prevent resizing the system's partition to be able to use the free space in the nextcloud's LVM VG
