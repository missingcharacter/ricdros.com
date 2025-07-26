---
title:  'My Windows 11 and Pop!_OS 22.04 dual boot setup'
toc: true
date:   2022-08-07 00:00:00 +0000
---
## Tools and characteristics

- [GUID Partition Table (GPT)](https://en.wikipedia.org/wiki/GUID_Partition_Table)
- [UEFI](https://en.wikipedia.org/wiki/UEFI)
- [luks](https://gitlab.com/cryptsetup/cryptsetup/)
- [Ventoy](https://github.com/ventoy/Ventoy/releases)
- USB drive with 32+ GB of space
- ISOs:
  - [`Pop!_OS`](https://pop.system76.com/)
    ![download-pop-os](/assets/images/win-linux-dual-boot/01-download-pop-os.png)
  - [`Windows 11`](https://www.microsoft.com/software-download/windows11)
    ![download-win-11](/assets/images/win-linux-dual-boot/02-download-windows-11.png)

## Setup

### Install and configure Ventoy

- Ventoy installation instructions
  [here](https://www.ventoy.net/en/doc_start.html)
- Add `Windows` and `Pop!_OS` ISOs to ventoy folder in USB drive

### Create partitions first using `gparted` in live `Pop!_OS`

- Boot into `Pop!_OS` and create 5 partitions:

 1. The `EFI` partition I recommend you use 512MB `FAT32` (it is more than
    necessary)
 2. The `MSR` partition should be 128MB FAT32 will be for Windows
 3. The actual windows installation whatever you want, I recommend more than
    100GB `NTFS`
 4. The `/boot` partition for linux I recommend 1GB `xfs`
 5. The `Linux LVM` partition whatever is left of the disk using `linux pv`

### Install Windows 11, without encryption

- When choosing partitions press `Shift` + `F10` to open `command prompt`
- In Command prompt, run `diskpart`
- Run `list disk` to get a list of disks and their numbers
- Run `select disk <number>` to switch context to a specific disk
- Run `list partition` to get a list of partitions and their numbers, you
  should see the partitions you created in step `1`
- Run `select partition <number for EFI>`
- Run `create partition efi size=512` to create `efi` partition
- Run `select partition <number for MSR>`
- Run `create partition msr size=128` to create `msr` partition
- Run `exit` to go back to windows installation
- Make sure windows installer sees EFI and MSR partitions
- Select the partition you created in step `1` for windows to be installed on
- Install windows, updates, activate etc

### Now install `Pop!_OS` (or Ubuntu)

- Press `Try Demo Mode` to get to the desktop
- Open terminal
- `sudo su -`
- `fdisk -l` to find your `Linux LVM` partition
- `cryptsetup luksFormat /dev/nvme#n#p#` this will encrypt and ask you to
  setup a passphrase
- `cryptsetup open /dev/nvme#n#p# luks1` to decrypt the partition and mount it
  at `/dev/mapper/luks1`
- `pvcreate /dev/mapper/luks1` to create a `Physical Volume` that LVM can use
- `vgcreate vg_hostname /dev/mapper/luks1` (used `vg_thinkpad` last time)
- `lvcreate -L 100G -n lv_root vg_hostname` # at least 100 GB
- `lvcreate -L 4G -n lv_swap vg_hostname`
- `lvcreate -l100%FREE -n lv_home vg_hostname`
- Open the regular installer, choose `custom partitioning`
- Create `/boot` without encryption
- Select `EFI` partition and mount it at `/boot/efi`
- Decrypt `Linux LVM` partition
- Select 100 GB and pick `/` for it and format using `xfs`
- Select 4 GB as swap
- Select last LVM partition for `/home` and format using `xfs`
- Should look something like this
  ![custom-install](/assets/images/win-linux-dual-boot/00-gparted-looks-like.png)
- When the installer finishes, **don't reboot**
- Open terminal, again
- `sudo su -`
- `blkid /dev/nvme#n#p#` for the `Linux LVM` partition
- `cryptsetup open /dev/nvme#n#p# luks1` to decrypt the partition
- `parted -ls`
- `mount /dev/mapper/vg_thinkpad-lv_root /mnt` Change volume group name
- `mount /dev/mapper/vg_thinkpad-lv_home /mnt/home` Change vg name
- `mount /dev/nvme#n#p# /mnt/boot` Change device and partition numbers
- `mount /dev/nvme1n1p1 /mnt/boot/efi` Change device and partition numbers
- `for i in dev dev/pts proc sys run; do mount -B "/${i}" "/mnt/${i}"; done`
- `chroot /mnt`
- We are now inside the installed system, not the live environment
- `cat /etc/crypttab` Confirm UUID matched blkid output above

  ```shell
  # cat /etc/crypttab
  luks1 UUID=0000000 none luks
  ```

- if `/etc/crypttab` does not exist create it
- Next, confirm the `/` root partition UUID matches the `Pop_OS` UUID in
  `/boot/efi/EFI/Pop-OS*`

  ```shell
  # grep "$(ls -d /boot/efi/EFI/Pop_OS-* | cut -d '-' -f2-)" /etc/fstab
  UUID=3333333333  /  xfs  defaults  0  0
  ```

- Confirm you have `/`, `/boot`, `/boot/efi`, `/home` and swap partitions in
  `/etc/fstab`

  ```shell
  # cat /etc/fstab
  # /etc/fstab: static file system information.
  #
  # Use 'blkid' to print the universally unique identifier for a
  # device; this may be used with UUID= as a more robust way to name devices
  # that works even if disks are added and removed. See fstab(5).
  #
  # <file system>  <mount point>  <type>  <options>  <dump>  <pass>
  PARTUUID=44444  /boot/efi  vfat  umask=0077  0  0
  UUID=55555  /boot  xfs  defaults  0  0
  UUID=666666  /home  xfs  defaults  0  0
  UUID=3333333333  /  xfs  defaults  0  0
  /dev/dm-2  none  swap  defaults  0  0
  ```

- if `swap` in `/etc/fstab` does not show `UUID`, you can obtain it via the
  command: `blkid | grep swap`, now update swap line in `/etc/fstab`
- As of kernel `5.11.0-40-generic` there's a ~45-second pause at boot while
  the system tries to find a non-existent resume device, so we'll disable
  resume.
- Create the file `/etc/initramfs-tools/conf.d/noresume.conf` with contents:

  ```text
  RESUME=none
  ```

- If you want to mount `/tmp` as `tmpfs` (ramdisk) then:

  ```shell
  ln -s /usr/share/systemd/tmp.mount /etc/systemd/system/
  systemctl enable tmp.mount
  ```

### Add timeout to UEFI boot loader

Reboot. Spam your spacebar for the menu. Select with arrows, add timeout with
"t" or reduce with "T" (+/- also work), select default with "d". Hold "l" to
boot linux after POST or "w" to boot Windows after POST without visiting the
menu.

## Related links

- [Breaking luks encryption](https://blog.elcomsoft.com/2020/08/breaking-luks-encryption/)
- [How to install Ubuntu with LUKS Encryption on LVM - superjamie gist](https://gist.github.com/superjamie/d56d8bc3c9261ad603194726e3fef50f)
- [How to install Ubuntu encrypted with LUKS with dual-boot](https://super-unix.com/ubuntu/ubuntu-how-to-install-ubuntu-encrypted-with-luks-with-dual-boot/)
- [reddit - Is there a way to Dual boot Pop OS and Windows 10?](https://www.reddit.com/r/pop_os/comments/mme286/is_there_a_way_to_dual_boot_pop_os_and_windows_10/)
- [Dual Boot Pop!_OS with Windows using systemd-boot - github.com/spxak1/weywot](https://github.com/spxak1/weywot/blob/main/Pop_OS_Dual_Boot.md)
- [How can I install Ubuntu encrypted with LUKS with dual-boot? - askubuntu](https://askubuntu.com/questions/293028/how-can-i-install-ubuntu-encrypted-with-luks-with-dual-boot)
- [Full_Disk_Encryption_Howto_2019 - help.ubuntu.com](https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019)
- [Creating logical volumes - RedHat 5 at mit.edu](https://web.mit.edu/rhel-doc/5/RHEL-5-manual/Cluster_Logical_Volume_Manager/LV_create.html)
