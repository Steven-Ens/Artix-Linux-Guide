# Artix Linux Installation Guide
* Official Installation Guide: https://wiki.artixlinux.org/Main/Installation

## What This Guide Provides
* This guide implements the following:
	* GPT + UEFI installation
    * rEFInd boot manager
    * runit init system
    * LUKS-encrypted root partition

## Verify the ISO Checksum
```
$ sha256sum artix-base-runit-version-x86_64.iso
```
* Compare the output against the SHA256 checksum listed on the Artix download page (https://artixlinux.org/download.php).
  
## Verify the ISO Signature
```
$ gpg --auto-key-retrieve --verify artix-base-runit-version-x86_64.iso.sig artix-base-runit-version-x86_64.iso
```
* Make sure the RSA key is the key specified on the Artix website under 'Official ISO Images' (eg. ```0xB886B428```).
* Ensure you see ```Good signature from "Christos Nouskas <nous@artixlinux.org>"```
* You may see:
```
WARNING: This key is not certified with a trusted signature!
```
* This warning is normal on a fresh system. It means the signing key is not yet trusted in your local GPG keyring through GPG’s web-of-trust model.

## Prepare the Installation Medium
* Make sure that the USB is not mounted.
* Run the following command, replacing ```/dev/sdX``` (or ```/dev/nvmeXn1```) with your drive and not appending a partition number:
```
$ dd if=~/artix-base-runit-version-x86_64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```
* ```if``` → Input file
* ```of``` → Output file
* ```bs``` → Writes in 4 MiB chunks
* ```status=progress``` → Shows transfer progress
* ```conv=fsync``` → Flushes data before dd exits

## Verify UEFI Boot Mode
```
# cat /sys/firmware/efi/fw_platform_size
```
* If the command returns 64, the system is booted in UEFI mode and has a 64-bit x64 UEFI.
* If it returns ```No such file or directory```, the system may be booted in BIOS or CSM mode.

## ThinkPad T440 UEFI Troubleshooting
* On the ThinkPad T440, ```UEFI Only``` mode may fail to boot Linux installation media even when the USB is properly configured for UEFI.
* After resetting the BIOS to default settings, these boot options worked:
	* ```Boot Mode: Both```
	* ```Boot Priority: UEFI First```
	* ```CSM Support: Yes``` (Automatic)

## Partition the Disk
* List the block devices and identify the target drive ```/dev/sdX```:
```
# lsblk
```
* For a UEFI system, use a GPT partition table with the following layout: 
	* /dev/sdX1 for EFI System (1GB)
	* /dev/sdX2 for Linux swap (8-16GB)
	* /dev/sdX3 for Linux filesystem (Remaining space)

* Open fdisk on the target drive, not a partition:
```
# fdisk /dev/sdX
```
* Inside fdisk:
	* ```g``` → Create a new GPT partition table
	* ```n``` → Create a new partition
	* ```t``` → Change partition type
   		* ```1``` → EFI System
       	* ```19``` → Linux swap
       	* ```20``` → Linux filesystem (Default)
	* ```w``` → Write changes to disk and exit
* Verify your partitions:
```
# fdisk -l /dev/sdX
```

## Encrypt /dev/sdX3 with LUKS
* Create the encrypted container:
```
# cryptsetup luksFormat /dev/sdX3
```
* Type ```YES``` and enter your passphrase.
* Unlock the encrypted partition:
```
# cryptsetup open /dev/sdX3 cryptroot
```
* This creates ```/dev/mapper/cryptroot```

## Format the Partitions
* Format the boot partition to FAT32:
```
# mkfs.fat -F 32 /dev/sdX1
```
* Initialize and enable the swap partition:
```
# mkswap /dev/sdX2
# swapon /dev/sdX2
```
* Format the unlocked encrypted root partition:
```
# mkfs.ext4 /dev/mapper/cryptroot
```

## Mount the Filesystems
* Mount the filesystem on the encrypted root device to ```/mnt```:
```
# mount /dev/mapper/cryptroot /mnt
```
* Mount the EFI System Partition (ESP) to ```/mnt/boot```:
```
# mount --mkdir /dev/sdX1 /mnt/boot
```
* Verify the mounted filesystems:
```
# lsblk
```

## Setup Internet Connection
* If using WiFi, connect to your network:
```
# nmtui
```
* Confirm the connection:
```
# ip addr 
# ping -c4 artixlinux.org
```

## Select the Mirrors
* Mirrors are defined in /etc/pacman.d/mirrorlist, ensure the ones on the top are closest to you geographically:
```
# vim /etc/pacman.d/mirrorlist
```

## Install the Base System
```
# basestrap /mnt base base-devel runit elogind-runit linux-lts linux-firmware vim cryptsetup 
```

## Generate the Filesystem Table
```
# fstabgen -U /mnt >> /mnt/etc/fstab
```
* ```U``` → Use UUIDs instead of device names.
* Confirm:
```
# cat /mnt/etc/fstab
```

## chroot into the New System
```
# artix-chroot /mnt /bin/bash
```

## Install microcode:
```
# pacman -S intel-ucode
```
* Or:
```
# pacman -S amd-ucode
```

## Configure mkinitcpio for LUKS Encryption
```
# vim /etc/mkinitcpio.conf
```
* Find the uncommented ```HOOKS=(...)```:
* Add ```encrypt``` before ```filesystems```
* Remove ```consolefont```
```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap block encrypt filesystems fsck)
```
* Rebuild initramfs:
```
# mkinitcpio -P
```

## Update the System Clock
```
# pacman -S ntp
# ntpd -qg
```
* ```q``` → Quit after syncing once.
* ```g``` → Allow a large initial time correction.

## Set the Time Zone:
```
# ln -sf /usr/share/zoneinfo/Canada/Pacific /etc/localtime
```

## Run hwclock to Generate /etc/adjtime:
```
# hwclock --systohc
```
* ```sys``` → System clock
* ```to``` → To
* ```hc``` → Hardware clock

## Localization
* Uncomment ```en_US.UTF-8 UTF-8``` and ```en_US ISO-8859-1``` in /etc/locale.gen:
```
# vim /etc/locale.gen
```
* Generate locales:
```
# locale-gen
```
* Set the system locale:
```
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```
* Confirm:
```
# locale -a
```

## Configure rEFInd:
```
# pacman -S refind
```
* Install:
```
# refind-install
```
* Edit the config:
	* Change ```timeout``` to ```-1``` seconds to automatically start the kernel.
	* Uncomment ```textonly``` to remove the icons.
```
# vim /boot/EFI/refind/refind.conf
```
* Configure refind_linux.conf to pass the correct LUKS boot options, starting with the UUID of ```/dev/sdX3```:
```
# blkid -s UUID -o value /dev/sdX3 > /boot/refind_linux.conf
```
* Add the following line to the top of ```/boot/refind_linux.conf``` using the UUID of ```/dev/sdX3```:
```
"Boot with LUKS" "cryptdevice=UUID=abcd1234-5678-efgh:cryptroot root=/dev/mapper/cryptroot rw"
```

## Install networkmanager
```
# pacman -S networkmanager networkmanager-runit
```
* Autostart with runit:
```
# ln -s /etc/runit/sv/NetworkManager /etc/runit/runsvdir/default/
```
* Confirm NetworkManager is running:
```
# sv status NetworkManager
```

## Set hostname
```
# echo <hostname> > /etc/hostname
```

# Set root Password
```
# passwd
```

# Add New User 
* Add user to the wheel group:
```
# useradd -m -G wheel user
```
* ```m``` → Create home directory
* ```G``` → Add supplementary groups
* Set user password:
```
# passwd user
``` 

## Add User to sudoers
* Give wheel group sudo permissions:
```
# vim /etc/sudoers
```
* Uncomment ```%wheel ALL=(ALL:ALL) ALL```

## Reboot
* Exit the chroot environment:
```
# exit
```
* Recursively unmount /mnt and reboot:
```                         
# umount -R /mnt
# reboot
```

## Install the Terminal
```
$ sudo pacman -S rxvt-unicode 
```

## Install the System Font
```
$ sudo pacman -S ttf-dejavu
```

## Install xorg:
```
$ sudo pacman -S xorg-server xorg-xinit xorg-xset
```

## Install i3wm:
* Select ```i3-wm```, ```i3status``` and ```i3lock```:
```
$ sudo pacman -S i3 
```

## Install man and git
```
$ sudo pacman -S man git
```

## Install firefox
```
$ sudo pacman -S firefox
```

## Download Dotfiles
```
$ git clone https://github/com/Steven-Ens/Dotfiles .
```
* Copy files to their locations.

## Install ufw
```
$ sudo pacman -S ufw ufw-runit
```
* Autostart with runit:
```
$ sudo ln -s /etc/runit/sv/ufw /run/runit/service/
```
* Configure settings:
```
$ sudo ufw default allow outgoing
$ sudo ufw default deny incoming
```
* Enable ufw rules:
```
$ sudo ufw enable
```
* Confirm ufw is running:
```
$ sudo sv status ufw
$ sudo ufw status verbose
```

## Private Internet Access VPN


## Auto Mount USB:
* Find the UUID of the USB drive:
```
$ lsblk -f
```
* Add the following line to ```/etc/fstab``` using the device UUID:
```
# UUID=<UUID>  /mnt/usb  ext4  nofail  0 2
```
* ```nofail``` → Continue booting even if the USB drive is disconnected. → Continue booting even if the USB drive is disconnected.
* ```dump``` → Legacy backup field used by the ```dump``` utility. Almost always set to ```0```.
* ```fsck``` → Sets filesystem check order at boot.
    * ```1``` → Root filesystem
    * ```2``` → Other filesystems
    * ```0``` → Disable checking

## Auto Backup home
* Install:
```
$ sudo pacman -S rsync cronie cronie-runit
```
* Autostart the service:
```
$ sudo ln -s /etc/runit/sv/cronie /run/runit/service/
```
* Create a separate mount point for usb devices ```/mnt/usb```:
```
# mkdir /mnt/usb/
```
* Create the backup script:
```
$ sudo vim /etc/cron.hourly/backup
```
```bash
#!/bin/bash
rsync -a --delete /home/steve/ /mnt/usb/
```
* ```a``` → Archive mode that recursively preserves permissions, ownership, timestamps, symlinks, and other file attributes.
* ```delete``` → Removes files from the backup that no longer exist in the source directory, keeping an exact mirror.
* Make the script executable:
```
$ sudo chmod +x /etc/cron.hourly/backup
```
* Verify cronie is running:
```
$ sudo sv status cronie
```

# First Update of the System

## pacman
* Uncomment the following lines in ```/etc/pacman.conf```:
* ```Color``` → Improves readability by highlighting package information and status messages.
* ```CheckSpace``` → Verifies sufficient disk space is available before installing or upgrading packages.
* ```VerbosePkgLists``` → Displays installed and available package versions during upgrades, making changes easier to review.

## Install pacdiff
```
$ sudo pacman -S pacman-contrib
```

## Initialize Pacman Keys
```
# pacman-key --init
```
* ```init``` → Creates the local pacman GPG keyring used to verify package signatures.
```
# pacman-key --populate artix
```
* ```populate``` → Imports the trusted Artix package signing keys into pacman's keyring.
* ```artix``` → Uses the Artix keyring only.

# System Maintenance

## Update the System
```
# pacman -Syu
```
* ```S``` → Synchronize packages.
* ```y``` → Refresh package database.
* ```u``` → Upgrade all installed packages.
* Review differences between configuration files and their corresponding ```.pacnew``` files, then merge or remove them as needed.
```
$ sudo pacdiff
```

## Troubleshooting pacman
* If you see errors related to the package signing keyring, refresh the keys:
```
$ sudo pacman-key --refresh-keys
```

## Using runit:
* Enable a service:
```
# ln -s /etc/runit/sv/<service> /etc/runit/runsvdir/default/
```
* Disable a service for the next boot:
```
# rm -f /etc/runit/runsvdir/default/
```
* Start a service:
```
# sv up service
```
* Stop a service (it still starts on the next boot):
```
# sv down service
```
* Restart a service:
```
# sv restart service
```
* Service status:
```
# sv status service
```
* List all running services:
```
# sv status /run/runit/service/*
```

## Using pacman:
-List all explicitly installed packages: 
pacman -Qe.
-List orphan programs (dependencies of deleted program)
$ sudo pacman -Qtdq
-Recursively delete orphans:
$ sudo pacman -Rns $(pacman -Qtdq)
-To remove a package and its dependencies which are not required by any other installed package:
# pacman -Rs package_name







OpenSSH:
# pacman -S openssh openssh-runit
# ln -s /etc/runit/sv/sshd/ /run/runit/service/
Key Creation and Copying:
-

SCP:
-Copy file from remote host to local host
$ scp username@from_host:file.txt /local/directory/
-Copy file from local host to remote host
$ scp file.txt username@to_host:/remote/directory/
-Copy directory from remote host to local host
$ scp -r username@from_host:/remote/directory /local/directory
-Copy directory from local host to remote host
$ scp -r /local/directory/ username@to_host:/remote/directory/

docker:
sudo pacman -S docker docker-runit
sudo ln -s /etc/runit/sv/docker /run/runit/service

dbus and elogind error:
elogind[827]: Failed to connect to system bus: No such file or directory
elogind[827]: Failed to fully start up daemon: No such file or directory
-That's because runit services are started in parallel, elogind immediately looks for a dbus service, while dbus was still initializing. Add a 0.4 second delay before the elogind dbus check:
# vim /etc/runit/runsvdir/current/elogind/run
-Add the following line above the dbus check so it's the first line under #!/bin/bash
#!/bin/bash
sleep 0.4

to stop the 'zoom' udev error on login screen:
copy /lib/udev/hwdb.d/60-keyboard.hwdb to /etc/udev/hwdb.d/
comment out the following lines with 'zoom' on them:
683, 689, 703, 731, 749, 1456       (880 and 881 995 test it)
next:
$ sudo udevadm hwdb --update
$ sudo udevadm control --reload-rules
$ sudo udevadm trigger



df vs du???
-Check disk usage of directories/ like /home/steve/
$ du -sh /home/steve
-s gives total size of specified folder
-h is human readable format

zip/unzip stuff and quick usage 

ln -s directory/file1 directory/file2
-Linking existing file1 to new file2 that will be created
-Can also link directories to directories:
ln -s /home/steve directory
-This would link /home/steve to a created directory called 'directory'
-hard link by default, that's why you specify -s for symbolic or 'soft' link
-Any changes to either file is reflected in the other
-Can unlink the link
unlink name_of_systemlink
-You can have broken links if the file/directory that the link points to changes path or is deleted
-If you get permission denied while trying to edit it you may have created a reference to itself, try going in the directory you're linking to and/or using full directory paths=

xrandr:
-Set screen resolution and configure monitors as it's 'Resize and Rotate'  
$ sudo pacman -S xorg-xrandr
-type
$ xrandr
-It will show names/resolutions of different outputs. * in the resolution line shows the current refresh rate and resolution, and + shows the preferred one

Screen Brightness and Laptop Brightness Keys:
-xorg-xbacklight by itself wasn't working, so use 
$ sudo pacman -S acpilight
-acpilight provides xorg-xbacklight so don't install it too, it is backwards compatible and uses ACPI interface to set display brightness 
-Place the following data in a created file /etc/udev/rules.d/90-backlight.rules. As you can see it allows members of wheel group to adjust the brightness file by changing its permissions to be writable by wheel group members:
SUBSYSTEM=="backlight", ACTION=="add", \
  RUN+="/bin/chgrp wheel /sys/class/backlight/intel_backlight/brightness", \
  RUN+="/bin/chmod g+w /sys/class/backlight/intel_backlight/brightness"
-reboot to see changes
-Can adjust brightness between 0 and 100

interactive
alias rm='rm -i'
alias mv='mv -i'
alias cp='cp -i'






