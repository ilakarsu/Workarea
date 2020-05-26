
# WORK IN PROGRESS
-----
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

Before we encrypt anything, we will run some commands and choose our cipher.

	modprobe dm-crypt
	cryptsetup benchmark

The last command will run some default ciphers and test them. It will yield an overview of the test results. Pick the one, you feel is right for you. Note that we don't use any special options below, using the default parameters. E.g.:

	cryptsetup -v /dev/[devicename]
	# which is equivalent to this:
	cryptsetup -v --type luks --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time 2000 --use-urandom --verify-passphrase luksFormat /dev/[devicename]

This will show a warning and we have to confirm by typing in: 

	Yes <enter>

Next, define a password for your encrypted storage.

	<password> <enter>

To access our encrypted partition,  we must first open it. This procedure is done using the passphrase we have defined in the previous step. Our unlocked partition will be handled by the Kernel using the device mapper, in order to avoid confusion when writing into the partition. This basically alerts our kernel that the device will be available under its new mapped name::

	cryptsetup open /dev/[devicename] [mapped_name]
	cryptsetup open /dev/vda2 cryptroot

Now we have full access to our encrypted storage and can run all the commands as usual. For instance, try to ls and see the output.

##### Organizing the volumes
Next, we need to create our physical volumes, logical volumes and volume groups.

	pvcreate /dev/mapper/cryptroot
	vgcreate main /dev/mapper/cryptroot
	lvcreate -L 2GB -n swap main
	lvcreate -l 100%FREE -n root main

#### Format the partitions
So we have a main volume group that cointains two logical volumes, one for the *swap* partition and one for the *root* partition. This volume group resides on the physical volume /dev/mapper/cryptroot.

Time to format our volumes.

	mkfs.fat -F32 -n BOOT /dev/vda1
	mkfs.btrfs -L root /dev/mapper/main-root

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

 #### Mounting the partitions

Since everything has been created and is ready for mounting, lets continue as follows:

	mount /dev/mapper/main-root /mnt
	mkdir /mnt/boot
	mount /dev/vda1 /mnt/boot

##### BtrFS specific tasks
Ok, so far so good, our boot and root partitions are mounted. Our root partition is using BtrFS. Therefore, we will create some subvolumes now.

    btrfs su cr /mnt/@hometop
    btrfs su cr /mnt/@roottop
    btrfs su cr /mnt/@vlogtop
    btrfs su cr /mnt/@vcchtop

Since we only want to keep snapshots of two subvolumes, namely: root and home. We will only consider those by creating another folder where we store the snapshot images:

	mkdir /mnt/@hometop/live
	mkdir /mnt/@roottop/live

And these itself, will also be subvolumes in our filesystem:

	btrfs su cr /mnt/@hometop/live/snapshot
	btrfs su cr /mnt/@roottop/live/snapshot

Ok, we unmount our fs and will mount in the newly created subvolumes instead.

First:

	umount /mnt

Then (a lot of work but, hey we want a fancy setup ;) ):

    # mount our encrypted volume on /mnt
    mount -o noatime,compress=zstd,autodefrag,subvol=@roottop/live/snapshot /dev/mapper/main-root /mnt
    
    # create subdirectories on roottop
    mkdir -p /mnt/{boot,home,var/log,var/cache,.snapshots}

	# mount the subvolume for homes current (or live) snapshot to the /mnt/home directory
    mount -o noatime,compress=zstd,autodefrag,subvol=@hometop/live/snapshot /dev/mapper/main-root /mnt/home
    
    # mount our roottop subvolume to our roots .snapshot directory
    mount -o noatime,compress=zstd,autodefrag,subvol=@roottop /dev/mapper/main-root /mnt/.snapshots
	
	# as we have mounted our home in the previous step, now we can create a .snapshot dir in it.
    mkdir -p /mnt/home/.snapshots

	# and mount it, same as we did with the root dir.
    mount -o noatime,compress=zstd,autodefrag,subvol=@hometop /dev/mapper/main-root /mnt/home/.snapshots
    
    # we are not keeping any snapshots of our logs or cache (overkill) thus, mounting them on the toplevel is sufficient. Notice, we are not mounting any .snapshot directories.
    mount -o noatime,compress=zstd,autodefrag,subvol=@vlogtop /dev/mapper/main-root /mnt/var/log
    mount -o noatime,subvol=@vcchtop /dev/mapper/main-root /mnt/var/cache

## Configuration of Arch Linux
In this section, we will focus on installing all the necessary software packages. The first sections focus was on getting our preperation tasks done, such as formatting the storage volumes and making sure we have the filesystems we want etc.

First, we will install the basic packages we need, this might differ from use case to use case, but we continue with:

    pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware mkinitcpio lvm2 dhcpcd inetutils intel-ucode nano networkmanager --ignore linux

Not going into the details of all these packages, they can easily be googled, next we update the kernel:

	syslinux-install_update -l -a -m -c /mnt


# Sources
#### Wiki-Pages
[Entire system encryption on a BtrFS system](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Overview)
[Arch Linux dm-crypt](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system)

#### YouTube
[Install Arch Linux with encrypted volume](https://www.youtube.com/watch?v=x4ZFZ9B0t-8&t=1262s)
[Install Arch Linux with BtrFS](https://www.youtube.com/watch?v=sm_fuBeaOqE&t=596s)

#### Reddit
[Reddit comment explaning a more elaborate BtrFS layout](https://www.reddit.com/r/archlinux/comments/fkcamq/noob_btrfs_subvolume_layout_help/fkt6wqs/?context=1000)

#### GitHub
[Note of pkulak (redditor) testing the advanced BtrFS setup](https://gist.githubusercontent.com/pkulak/93270e06ebed35ddc51f4c64bcc3b9b6/raw/e3263c2b341f585b89883c023261fdd810f07ba7/encrypted_btrfs_snapper_notes.sh)
