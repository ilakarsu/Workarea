
# WORK IN PROGRESS

# Preamble
This document was created with the purpose of keeping notes and aiding myself through the various types of ArchLnx installations. I have started out installing a very simple ArchLnx setup and then advanced to more enhanced configurations using BtrFS and encryption. Furthermore, I wan't to stress that the reader is encouraged and advised to do his own research. Again, these are my personal notes. I strongly suggest relying on official documentation when researching/googling. The likelyhood of running into guides that contain false information or outdated information is high.

Tips when looking for answers on google: make sure you contain keywords that will redirect you to official docs, such as 'manpage' or 'wiki'. Look for your answers in these documents first, always.

I have spend a great amount of time, looking at youtube videos and reading blogs, again, a lot of these are unreliable and it is even more misleading if you are new into this subject.

Last but not least, I hope this installation guide will help you out, feel free to make amendment requests.
# Arch Linux installation
Basic preperation tasks to successfully install our distro and continue with more advanced steps.

#### Keyboard layout
If necessary, load your keyboard layout as first step, e.g.:
	
	loadkeys de-latin1

#### Establish an internet connection
Will yield an ip address if everything is ok. Usually, this means we are good to go.

##### Ethernet
    ip a

if you we want to be really sure, we might also check our ping results:

	ping google.com 

##### WiFi
If we are not running a wired connection, we need to configure the wifi setup, by executing the following command:

	wifi-menu

#### Sync the network time protocol
With this command we synchronize our Arch Linux time with the current network time.

	timedatectl set-ntp true

#### Update package manager
Throughout the installation, we will continously be using our package manager 'pacman'. In order to be efficient and have up-to-date sources, we will first update it. And also, we want to make sure we are getting a fast connection to the physically closest server available.


	pacman -Syyy
	pacman -S reflector
	
	reflector -c Australia -a 6 --sort rate --save /etc/pacman.d/mirrorlist
	pacman -Syyy

#### Partitioning and Formatting the storage

This step is very important, it ll be the foundation of our installation.

First check what your current storage looks like, by running:

	lsblk

The output should be something like:

	TODO

For a basic installation, we need three partitions:

 1. Boot (500MB should suffice)
 2. SWAP (Personal preference, I usually go with 2-3GB)
 3. Root (Remainder of your available storage space)

However, since we are going to include the SWAP partition in our main partition, we will use two partitions only: boot and root. The latter will be encrypted. See graph below:

TODO

We will use fdisk, you can also achieve the same results using cfdisk or any other partitioning tool of your choice.

	fdisk /dev/[diskname]
	g
	n <enter> <enter> +500M t 1
	n <enter> <enter> <enter>
	w

Compare before after:

	lsblk

#### Encrypting our storage

*[Optional] Before we encrypt anything, we will run some commands and choose our cipher.
	
		modprobe dm-crypt
		cryptsetup benchmark*

The last command will run some default ciphers and test them. It will yield an overview of the test results. Pick the one, you feel is right for you. Note that we don't use any special options below, using the default parameters. E.g.:

	cryptsetup -v luksFormat /dev/[devicename]
	# which is equivalent to this:
	cryptsetup -v --type luks --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time 2000 --use-urandom --verify-passphrase luksFormat /dev/[devicename]

This will show a warning and we have to confirm by typing in yes in capital letters (important): 

	YES <enter>

Next, define a password for your encrypted storage.

	<password> <password> <enter>

To access our encrypted partition,  we must first open it. This procedure is done using the passphrase we have defined in the previous step. Our unlocked partition will be handled by the Kernel using the device mapper, in order to avoid confusion when writing into the partition. This basically alerts our kernel that the device will be available under its new mapped name::

	cryptsetup open /dev/[devicename] [mapped_name]
	cryptsetup open /dev/vda2 cryptroot

Now we have full access to our encrypted storage and can run all the commands as usual. For instance, try to ls and see the output.

#### Format the partitions
Notice how we are accessing our main partition by its mapper name, and not by the device name.

	mkfs.fat -F32 -n BOOT /dev/vda1
	mkfs.btrfs -L root /dev/mapper/cryptroot

 #### Handling the partitions

Since everything has been created and is ready for mounting, lets continue as follows:

	mount /dev/mapper/cryptroot /mnt
	mkdir /mnt/boot

##### Create BtrFS subvolumes
Ok, so far so good, our boot and root partitions are mounted. Our root partition is using BtrFS. Therefore, we will create some subvolumes now.

	# the @ indicates that we are at a top-level subvolume.
    btrfs su cr /mnt/@root
    btrfs su cr /mnt/@home
    btrfs su cr /mnt/@vlog
    btrfs su cr /mnt/@vche

Since we only want to keep snapshots of two subvolumes: root and home. We will only consider those by creating another folder where we store the snapshot images:

	mkdir /mnt/@home/live
	mkdir /mnt/@root/live

And these itself, will also be subvolumes in our filesystem (one of the main reasons being, if you create a snapshot of a subvolume in btrfs - its nested subvolumes will be ignored! ):

	# notice how we dont use @ as a prefix anymore, because these snapshots won't reside on the top-level.
	btrfs su cr /mnt/@root/live/snapshot
	btrfs su cr /mnt/@home/live/snapshot


##### Mount the BtrFS subvolumes and boot
Ok, we unmount our fs and will mount in the newly created subvolumes instead.

	umount /mnt

Lets set some variables for the sake of re-usability:

	btrfs_opt=noatime,compress=zstd,autodefrag #note: might use lzo for compression
	cryptroot_path=/dev/mapper/cryptroot

The mount commands below can be a bit confusing, what has helped me a lot is to understand what is happening, and how the mount command works. Look it up on the manpage for the full details. But basically, we are telling to mount our 'device' to our 'dir'. The latter being the root of our filesystem on our device. What is important to note here, by using the 'subvol' option, we override the 'device' (which is $cryptroot_path in our example).  

    # The live snapshot (with btrfs/snapper your live-environment is a snapshot too..). Therefore, this will be our main point of mounting root.
    mount -o $btrfs_opt,subvol=@root/live/snapshot $cryptroot_path /mnt
    
    # create subdirectories on which will be later referenced for mounting again
    mkdir -p /mnt/{boot,home,var/log,var/cache,.snapshots}
    
    # same as we did with our @root subvolume, mount the live snapshot on the toplevel
    mount -o $btrfs_opt,subvol=@home/live/snapshot $cryptroot_path /mnt/home
    
	# this might seem confusing at first, but basically we are telling our filesystem, that everything root-revelant resides in the .snapshot directory, including the live snapshot.
    mount -o $btrfs_opt,subvol=@root $cryptroot_path /mnt/.snapshots
	
	# have to do the same for our home subvolume
    mkdir -p /mnt/home/.snapshots

	# and mount it, same as we did with the root dir.
    mount -o $btrfs_opt,subvol=@home $cryptroot_path /mnt/home/.snapshots
    
    # we are not keeping any snapshots of our logs or cache (overkill) thus, mounting them on the toplevel is sufficient. Notice, we are not mounting any .snapshot directories.
    mount -o $btrfs_opt,subvol=@vlog $cryptroot_path /mnt/var/log
    mount -o noatime,subvol=@vche $cryptroot_path /mnt/var/cache #note, we dont use compression for the cache dir

	# mount boot directory
	mount /dev/vda1 /mnt/boot

##### SwapFile
Since we are using a btrfs and our swapfile will also reside in this partition, we need to follow some extra steps in order to have a fully functional swapfile:

	# btrfs specific swapfile creation
	cd /mnt/var/cache
	truncate -s 0 swapfile
	chattr +C swapfile
	btrfs property set swapfile compression none
	fallocate -l 2G swapfile
	chmod 600 swapfile
	mkswap swapfile
	swapon swapfile
	cd /


# Arch Linux configuration
In this section, we will focus on installing all the necessary software packages. The first sections focus was on getting our preperation tasks done, such as formatting the storage volumes and making sure we have the filesystems we want etc.

First, we will install the basic packages we need, this might differ from use case to use case:

    pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware inetutils intel-ucode nano wpa_supplicant wireless_tools networkmanager network-manager-applet --ignore linux


The installation will take a while, when it is finished, we need to generate a fstab file by executing:

	genfstab -U /mnt >> /mnt/etc/fstab

	# TODO: get rid of UUIDs to avoid confusion?
	nano /mnt/etc/fstab


Now, we are ready to enter our new system, most of these commands are self-explanatory, thus not going to comment much:

	arch-chroot /mnt

	# time and date configuration
	ln -s /usr/share/zoneinfo/Australia/Adelaide /etc/localtime

	# sync sys time with hw clock
	hwclock --systohc

	# Uncomment en_US.UTF-8	nano /etc/locale.gen
	locale-gen
	echo "LANG=en_US.UTF-8" >> /etc/locale.conf

	# set hostname (name of your 'machine')
	echo "[hostname]" >> /etc/hostname

	# set localhost ip's
	nano /etc/hosts
	# 127.0.0.1	localhost
	# ::1		localhost
	# 127.0.1.1	[hostname].localdomain	[hostname]

	# configure pw for root
	passwd
	<password> <enter> <password> <enter>

	pacman -S grub efibootmgr os-prober mtools dosfstools

	grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

	# add new user
	useradd -mG wheel [username]
	passwd [pw]

	# set the default editor to nano
	export EDITOR=nano
	
	# wrapper to run <default editor> /etc/sudoers command. Could also run manually: nano /etc/sudoers
	# allow all members of %wheel group to execute sudo commands
	visudo

	# Add 'encrypt' to HOOKS before filesystems
	# HOOKS="base udev autodetect modconf block encrypt filesystems keyboard fsck"
	nano /etc/mkinitcpio.conf

	# Regenerate initrd image
	mkinitcpio -p linux-zen

	# Setup grub
	nano /etc/default/grub
	# add GRUB_ENABLE_CRYPTODISK=y and GRUB_DISABLE_SUBMENU=y
	# add new line (allow-discards is for SSDs):
	GRUB_CMDLINE_LINUX: "cryptdevice=/dev/[encrypted-device (e.g.sda2)]:cryptroot:allow-discards root=/dev/mapper/cryptroot"

	# regenerate config file for grub
	grub-mkconfig -o /boot/grub/grub.cfg

	# enable some system services
	systemctl enable NetworkManager
	systemctl enable bluetooth
	systemctl enable orgs.cups.cupsd
	
	# exit, unmount everything and reboot
	exit
	umount -a
	reboot

	# Post Arch Install
	sudo su

	pacman -S reflector
	reflector -c Australia -a 6 --sort rate --save /etc/pacman.d/mirrorlist

	# Sync with network time
	pacman -S ntp
	systemctl enable ntpd.service
	systemctl start ntpd.service

	# Install snapper to take snapshots
	pacman -S snapper
	cd /etc/snapper/config-templates/
	cp default ../configs/root
	cp default ../configs/home
	# add list of config files to snapper-config seperated by spaces
	nano /etc/conf.d/snapper

	# start snapper services
	systemctl start snapper-timeline.timer
	systemctl enable snapper-timeline.timer
	systemctl start snapper-cleanup.timer
	systemctl enable snapper-cleanup.timer

# DE
In this example we will install the SSDM window manager and Plasma as our desktop environment:

	# first install some basic UI related packages
	pacman -S xorg-server xf86-video-intel xf86-video-nouveau xf86-input-synaptics \
            xorg-xbacklight xorg-xinit xterm rxvt-unicode compton openbox tint2 \
            conky  dmenu  volumeicon slock feh nitrogen scrot xarchiver p7zip \
            unzip unrar rfkill ttf-liberation ttf-droid ttf-hack terminus-font \
            powertop wget whois ethtool archey3 gvim

	pacman -S ssdm plasma-desktop firefox kate dolphin kdewallet

# Snapper
Hopefully, your system is up and running and everything is configured as described above. By now, we should run some snapper tests. You can either wait for the snapper-services to snapshot your subvolumes automatically, or do one manually:

	# -c configuration file we want to use (contains also details of the subvolume to snapshot etc.
	# -c after create is the flag for the cleanup operation, in our case we want this to be picked up by the snapper-cleanup service
	snapper -c root create -c timeline --description ArchPostInstall

# Sources
#### Wiki-Pages
[Dm-crypt](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Overview)
[BtrFS](https://wiki.archlinux.org/index.php/Btrfs#Creating_a_subvolume)
[Snapper](https://wiki.archlinux.org/index.php/Snapper#Simple_snapshots)
[BtrFS swapfile](https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs%285%29#SWAPFILE_SUPPORT)

#### YouTube
[Install Arch Linux with encrypted volume](https://www.youtube.com/watch?v=x4ZFZ9B0t-8&t=1262s)
[Install Arch Linux with BtrFS](https://www.youtube.com/watch?v=sm_fuBeaOqE&t=596s)
[Customize KDE](https://www.youtube.com/watch?v=uyz4-KZOzyI&t=10s)

#### Reddit
[Reddit comment explaning a more elaborate BtrFS layout](https://www.reddit.com/r/archlinux/comments/fkcamq/noob_btrfs_subvolume_layout_help/fkt6wqs/?context=1000)

#### GitHub
[Note of pkulak (redditor) testing the advanced BtrFS setup](https://gist.githubusercontent.com/pkulak/93270e06ebed35ddc51f4c64bcc3b9b6/raw/e3263c2b341f585b89883c023261fdd810f07ba7/encrypted_btrfs_snapper_notes.sh)

[Full walkthrough by Angel encrypted simple BtrFS setup](https://gist.githubusercontent.com/ansulev/7cdf38a3d387599adf9addd248b09db8/raw/ae318dc9407e0a8b0e7be869c0383edd172599e1/install-arch-linux-on-btrfs-subvolume-inside-luks)

#### Misc
[EF Installation guide (YouTube transcript)](https://ermannoferrari.net/arch-linux-install-with-btrfs-snapper)
