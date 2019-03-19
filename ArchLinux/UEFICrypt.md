# Arch Linux Installation on UEFI Systems with Disk Encryption

## Arch Installation

1. Boot (UEFI) from the installation media
	
1. Optionally, for an easier (because copy and paste) remote installation...

	1. Set root password

			passwd

	1. Setup SSH

			systemctl start sshd.service

	1. Find IP

			ip addr

	1. Log in via SSH from a remote workstation to continue

1. Clean and partition the disk

		sgdisk --zap-all /dev/nvme0n1
		cgdisk /dev/nvme0n1

	* **EFI System Partition** - ef00 (550 MiB)
	* **LUKS Partition** - 8300

1. Setup encryption and LUKS

		cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
		cryptsetup luksOpen /dev/nvme0n1p2 luks
		pvcreate /dev/mapper/luks
		vgcreate vg0 /dev/mapper/luks
		lvcreate -L 24G vg0 -n swap
		lvcreate -l +100%FREE vg0 -n root

1. Format partitions
	- EFI

			mkfs.vfat -F32 -n "EFI System Partition" /dev/nvme0n1p1

	- Root

			mkfs.ext4 -L "Arch Linux Root" /dev/mapper/vg0-root

	- Swap

			mkswap -L "Arch Linux Swap" /dev/mapper/vg0-swap

2. Mount the partitions
	- Root

			mount /dev/mapper/vg0-root /mnt

	- EFI

			mkdir -p /mnt/boot
			mount /dev/nvme0n1p1 /mnt/boot

	- Swap

			swapon /dev/mapper/vg0-swap

1. Install the base system

		pacstrap /mnt base base-devel intel-ucode efibootmgr dosfstools networkmanager openssh net-tools bind-tools sudo wget git vim tmux zsh
		
	Optionally, append the following packages. 
	- GNOME
	
			gdm gnome gnome-power-manager gnome-tweaks aspell-en

2. Configure the system (see [ArchWiki](https://wiki.archlinux.org/index.php/Installation_Guide#Configure_the_system))
	- Generate fstab

			genfstab -U /mnt | awk '($1!~"^#"&&($2~"^/"||$3~"^swap$")){$4=$4",discard"}1' OFS='\t' >> /mnt/etc/fstab

	- Change root

			arch-chroot /mnt

	- Set time zone

			ln -sf /usr/share/zoneinfo/US/Central /etc/localtime
			hwclock --systohc

	- Configure locale

			sed -i.orig -e 's/^#\(en_US\.UTF-8 .*$\)/\1/g' /etc/locale.gen
			locale-gen
			echo "LANG=en_US.UTF-8" > /etc/locale.conf
			localectl set-locale LANG=en_US.UTF-8

	- Set hostname

			MyHostName=arch; printf "127.0.0.1\tlocalhost\n::1\t\tlocalhost\n127.0.1.1\t${MyHostName}.localdomain ${MyHostName}\n" >> /etc/hosts; echo "${MyHostName}" > /etc/hostname

	- Add encryption support to mkinitcpio
	
			sed -i.orig -e 's/^\(HOOKS=.* block \)\(.*\)$/\1encrypt lvm2 \2/g' /etc/mkinitcpio.conf

	- Create init RAM disk

			mkinitcpio -p linux

	- Set root password

			passwd

	- Install systemd-boot to the ESP and EFI variables

			bootctl --path=/boot install
			UUID=$(dumpe2fs -h $(grep " / " /etc/mtab | grep -v "rootfs\|arch_root-image" | awk '{ print $1 }') | grep "Filesystem UUID" | awk '{ print $3 }'); printf "title\tArch Linux\nlinux\t/vmlinuz-linux\ninitrd\t/intel-ucode.img\ninitrd\t/initramfs-linux.img\noptions\tcryptdevice=UUID=${UUID}:vg0:allow-discards root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap rw\n" > /boot/loader/entries/arch.conf

	- Enable (periodic) TRIM

			systemctl enable fstrim.timer

	- Enable Network Manager

			systemctl enable NetworkManager.service

	- Enable SSH

			systemctl enable sshd.service

	- Enable timesyncd

			systemctl enable systemd-timesyncd.service

	- Add user

			useradd -m -g users -G wheel -s /bin/zsh uncon
			chfn uncon
			passwd uncon

	- Configure sudo

			visudo

		Uncomment the following line.

			## Uncomment to allow members of group wheel to execute any command
			%wheel ALL=(ALL) ALL

	- Enable GDM (Optional)

			sudo systemctl enable gdm.service

	- Exit

			exit

1. Unmount and Reboot

		umount -R /mnt
		systemctl reboot

## Post-Installation

1. Install CUPS

		sudo pacman -Sy cups cups-pdf system-config-printer foomatic-db-engine foomatic-db foomatic-db-ppds foomatic-db-nonfree-ppds foomatic-db-gutenprint-ppds
		sudo systemctl enable --now org.cups.cupsd.service

1. Install [yay](https://github.com/Jguer/yay)

		mkdir ~/aur
		cd ~/aur
		PKG="yay" && git clone "https://aur.archlinux.org/${PKG}.git/" && cd "${PKG}" && makepkg -i -s -r --skippgpcheck

1. Install VMware Tools

		sudo pacman -Sy open-vm-tools
		sudo systemctl enable --now vmtoolsd.service

1. Install Google Chrome

		yay -Sy google-chrome

1. Install [tlp](https://wiki.archlinux.org/index.php/TLP)

		sudo pacman -Sy tlp x86_energy_perf_policy smartmontools ethtool
		sudo systemctl enable --now tlp.service
		sudo systemctl enable --now tlp-sleep.service

1. Install [Insync](https://www.insynchq.com/)

		yay -Sy insync insync-nautilus

1. Install [u2f-hidraw-policy](https://github.com/amluto/u2f-hidraw-policy)

		yay -Sy u2f-hidraw-policy