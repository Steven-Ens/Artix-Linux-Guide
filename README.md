# Artix Linux Guide
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
    * ```Safe Boot: Disabled```

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
# basestrap /mnt base base-devel runit elogind-runit linux-lts linux-firmware cryptsetup vim man git
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
$ sudo pacman -S kitty 
```

## Install the System Font
```
$ sudo pacman -S ttf-dejavu
```

## Auto Numlock On Boot
* Install ```numlockx``` and add it to ```~/.xinitrc``` before ```exec i3```:
```
$ sudo pacman -S numlockx
```

## Keyboard Brightness
* Install:
```
$ sudo pacman -S brightnessctl
```
* Add user to the video group so ```brightnessctl``` can be used without sudo:
```
$ sudo usermod -a -G video steve
```
* ```a``` → Append group keeping existing groups.
* ```G``` → Add supplementary groups

## Audio
* Install:
```
$ sudo pacman -S pipewire pipewire-pulse wireplumber
```

## Screen Resolution
* Install:
```
$ sudo pacman -S xorg-xrandr
```
* Show available displays and resolutions:
```
$ xrandr
```
* Set a resolution and refresh rate:
```
$ xrandr --output eDP-1 --mode 1920x1080 --rate 60
```

## Install zip & unzip
```
$ sudo pacman -S zip unzip
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

## Install Firefox
* Select the ```pipewire-jack``` provider:
```
$ sudo pacman -S firefox pipewire-jack
```
* Prevent the window from auto hiding in fullscreen:
* Open new tab and type ```about:config``` 
* Double click to set to ```False```

## Install feh
```
$ sudo pacman -S feh
```
* Open single file:
```
$ feh <file>
```
* Open all files in directory:
```
$ feh
```

## Install mpv
```
$ sudo pacman -S mpv
```
* Run:
```
$ mpv <file>
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
* Run the two installation scripts in Dotfiles/scripts:
```
$ sudo ./scripts/install_dotfiles.sh
$ ./scripts/install_vim_plugins.sh 
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

## Install Solidity LSP for coc.nvim
```
$ npm install -g @nomicfoundation/solidity-language-server
```

## Install solhint
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
* Confirm ufw is running:
```
$ sudo sv status ufw
$ sudo ufw status verbose
```

## Static IP Address
* Show active connections:
	* Ethernet uses ```Wired connection 1``` by default.
	* Wireless uses the network name (SSID).
```
$ nmcli con show
```
* Find the default gateway:
```
$ ip r | grep default
```
* Set the default gateway:
```
$ nmcli con mod "Wired connection 1" ipv4.gateway 192.168.0.1
```
* Set the IP address:
```
$ nmcli con mod "Wired connection 1" ipv4.addresses 192.168.0.X/24
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
* Download the configuration file:
```
$ sudo curl -o openvpn.zip https://www.privateinternetaccess.com/openvpn/openvpn-nextgen.zip
```
* Extract the downloaded file to ```client/```:
```
$ sudo unzip openvpn.zip -d /etc/openvpn/client/
```
* Create ```/etc/openvpn/login.txt``` with the format:
```username```
```password```
* Secure ```login.txt``` so only root can access the stored username and password:
```
sudo chown root:root /etc/openvpn/login.txt
sudo chmod 600 /etc/openvpn/login.txt
```
* Test connection:
```
$ sudo openvpn --config /etc/openvpn/client/ca_vancouver.ovpn --auth-nocache --auth-user-pass /etc/openvpn/login.txt'
```
* ```--auth-nocache``` Prevents OpenVPN from retaining the username and password in memory after authentication.

## OpenSSH
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
$ sv status sshd
```
* Allow ssh in ufw:
```
$ sudo ufw allow ssh
```
* Verify:
```
$ sudo ufw status
```
* Connect from another machine:
```
$ ssh steve@192.168.0.X
```

## SSH Key Creation
* Generate a key pair on the client machine:
```
$ ssh-keygen -t ed25519 -a 100
```
* ```-t ed25519``` → Edwards-curve Digital Signature Algorithm
* ```-a 100``` → Specifies the number of Key Derivation Function (KDF) rounds used to protect the private key if you set a passphrase.
* Accept the default location ```~/.ssh/id_ed25519```.
* Copy the public key to the server:
```
$ ssh-copy-id steve@192.168.0.X
```
* Verify logging in without entering the account password:
```
$ ssh steve@192.168.0.X
```
* Disable password authentication:
```
$ sudo vim /etc/ssh/sshd_config
```
* Change:
* ```PasswordAuthentication no```
* ```PubkeyAuthentication yes```
* ```PermitRootLogin no```
* ```AllowUsers steve```
* Restart sshd:
```
$ sudo sv restart sshd
```

## Auto Mount USB:
* Find the UUID of the USB drive:
```
$ lsblk -f
```
* Add the following line to ```/etc/fstab``` using the device UUID:
```
# /dev/sdb1
UUID=<UUID>  /mnt/usb  ext4  nofail  0 2
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
$ sudo mkdir /mnt/usb/
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

# System Usage

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
* Use CTL-Shift-c and CTL-Shift-p to copy and paste between vim and the terminal
* Use CTL-c to copy inside of firefox, then use CTL-Shift-V to paste to vim and the terminal

## Create a zip
```
$ zip archive.zip <file>
```
* zip a directory recursively:
```
$ zip -r archive.zip <directory>/
```

## Extract a zip
```
$ unzip archive.zip
```
* Extract to a specific directory:
```
$ unzip archive.zip -d <directory>/
```

## scp
* Copy file from remote host to local host
$ scp username@from_host:file.txt /local/directory/
-Copy file from local host to remote host
$ scp file.txt username@to_host:/remote/directory/
-Copy directory from remote host to local host
$ scp -r username@from_host:/remote/directory /local/directory
-Copy directory from local host to remote host
$ scp -r /local/directory/ username@to_host:/remote/directory/

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

## Using pacman:
* Remove a package and its dependencies which are no longer required:
```
# pacman -Rns <package>
```
* ```R``` → Remove package.
* ```n``` → Remove backup configuration files.
* ```s``` → Remove unneeded dependencies.
* List all explicitly installed packages:
```
$ sudo pacman -Qe
```
* ```Q``` → Query the local package database.
* ```e``` → Show explicitly installed packages only.
* List orphaned programs:
```
$ sudo pacman -Qdtq
```
* ```Q``` → Query the local package database.
* ```d``` → Restrict to packages installed as dependencies.
* ```t``` → Restrict to unrequired packages.
* ```q``` → Show package names only.
* Recursively delete orphans:
```
$ sudo pacman -Rns $(pacman -Qdtq)
```

## Troubleshooting pacman
* If you see errors related to the package signing keyring, refresh the keys:
```
$ sudo pacman-key --refresh-keys
```
