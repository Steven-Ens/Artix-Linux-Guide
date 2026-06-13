# Artix Linux Installation Guide
* Official Installation Guide: https://wiki.artixlinux.org/Main/Installation

## What This Guide Provides
* This guide implements the following:
	* rEFInd
   	* runit
 	* LUKS
    * 

## Verify the ISO Checksum
```
$ sha256sum artix-base-runit-version-x86_64.iso
```
* Compare the output against the SHA256 checksum listed on the Artix download page (https://artixlinux.org/download.php).
## Verify the ISO Signature
```
$ gpg --auto-key-retrieve --verify artix-base-runit-version-x86_64.iso.sig artix-base-runit-version-x86_64.iso
```
* Make sure ```using RSA key``` is the key specified on the website under 'Official ISO Images' (eg. 0xB886B428).
* Ensure you see ```Good signature from "Christos Nouskas <nous@artixlinux.org>"```
* You may see:
```
WARNING: This key is not certified with a trusted signature!
```
* This warning is normal on a fresh system. It means the signing key is not yet trusted in your local GPG keyring through GPG’s web-of-trust model.

## Prepare the Installation Medium
* Make sure that the USB is NOT mounted.
* Run the following command, replacing /dev/sdX with your drive and NOT appending a partition number:
```
$ dd if=~/artix-base-runit-version-x86_64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```
* ```if``` → Input file
* ```of``` → Output file
* ```bs``` → Writes in 4 MiB chunks
* ```status=progress``` → Shows transfer progress
* ```conv=fsync``` → Flushes data before dd exits

## ThinkPad T440 UEFI Troubleshooting
* On the ThinkPad T440, ```UEFI Only``` mode may fail to boot Linux installation media even when the USB is properly configured for UEFI.
* After resetting the BIOS to default settings, these boot options worked:
	* ```Boot Mode: Both```
	* ```Boot Priority: UEFI First```
	* ```CSM Support: Yes (Automatic)```

## Check Whether the Installation Medium was Booted in UEFI Mode
```
# cat /sys/firmware/efi/fw_platform_size
```
* If the command returns 64, the system is booted in UEFI mode and has a 64-bit x64 UEFI.
* If it returns ```No such file or directory```, the system may be booted in BIOS or CSM mode.

## Partition the Disk
* List the block devices and identify the target drive:
```
# lsblk
```
* For a UEFI system, use a GPT partition table with the following layout: 
	* /dev/sdX1 for EFI System (1GB)
	* /dev/sdX2 for Linux swap (8-16GB)
	* /dev/sdX3 for Linux filesystem (remaining space)

* Open fdisk on the target drive and NOT adding a partition:
```
# fdisk /dev/sdX
```
* Inside fdisk:
	* g → Create a new GPT partition table
	* n → Create a new partition
	* t → Change partition type
   		* 1 → EFI System
       	* 19 → Linux swap
       	* 20 → Linux filesystem (Default)
	* w → Write changes to disk and exit
* Verify your partitions:
```
# fdisk -l /dev/sdX
```

## Encrypt /dev/sdX3 with LUKS
* Create encrypted container:
```
# cryptsetup luksFormat /dev/sdX3
```
* Type ```YES``` and enter your passphrase.
* Unlock the encrypted partition:
```
# cryptsetup open /dev/sda3 cryptroot
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
* Format the encrypted root partition:
```
# mkfs.ext4 /dev/mapper/cryptroot
```

## Mount the Filesystems
* Mount the encrypted root partition to /mnt:
```
# mount /dev/mapper/cryptroot /mnt
```
* Mount the EFI partition to /mnt/boot:
```
# mount --mkdir /dev/sdX1 /mnt/boot
```
* Create a separate mount point for usb devices /mnt/usb:
```
# mkdir /mnt/usb
```
* To confirm your mount points:
```
# lsblk
```

## Internet Connection
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
* Mirror services are defined in /etc/pacman.d/mirrorlist, ensure the ones on the top are closest to you geographically:
```
# vim /etc/pacman.d/mirrorlist
```

## Install the Base System
```
# basestrap /mnt base base-devel runit elogind-runit linux-lts linux-firmware vim cryptsetup 
```

## Generate the File System Table
```
# fstabgen -U /mnt >> /mnt/etc/fstab
```
* Check that it succeeded:
```
# cat /mnt/etc/fstab
```

## Change root (chroot) into the New System
```
# artix-chroot /mnt /bin/bash
```

## Configure mkinitcpio for LUKS Encryption
```
vim /etc/mkinitcpio.conf
```
Find the uncommented ```HOOKS=(...)``` and add ```encrypt``` before ```filesystems```:
```
# HOOKS=(
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
* ```q``` →
* ```g``` →

## Set the Time Zone:
```
# ln -sf /usr/share/zoneinfo/Canada/Pacific /etc/localtime
```

## Run hwclock to Generate /etc/adjtime:
```
# hwclock --systohc
```
* ```systohc``` →

## Uncomment en_US.UTF-8 UTF-8 in /etc/locale.gen
```
# vim /etc/locale.gen
```
* Generate new location:
```
# locale-gen
```
*Set the system locale:
```
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```
* Check if it worked:
```
# locale -a
```

rEFInd:
pacman -S refind
-First, copy the executable to the ESP (EFI System Partition):
# mkdir -p /boot/EFI/refind
-p => parents, where it creates the parent directories as needed
# cp /usr/share/refind/refind_x64.efi /boot/EFI/refind/
-Then use efibootmgr to create a boot entry in the UEFI NVRAM, where /dev/sdX and Y are the device and partition number of your EFI system partition. 
# efibootmgr --create --disk /dev/sdX --part Y --loader /EFI/refind/refind_x64.efi --label "rEFInd Boot Manager" --verbose
-Make sure from the output that the boot entry 'rEFInd Boot Manager' was created
-Now rEFInd should have a boot entry for your kernel, but it will not pass the correct kernel parameters. 
-Copy the config file to /boot/EFI/refind/refind.conf:
# cp /usr/share/refind/refind.conf-sample /boot/EFI/refind/refind.conf
-Change 'timeout' to -1 seconds to automatically start the kernel, and uncomment 'textonly' to remove the pointless icons. You should now be able to boot your kernel using rEFInd
-The rEFInd package includes the refind-install script to simplify the process of setting rEFInd as your default EFI boot entry. The script has several options for handling differing setups and UEFI implementations. See refind-install(8) or read the comments in the install script for explanations of the various installation options. For many systems it should be sufficient to simply run:
-I run this after just to make sure it all works
# refind-install
This will attempt to find and mount your ESP, copy rEFInd files to esp/EFI/refind/, and use efibootmgr to make rEFInd the default EFI boot application.


## Configure rEFInd Boot Options
```
# blkid -s UUID -o value /dev/sda3 >> /boot/refind_linux.conf
```
* Add the following line to the top of /boot/refind_linux.conf using the copied written UUID:
```
"Boot with LUKS" "cryptdevice=UUID=abcd1234-5678-efgh:cryptroot root=/dev/mapper/cryptroot rw"
```

Get most recent microcode for processor
#pacman -S intel-ucode
OR
#pacman -S amd-ucode

runit:
-need to create /run/runit/service/ as this is where artix stores its service directory 
enabling service: $ sudo ln -s /etc/runis/sv/service_name /run/runit/service/
disable service for next boot: $ sudo rm /run/runit/service/
stop service (still starts on next boot): $ sudo sv down service_name
start service: $ sudo sv up service_name
restart service: $ sudo sv restart service_name
service status: $ sudo status service name
List all running services: $ sudo sv status /run/runit/service/*

-Network Manager:
networkmanager networkmanager-runit 
-If you don't get wifi when rebooted, sometimes you have to do the link command below again
pacman -S networkmanager networkmanager-runit
-Autostart runit scripts:
# ln -s /etc/runit/sv/NetworkManager/ /run/runit/service/

-Set hostname
echo hostname > /etc/hostname

root passwd:
# passwd

Add user:
# useradd -m -g wheel user_name
# passwd user_name 

Add user to sudoers:
# vim /etc/sudoers
root ALL=(ALL:ALL) ALL
user_name ALL=(ALL:ALL) ALL

pacman -Syu 

Now, you can reboot and enter into your new installation:
<- exit chroot environment 
# exit                           
# umount -R /mnt
# reboot

-Install xorg:
$ sudo pacman -S xorg-server xorg-xinit xorg-xset
-Install i3wm:
$ sudo pacman -S i3 (choose i3-wm, i3status and i3lock) 
-Install a font and terminal manager (urxvt default not 256 colours it seems):
$ sudo pacman -S ttf-dejavu rxvt-unicode

Dotfiles
.xinitrc
exec i3

-Install man and git:
$ sudo pacman -S man git

Install firefox
$ sudo pacman -S firefox

ufw
sudo pacman -S ufw ufw-runit
sudo ln -s /etc/runit/sv/ufw /run/runit/service/
$ sudo ufw default allow outgoing
$ sudo ufw default deny incoming
$ sudo ufw enable
to check use
$ sudo ufw status verbose
make sure it's active

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

as of june 2021 arch repositories removed by default. Install 
sudo pacman -S artix-linux-support
follow the instructions to add [extra] [community] and [multilib] to /etc/pacman.conf

to stop the 'zoom' udev error on login screen:
copy /lib/udev/hwdb.d/60-keyboard.hwdb to /etc/udev/hwdb.d/
comment out the following lines with 'zoom' on them:
683, 689, 703, 731, 749, 1456       (880 and 881 995 test it)
next:
$ sudo udevadm hwdb --update
$ sudo udevadm control --reload-rules
$ sudo udevadm trigger

Change ownership:                                              
chown -R user:group directory/
-R is recursively, can't use lower case r for some reason
use 'groups user' or 'groups' as the user to establish what groups they're apart of
chmod -R 777 directory/
-chmod needs -R too for directories

pacman modifications:
-Pacman has a color option. Uncomment the 'Color' line in /etc/pacman.conf.
-Also uncomment 'CheckSpace' and 'VerbosePkgLists'  





-Auto Mount USB Stick: 
```bash
# /dev/sdb1 UUID=11834887-d9d9-4f74-b7b8-28428eb9f93a   /mnt/usb    ext4        defaults    0 0
```
-<dump> is checked by the dump(8) utility. This field is usually set to 0, which disables the check. 
-<fsck> sets the order for file system checks at boot time; see fsck(8). For the root device it should be 1. For other partitions it should be 2, or 0 to disable checking.



nginx:
$ sudo pacman -S nginx nginx-runit 
-It's free open-source HTTP server/reverse proxy with low resource consumption
-The default page served at http://127.0.0.1 is /usr/share/nginx/html/index.html
-Main config file is /etc/nginx/nginx.conf
-nginx has one master process and several worker processes, and the master process reads/evaluates configuration and maintains worker processes. Worker processes do actual processing of requests. nginx uses event-based model and OS-dependant mechanisms to distribute requests among worker processes
-Must define number of worker processes as either fixed or automatically adjusted to the number of CPU cores
-To start nginx, run executable file as sudo
$ sudo nginx
-Controls below should be implemented by same user that initiated nginx
-Can be controlled with -s or signal flag:
-s stop = fast shutdown
-s quit = graceful shutdown allowing worker processes to finish serving current requests
-s reload = reload config file
-s reopen = reopening the log files
-Once the master process (started by nginx init script) receives the signal to reload configuration, it checks the syntax validity of the new configuration file and tries to apply the configuration provided in it. If this is a success, the master process starts new worker processes and sends messages to old worker processes, requesting them to shut down. Otherwise, the master process rolls back the changes and continues to work with the old configuration. Old worker processes, receiving a command to shut down, stop accepting new connections and continue to service current requests until all such requests are serviced. After that, the old worker processes exit. 
-By default running on port 80 accessed by 127.0.0.1 or 'http://localhost'
-You will get file permission error trying to change nginx directory out of /usr/share/nginx and you should instead copy over the Website/ directory into /usr/share/nginx/html/ and it will be owned by root then and work properly, also you only have the 'working' version there so you can work on the other version without worry
-Only root processes can listen to ports below 1024 like 80/443 so this is why nginx must be started as root permissions (with sudo)
-nginx config file has modules controlled by directives, which are divided into simple directives ending with ; and block directives surrounded by {}
-Serving files requires a server block (or multiple) inside of an http block with at least one location block
http {
    server {
        location / {
            root /usr/share/nginx/html/
            index index.html
    server { 
        location /images/ {
            root /usr/share/nginx/html/images
            
    }
}
-After deciding which server to process request, nginx tests the URI specified in the request's header against parameters of location directive defined in server block
-URI is identifier of specific resource and URL is a specific type of identifier telling you how to access it (e.g. HTTPS, FTP) such as https://www.website.com. A URL is a URI but not all URIS are URLs
-The location block specifies '/' prefix compared with the URI from the request, and for matching request the URI will be added to the path specified inthe root directive (/usr/share/nginx/html/) to form the path to the requested file on the local file system
-In above example if request for http://localhost/images/example.png nginx will send the file, and if it doesn't exist nginx will send response indicating 404 error. Requests with URIS not stating /images/ will be mapped to the /sur/share/nginx/html/ directory


rsync and cronie and cronie-runit
$ sudo pacman -S rsync cronie cronie-runit
$ sudo vim /etc/cron.hourly/rsync
#!/bin/bash
rsync -ahvP --delete /home/steve/ /mnt/usb/rsync/
$ sudo chmod +x /etc/cron.hourly/rsync
$ ln -s /

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
-If you get permission denied while trying to edit it you may have created a reference to itself, try going in the directory you're linking to and/or using full directory paths

-Official Artix Wiki Installation: https://wiki.artixlinux.org/Main/Installation
-Use 'man package' for package manuals

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

-You can uninstall AUR packages normally, 
$ sudo pacman -Rns 'package_name'

Updating:
****Make update command chain of 
pacman -Syu
-Had issue with 'corrupted or invalid pgp signature' running archlinux-keyring, so refresh keys first
$ sudo pacman-key --refresh-keys
$ sudo pacman -S archlinux-keyring
$ sudo pacman -Syu
$ sudo pacdiff

Before first update:
pacman -S archlinux-keyring
-To go through .pacnew files after updating:
sudo pacman -S pacman-contrib
-To run:
sudo pacdiff

windows/linux fs transfers
sudo fdisk with 'o' at the start for dos
sudo pacman -S dosfstools
sudo mkfs.msdos -F 32 /dev/sdXX

You should be able to get dos2unix from your package manager on Linux.
If you are using a Debian-based distro, you should be able to do 
sudo pacman -S dos2unix
If you are using a RH-like distro, you should be able to do sudo yum install dos2unix.
Once it is installed, you can just give the target file as an argument'
dos2unix test.py
Also, note that this may not be the only problem you might run into while trying to move a script to Linux from Windows.
For example, if you are invoking any external tools in your script, those tools will probably have different names or not exist at all on the other platform.
Also, if you are using any relative file paths with path separators, the separator is different on Linux (which uses /) than Windows (which uses \).

interactive
alias rm='rm -i'
alias mv='mv -i'
alias cp='cp -i'

-pacman:
-List orphan programs (dependencies of deleted program)
$ sudo pacman -Qtdq
-Recursively delete orphans:
$ sudo pacman -Rns $(pacman -Qtdq)
-To install a single package or list of packages, including dependencies, issue the following command:
# pacman -S package_name1 package_name2 ...
-To remove a package and its dependencies which are not required by any other installed p ackage: 
pacman -Rs package_name
-Pacman can update all packages on the system with just one command.The following command synchronizes the repository databases and updates the system's packages, excluding "local" packages that are not in the configured repositories:
# pacman -Syu
-List all explicitly installed packages: 
pacman -Qe.






