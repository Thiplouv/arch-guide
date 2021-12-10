# Archlinux Installation Guide - Thiplouv's Edition
> This guide is a reminder for a specific Arch Linux installation. You can use it as you wishes, but remember that it contains a lot of specific stuff that might differ from your tastes.

___
## **Pre-installation**
> This part ensure that the iso is ready for a optimised installation.
* Archlinux iso can be downloaded via [this French repo](http://archlinux.polymorf.fr/iso/latest/).

* Changing to **AZERTY** keyboard layout can be achieved this way: 
  >```loadkeys fr-latin1```

* To see if the system boot mode (uefi or legacy boot), check if there is any efi partitions mounted this way: 
  >```ls /sys/firmware/efi/efivars```

* Verify is the iso is connected to internet:
  * List the network interfaces:
    > ```ip link```
  * Check if the connection is working:
    > ```ping www.archlinux.org```

* Update the system clock:
  * Enable the Network Time Protocol
    > ```timedatectl set-ntp true```
  * Check the service status
    > ```timedatectl status```
___
## **Disks partitionning**
> This part covers the disk partitionning process. This guide shows only the uefi partitioning scheme, as it is the boot mode I use. However, you can choose to use different schemes, as described in the [Archlinux wiki](https://wiki.archlinux.org/title/Installation_guide#Partition_the_disks).

To identify all the disks (and the partitions) that are currently mounted, run:
> ```fdisk -l```


### **Creation**
Next, we can launch the partitionning utility using:
> ```cfdisk```

As we are using the uefi boot mode fore this installation, we choose **GPT** as the partition table.

The partition sheme that we will choose is the following one:

| Mount point  | Partition   | Partition type        | Size                    |
|:-------------|:------------|:----------------------|:------------------------|
| _/mnt/efi_   | _/dev/sda1_ | EFI System Partition  | 512M                    |
| [SWAP]       | _/dev/sda2_ | Linux Swap            | 8G                      |
| _/mnt_       | _/dev/sda3_ | Linux x86-64 root (/) | Remainder of the device |

<span style="color: #FF0000">Warning</span>: here, sub-partitions are noted _sda_, but it can be anything else depending of the type of storage you use (example: /_dev/nvme0_ for NVME ssds).

### **Formatting**
Now that we have all the partitions we need, we can format them:

* We begin with the EFI partition:
    > ```mkfs.fat -F 32 /dev/sda1```

* Next, we can initialize the swap partition:
    > ```mkswap /dev/sda2```

* Finally, we can format the root partition in EXT4:
    > ```mkfs.ext4 /dev/sda3```

### **Mounting**
Now that the partitions are created and formatted, we can initialize them.

* First, let's mount the root volume to /mnt:
    > ```mount /dev/sda3 /mnt```

* Next, let's prepare the efi partition:
    > ```mkdir /mnt/efi```
    
    > ```mount /dev/sda1 /mnt/efi```

* Finally, we can enable the swap partition:
    > ```swapon /dev/sda2```

Now, all the partitions are ready and we can begin the installation process.
___

## **Installation**
> Now starts the real Archlinux installation process, with basic system configuration.

### **Essential packages installation**

To ensure a fast and stable connection, let's configure Pacman's mirror servers:

    nano /etc/pacman.d/mirrorlist

**Note:** Using _Alt+R_ keys, we can replace _Server_ and replace all of them by _#Server_.

Now we can add our prefered repository. Mine is:

    Server = http://archlinux.polymorf.fr/$repo/os/$arch

Now that the mirror servers are configured, it's time to install essentials program, such as:
* Linux kernel and headers,
* Bootloaders,
* Networking programs
* Text editors,
* Manual pages,
* Other stuff that you want to have installed when your system will restart.

Here is what I install:

    pacstrap /mnt base base-devel linux linux-firmware grub efibootmgr os-prober dhcpcd neovim nano man-db man-pages texinfo sudo

**Note:** don't forger to add _iwd_ if you need to use WIFI.


### **Basic system configuration**

Let's start by generating FSTAB file:

    genfstab -U /mnt >> /mnt/etc/fstab

Now, we can change root into the new system:

    arch-chroot /mnt

At this point, we can start to configure the new installed system.
Let's start with time zone and locales.

    ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
    hwclock --systohc

**Note:** The above line needs a Windows registery modification if a dualboot is installed. See [this guide](https://wiki.archlinux.org/title/System_time#UTC_in_Microsoft_Windows).

Let's now edit the localization :

    nvim /etc/locale.gen

As I want my system to be in English but with European features, such as metric measurements,etc... I will use the Ireland English Locale. Do as you wishes.

    # Uncoment these lines:
    en_IE.UTF-8
    en_US.UTF-8

We can now generates the locales by running:

    locale-gen

Let's now change the language:

    nvim /etc/locale.conf
    ---------------------
    LANG=en_IE.UTF-8
    LC_COLLATE=C

And we can make the console keyboard layout persistent:

    nvim /etc/vconsole.conf
    -----------------------
    KEYMAP=fr-latin1

Let's also add the hostname of the computer:

    nvim /etc/hostname
    ------------------
    <myhostname>

Before going to the bootloader installation, let's change the root password by running:

    passwd


### **Bootloader Installation**
> In this part we are going to install the GRUB bootloader, with UEFI boot mode.

**Note:** Remember that the EFI partition is already mounted to the system at /efi.

To install GRUB on your system, run this command:

    grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB

Before generating the main configuration file, I like to enable os-prober for searching for other os if a dualboot is configured:

    nvim /etc/default/grub
    ----------------------
    # Uncomment to enable os-prober
    GRUB_DISABLE_OS_PROBER=false

Once it is done, we can generate the GRUB main configuration file by running:

    grub-mkconfig -o /boot/grub/grub.cfg

Now that grub is installed, we can exit from the chrooted system and restart the computer to see if the installation was successful. 

Just run:

    exit
    reboot

## **Post-installation**
> In this part, we are going to cover some post-installation stuff that I like to do once my system restarted.

### **Internet connection**
After logging in as root, let's start the dhcp service to and verify the internet connection:

For a wired connection with dynamic ip address :

    systemctl start dhcpcd
    ping www.archlinux.org

For a wireless connection, run:

    systemctl start iwd
    systemctl start iwctl
    ping www.archlinux.org

If the connection works, don't forget to enable these systemd services by replacing the _start_ command to _enable_.

### **Video driver**
Before any further ado, let's install the video driver required by our hardware.

For a **Virtual Machine** (such as vmware), just run:

    pacman -S open-vm-tools
    systemctl enable vmtoolsd.service
    systemctl enable vmware-vmblock-fuse.service

For a **Nvidia** graphic card, just run:

    pacman -S nvidia nvidia-settings
    reboot

After the system has rebooted, just run:

    nvidia-xconfig

    # Verify your xorg configuration file:
    nvim /etc/X11/xorg.conf

**Note:** The documentation for the Nvidia Drivers installation can be found [here](https://wiki.archlinux.org/title/NVIDIA).

### **User Management**
Now that the system is working and is connected to internet, it is time to add our user to the system.

To do so, just run:

    useradd -m -G wheel -s /bin/bash thib
    passwd thib

As we added the new created user to the wheel group, let's give this group the permitions to use sudo commands:

    EDITOR=nvim visudo
    ------------------------------------------------------------------
    # Uncomment to allow members of group wheel to execute any command
    %wheel ALL=(ALL) ALL

Now that our new user is created and can run sudo commands, we can logout from the root user and login with our new user. Just run:

    exit


### **Window manager**
 Now, it is time to install a window manager.

Before that, we need to install the X Window System (Xorg). To do so, just run:

    sudo pacman -S xorg xinit

Once the operation is successful, we can install the window manager of our choice. I will go for i3wm.

    sudo pacman -S i3-gaps i3lock i3status i3blocks

These are the core packages for the i3 windoww manager, but I also usually install:

* A compositor
* A program launcher
* A terminal
* A wallpaper manager
* Git
* The user directories
* An AUR client
* etc...

This can be done by running:

    sudo pacman -S picom dmenu alacritty nitrogen git github-cli xdg-user-dirs
    xdg-user-dirs-update

I usually use yay as my AUR client:

    cd Downloads/
    git clone https://aur.archlinux.org/yay.git
    cd yay
    makepkg -si

Now that all of this stuff is done, we can configure Xinit to start our window manager. To do so, let's start by copying the default xinit configuration file:

    cp /etc/X11/xinit/xinitrc ~/.xinitrc

Now, let's replace the default configuration by ours:

    # Replace these lines:
    twm &
    xclock -geometry 50x50-1+1 &
    xterm -geometry 80x50+494+51 &
    xterm -geometry 80x20+494-0 &
    exec xterm -geometry 80x66+0+0 -name login

    # By those lines:
    setxkbmap -layout fr
    picom &
    nitrogen --restore &
    exec i3

Now, you should be able to start the newly installed window manager by running:

    startx

And that's the end ! Now, you can continue to configure your system as you wishes.