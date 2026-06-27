# Artix Linux Guide
* Official Installation Guide: https://wiki.artixlinux.org/Main/Installation

## What This Guide Provides
* This guide implements the following:
	* GPT + UEFI
	* rEFInd
	* runit
	* LUKS
   	* i3wm

## Verify the ISO Checksum
```
$ sha256sum artix-base-runit-<version>-x86_64.iso
```
* Compare the output against the SHA256 checksum listed on the Artix download page: https://artixlinux.org/download.php.
  
## Verify the ISO Signature
```
$ gpg --auto-key-retrieve --verify artix-base-runit-<version>-x86_64.iso.sig artix-base-runit-<version>-x86_64.iso
```
* Make sure the RSA key is the key specified on the Artix website under **Official ISO Images** (eg. [```0xB886B428```](https://pgpkeys.eu:11371/pks/lookup?search=0xA574A1915CEDE31A3BFF5A68606520ACB886B428&fingerprint=on&op=index)).
* You may see:
```
WARNING: This key is not certified with a trusted signature!
```
* This warning is normal on a fresh system. It means the signing key is not yet trusted in your local GPG keyring through GPG’s web-of-trust model.

## Prepare the Installation Medium
* Make sure that the USB is not mounted.
* Run the following command, replacing ```/dev/sdX``` (or ```/dev/nvmeXn1```) with your drive:
```
$ sudo dd if=~/artix-base-runit-<version>-x86_64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```
* ```if``` → Input file
* ```of``` → Output file
* ```bs``` → Writes in 4 MiB chunks
* ```status=progress``` → Shows transfer progress
* ```conv=fsync``` → Flushes data before exiting

## Verify UEFI Boot Mode
```
# cat /sys/firmware/efi/fw_platform_size
```
* If the command returns 64, the system is booted in UEFI mode and has a 64-bit x64 UEFI.
* If it returns ```No such file or directory```, the system may be booted in BIOS or CSM mode.

## ThinkPad T440 UEFI Troubleshooting
* On the ThinkPad T440, ```UEFI Only``` mode may fail to boot Linux installation media even when the USB is properly configured for UEFI.
* The following worked:
	* Reset the BIOS to its default.
   	* Set the installation media as the first boot device.
	* Boot options:
		* ```Boot Mode: Both```
		* ```Boot Priority: UEFI First```
		* ```CSM Support: Yes``` (Automatic)
    	* ```Safe Boot: Disabled```

## Partition the Disk
* List the block devices and identify the target drive ```/dev/sdX```:
```
# lsblk
```
* For a UEFI system, use a GPT partition table with the following layout: 
	* /dev/sdX1 for EFI System (1GB)
	* /dev/sdX2 for Linux swap (\<system ram\>GB)
	* /dev/sdX3 for Linux filesystem (\<remaining space\>GB)

* Open fdisk on the target drive:
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
* Mirrors are defined in ```/etc/pacman.d/mirrorlist```, ensure the ones on the top are closest to you geographically:
```
# vim /etc/pacman.d/mirrorlist
```

## Install the Base System
```
# basestrap /mnt base base-devel runit elogind-runit linux-lts linux-firmware cryptsetup vim man git
```

## Generate the Filesystem Table
```
# fstabgen -U /mnt >> /mnt/etc/fstab
```
* ```-U``` → Use UUIDs instead of device names.
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
* Result:
```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap block encrypt filesystems fsck)
```
* Rebuild initramfs:
```
# mkinitcpio -P
```
* ```-P``` → Build all presets.
* Safe to ignore ```WARNING: Possibly missing firmware for module: 'qat_6000'``` on systems without Intel QuickAssist (QAT) hardware.

## Update the System Clock
```
# pacman -S ntp
# ntpd -qg
```
* ```-q``` → Quit after syncing once.
* ```-g``` → Allow a large initial time correction.

## Set the Time Zone:
```
# ln -sf /usr/share/zoneinfo/<country>/<region> /etc/localtime
```

## Run hwclock to Generate /etc/adjtime:
```
# hwclock --systohc
```
* ```--systohc``` → System clock to hardware clock.

## Localization
* Uncomment ```en_US.UTF-8 UTF-8``` and ```en_US ISO-8859-1``` in ```/etc/locale.gen```:
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
* Confirm by listing all locales:
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
* Configure ```refind_linux.conf``` to pass the correct LUKS boot options, starting with the UUID of ```/dev/sdX3```:
```
# blkid -s UUID -o value /dev/sdX3 > /boot/refind_linux.conf
```
* Edit the file:
```
# vim refind_linux.conf
```
* Add the following line to the top using the UUID copied into the file:
```
"Boot with LUKS" "cryptdevice=UUID=<UUID>:cryptroot root=/dev/mapper/cryptroot rw"
```

## Install networkmanager
```
# pacman -S networkmanager networkmanager-runit
```
* Autostart with runit (```/run``` is tmpfs filesystem and is created at boot, so use ```/etc/runit/runsvdir/default/``` during installation):
```
# ln -s /etc/runit/sv/NetworkManager /etc/runit/runsvdir/default/
```

## Set hostname
```
# echo <hostname> > /etc/hostname
```

## Set root Password
```
# passwd
```

## Add New User 
* Add user to the wheel group:
```
# useradd -mG wheel <user>
```
* ```-m``` → Create home directory
* ```-G``` → Add supplementary groups
* Set user password:
```
# passwd <user>
``` 

## Add User to sudoers
* Give wheel group sudo permissions:
```
# vim /etc/sudoers
```
* Uncomment: ```%wheel ALL=(ALL:ALL) ALL```

## Reboot
* Exit the chroot environment:
```
# exit
```
* Recursively unmount ```/mnt```:
```                         
# umount -R /mnt
```
* Reboot, noting that wireless connections will need to be reactivated:
```
# reboot
```

## Install the Terminal & System Font
```
$ sudo pacman -S kitty ttf-dejavu
```

## Install numlockx
```
$ sudo pacman -S numlockx
```

## Install brightnessctl
```
$ sudo pacman -S brightnessctl
```
* Add user to the video group so ```brightnessctl``` can be used without sudo:
```
$ sudo usermod -aG video <user>
```
* ```-a``` → Append group keeping existing groups
* ```-G``` → Add supplementary groups

## Install pipewire
```
$ sudo pacman -S pipewire pipewire-pulse wireplumber
```

## Install zip & unzip
```
$ sudo pacman -S zip unzip
```

## Install xorg:
```
$ sudo pacman -S xorg-server xorg-xinit xorg-xset xorg-xrandr
```

## Install i3wm:
* Select ```i3-wm```, ```i3status``` and ```i3lock```:
```
$ sudo pacman -S i3 
```

## Install Firefox
* Select the ```pipewire-jack``` provider:
```
$ sudo pacman -S firefox pipewire-jack
```
* Prevent the window from auto hiding in fullscreen:
* Open new tab and type ```about:config```
* Type the following in the search box:
```
browser.fullscreen.autohide
```
* Double click to set to ```False```.

## Install feh
```
$ sudo pacman -S feh
```

## Install mpv
```
$ sudo pacman -S mpv
```

## Install Foundry
```
$ curl -L https://foundry.paradigm.xyz | bash
```
* Reload shell:
```
$ source ~/.bashrc
```
* Install Foundry binaries:
```
$ foundryup
```

## Install Dotfiles
```
$ git clone https://github.com/Steven-Ens/Dotfiles
```
* Go to ```scripts/```:
```
$ cd ~/Dotfiles/scripts/
```
* Run the following installation scripts:
```
$ sudo ./install_dotfiles.sh
$ ./install_vim_plugins.sh 
```
* Reboot:
```
$ sudo reboot
```

## Install nvm, node and npm
* Install NVM:
```
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
```
* Reload shell:
```
$ source ~/.bashrc
```
* Install Node.js + npm:
```
$ nvm install --lts
```
* Verify:
```
$ node -v
$ npm -v
```

## Install Solidity LSP Globally for coc.nvim
```
$ npm install -g @nomicfoundation/solidity-language-server
```

## Install solhint Globally
```
$ npm install -g solhint
```

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
* Confirm ufw is active and running:
```
$ sudo sv status ufw
$ sudo ufw status verbose
```

## Static IP Address
* Show active connections:
	* Ethernet uses ```Wired connection 1``` by default.
	* Wireless uses the network name \<SSID\>.
```
$ nmcli con show
```
* Set the IP address:
```
$ nmcli con mod "Wired connection 1" ipv4.addresses <ip address>/24
```
* Find the default gateway:
```
$ ip r | grep default
```
* Set the default gateway:
```
$ nmcli con mod "Wired connection 1" ipv4.gateway <ip address>
```
* Set the DNS servers to Cloudflare:
```
$ nmcli con mod "Wired connection 1" ipv4.dns "1.1.1.1 1.0.0.1"
```
* Change from DHCP to a static IP:
```
$ nmcli con mod "Wired connection 1" ipv4.method manual
```
* Disable IPv6 for PIA VPN:
```
$ nmcli con mod "Wired connection 1" ipv6.method disabled
```
* Reload the connection:
```
$ nmcli con up "Wired connection 1"
```

## Private Internet Access VPN
* Install OpenVPN:
```
$ sudo pacman -S openvpn
```
* Change to the OpenVPN directory:
```
$ cd /etc/openvpn
```
* Download the PIA configuration files for UDP Strong at: https://www.privateinternetaccess.com/openvpn/openvpn-strong.zip
* Extract the downloaded file to ```client/```:
```
$ sudo unzip openvpn-strong.zip -d client/
```
* Create ```login.txt```:
```
$ sudo vim login.txt
```
* Copy your PIA username and password in the format:
* ```username```
* ```password```
* Secure ```login.txt``` so only root can access the stored username and password:
```
sudo chmod 600 login.txt
```
* Confirm:
```
$ ls
```
* Test connection:
```
$ sudo openvpn --config /etc/openvpn/client/ca_vancouver.ovpn --auth-nocache --auth-user-pass /etc/openvpn/login.txt'
```
* ```--auth-nocache``` Prevents OpenVPN from retaining the username and password in memory after authentication.

## OpenSSH Server
* Install:
```
$ sudo pacman -S openssh openssh-runit
```
* Enable and start the service:
```
$ sudo ln -s /etc/runit/sv/sshd /run/runit/service/
```
* Verify:
```
$ sudo sv status sshd
```
* Allow ssh in ufw:
```
$ sudo ufw allow ssh
```
* Verify:
```
$ sudo ufw status
```
* Connect from client machine:
```
$ ssh <user>@<ip address>
```

## SSH Key Creation
* Generate a key pair on the client machine:
```
$ ssh-keygen -t ed25519 -a 100
```
* ```-t ed25519``` → Edwards-Curve Digital Signature Algorithm.
* ```-a 100``` → Specifies the number of Key Derivation Function (KDF) rounds used to protect the private key if you set a passphrase.
* Accept the default location ```~/.ssh/id_ed25519```.
* Copy the public key to the server:
```
$ ssh-copy-id <user>@<ip address>
```
* Verify logging in without entering the account password:
```
$ ssh <user>@<ip address>
```
* Harden sshd on the server:
```
$ sudo vim /etc/ssh/sshd_config
```
* Add under ```# Authentication```:
  	* ```AllowUsers <user>```
* Modify to:
  	* ```PermitRootLogin no```
	* ```PasswordAuthentication no```
* Restart sshd:
```
$ sudo sv restart sshd
```

## Auto Mount USB:
* Find the UUID of the USB drive:
```
$ lsblk -f
```
* Create a separate mount point for USB devices ```/mnt/usb```:
```
$ sudo mkdir /mnt/usb/
```
* Add the following line to ```/etc/fstab``` using the device UUID:
```
# /dev/sdb1
UUID=<UUID>  /mnt/usb  ext4  nofail  0 2
```
* ```nofail``` → Continue booting even if the USB drive is disconnected.
* ```dump``` → Legacy backup field used by the ```dump``` utility. Almost always set to ```0```.
* ```fsck``` → Sets filesystem check order at boot.
    * ```1``` → Root filesystem
    * ```2``` → Other filesystems
    * ```0``` → Disable checking

## Auto Backup /home/<user>
* Install:
```
$ sudo pacman -S rsync cronie cronie-runit
```
* Autostart the service:
```
$ sudo ln -s /etc/runit/sv/cronie /run/runit/service/
```
* Create the backup script:
```
$ sudo vim /etc/cron.hourly/backup
```
* Add:
```
#!/bin/bash
rsync -a --delete /home/<user>/ /mnt/usb/
```
* ```a``` → Archive mode that recursively preserves file attributes.
* ```delete``` → Removes files from the backup that no longer exist in the source directory, keeping an exact mirror.
* Make the script executable:
```
$ sudo chmod +x /etc/cron.hourly/backup
```
* Verify cronie is running:
```
$ sudo sv status cronie
```

# System Usage

## pacman:
* Remove a package and its dependencies which are no longer required:
```
$ sudo pacman -Rns <package>
```
* ```-R``` → Remove package.
* ```-n``` → Remove backup configuration files.
* ```-s``` → Remove unneeded dependencies.

* List all explicitly installed packages:
```
$ sudo pacman -Qe
```
* ```-Q``` → Query the local package database.
* ```-e``` → Show explicitly installed packages only.

## Troubleshooting pacman
* If you see errors related to the package signing keyring, refresh the keys:
```
$ sudo pacman-key --refresh-keys
```

## Using runit
* Enable and start a service in the current runlevel:
```
# ln -s /etc/runit/sv/<service> /run/runit/service/
```
* Disable and remove a service from the current runlevel:
```
# rm -f /run/runit/service/<service>
```
* Start a service:
```
# sv up <service>
```
* Stop a service:
```
# sv down <service>
```
* Restart a service:
```
# sv restart <service>
```
* Service status:
```
# sv status <service>
```
* List all running services:
```
# sv status /run/runit/service/*
```

## Copy & Paste
* Use ```CTL-Shift-c``` and ```CTL-Shift-p``` to copy and paste between vim and the terminal.
* Use ```CTL-c``` to copy inside of Firefox, then use ```CTL-Shift-p``` to paste into vim and the terminal.

## Create a zip
```
$ zip <file>.zip <file>
```
* zip a directory recursively:
```
$ zip -r <directory>.zip <directory>/
```

## Extract a zip
```
$ unzip <file>.zip
```
* Extract to a specific directory:
```
$ unzip <directory>.zip -d <directory>/
```

## scp
* Copy a file from a remote host to the local machine:
```
$ scp <remote-user>@<remote-host>:<remote-file> <local-directory>/
```
* Copy a directory from a remote host to the local machine:
```
$ scp -r <remote-user>@<remote-host>:<remote-directory> <local-directory>/
```
* Copy a file from the local machine to a remote host:
```
$ scp <local-file> <remote-user>@<remote-host>:<remote-directory>/
```
* Copy a directory from the local machine to a remote host:
```
$ scp -r <local-directory>/ <remote-user>@<remote-host>:<remote-directory>/
```

## feh
* Open a file:
```
$ feh <file>
```
* Open all files in the current directory:
```
$ feh
```

## mpv
* Open a file:
```
$ mpv <file>
```

## solhint
* In the root of each project repo add the file ```.solhint.json``` with the following configuration:
```
{
  "extends": "solhint:recommended"
}
```

# Before the First System Update

## pacman Output
* Uncomment the following lines in ```/etc/pacman.conf```:
* ```Color``` → Improves readability by highlighting package information and status messages.
* ```VerbosePkgLists``` → Displays installed and available package versions during upgrades, making changes easier to review.

## Install pacdiff
```
$ sudo pacman -S pacman-contrib
```

## Initialize Pacman Keys
```
$ sudo pacman-key --init
```
* ```init``` → Creates the local pacman GPG keyring used to verify package signatures.
```
$ sudo pacman-key --populate artix
```
* ```--populate``` → Imports the trusted Artix package signing keys into pacman's keyring.
* ```artix``` → Uses the Artix keyring only.

# System Maintenance

## Update the System
```
$ sudo pacman -Syu
```
* ```-S``` → Synchronize packages.
* ```-y``` → Refresh package database.
* ```-u``` → Upgrade all installed packages.

## List Orphaned Programs
```
$ sudo pacman -Qdtq
```
* ```-Q``` → Query the local package database.
* ```-d``` → Restrict to packages installed as dependencies.
* ```-t``` → Restrict to unrequired packages.
* ```-q``` → Show package names only.

## Recursively Delete Orphans
```
$ sudo pacman -Rns $(pacman -Qdtq)
```

## Review Conflicts
* Review differences between configuration files and their corresponding ```.pacnew``` files, then merge or remove them as needed:
```
$ sudo pacdiff
```

## Backup the EFI and Root Filesystems with fsarchiver (*** UNTESTED ***)
Install fsarchiver:
```bash
sudo pacman -S fsarchiver
```
Create a backup containing both the EFI partition and the encrypted root filesystem:
```bash
sudo fsarchiver savefs artix-backup.fsa \
    /dev/sda1 \
    /dev/mapper/cryptroot
```
View the filesystems stored in the archive:
```bash
fsarchiver archinfo artix-backup.fsa
```
Example output:
```text
Filesystem id=0: /dev/sda1
Filesystem id=1: /dev/mapper/cryptroot
```
Copy the archive to external storage for safekeeping.
## Restore the EFI and Root Filesystems
Boot from a live USB.
Recreate the GPT partition table and partitions if necessary:
```text
/dev/sda1  EFI System Partition
/dev/sda2  Swap
/dev/sda3  Linux partition for LUKS
```
Format the EFI partition:
```bash
mkfs.fat -F32 /dev/sda1
```
Recreate and open the LUKS container:
```bash
cryptsetup luksFormat /dev/sda3
cryptsetup open /dev/sda3 cryptroot
```
Restore the EFI partition:
```bash
sudo fsarchiver restfs artix-backup.fsa id=0,dest=/dev/sda1
```
Restore the root filesystem:
```bash
sudo fsarchiver restfs artix-backup.fsa id=1,dest=/dev/mapper/cryptroot
```
Recreate the swap partition:
```bash
mkswap /dev/sda2
```
Mount the restored system:
```bash
mount /dev/mapper/cryptroot /mnt
mount /dev/sda1 /mnt/boot
```
Reinstall or verify the bootloader if required.
