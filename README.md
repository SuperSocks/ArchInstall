# My Arch Linux install

This document exists to track the exact steps I take during my setup of an Arch Linux VM. Hopefully, this will allow me to retrace my steps when I inevitably make a mistake and need to restart. May God have mercy.

# Part 1: preinstall

## Step 1: Download and confirm ISO

I installed the ISO from mit.edu, which was titled "archlinux-2026.10.01-x86_64.iso". It took some time to download as it was 1.5 gigabytes large.

After the download, I had used the `Get-FileHash` command in powershell to get the hash for the file using SHA256

`Get-FileHash "C:\Users\coleb\Downloads\archlinux-2026.10.01-x86_64.iso" -Algorithm SHA256`

Hash I found:
684DED26C63240FF4A41E8C25EE84EA6DA233F557364821F13D12C2B0A9059A5

Hash from Arch wiki:
684ded26c63240ff4a41e8c25ee84ea6da233f557364821f13d12c2b0a9059a5

The hashes are the same, so I didn't download malware! (Unless you think the suffering from installing arch makes it malicious software)

## Step 2: Create new VM (DO NOT OVERWRITE YOUR HARDDRIVE)

From this point I booted up VMware workstation and select "Create a New Virtual Machine". I select the ISO to be the one I downloaded and imediately something offputting happens. The installation window says "Could not detect which operating system is in this disc image." Hopefully this means nothing but we shall see.

Unfortunately this immediately became an issue in the next window as there was no option for arch linux. The only available option was a general option for a linux kernel version 6.X. Some research showed that this option should work, so I went ahead with that.

For the VM options, I chose the following:
- Maximum Disk Size: 32GB
- Store VM disk in a single file

After the VM was built, I opened it up

## Step 3: It's already broken

The virtual machine successfully started and I selected "Arch Linux install medium" and it imediately put out an error message and stopped working. My capslock button was flashing on an off, which indicates some sort of error. I restarted multiple times but got the same error everytime.

Error message here

I tried changing the firmware from BIOS to UEFI and the boot still failed. 

After watching a tutorial at https://www.youtube.com/watch?v=dN7Zc2NaFxE, I realized I had not allocated enough memory to open the live environment. The default option was around 700 MiB while the live environment requires from 1 to 2.5 GiB according to the wiki. After increasin the memory to 4 GiB, I successfully booted into the live system. 

## Step 4: Console format and boot mode

The first step on the wiki after getting into the live environment is to set a keyboard layout and font. The default keyboard layout was working for me, but I did mess around changing fonts for a little before settling on "cybercafe" because I liked the name. 

The next step was to check the system boot mode. I had switched the setting in VMware back to BIOS when I finally got it working, so thats the mode I should be in. The command `cat /sys/firmware/efi/fw_platform_size` returned `No such file or directory` which meant I wasn't in UEFI. However, some further research told me to use UEFI instead as it was the new standard, so I restarted after switching the setting and repeated the command. The output was 64, meaning I was in the correct boot mode.

## Step 5: Internet check

The next step was to connect to the internet. Using `ip link` showed that I had an interface that was up. `rfkill` had no output but `ping google.com` worked, so I assume that has something to do with the fact that I'm on a virtual machine.

## Step 6: Time check

I used `timedatectl` to check the system time while I had a google tab open with the time in UTC. It was accurate to the second. I also used `timedatectl set-timezone America/Chicago` to make the local time correct.

## Step 7: partition the disk

The command `fdisk -l` showed me that I had two disks, "sda" and "loop0", which the wiki says should be ignored. I decided I would follow the table on the wiki almost exactly, 1 GiB for boot, 4 GiB for swap and the rest for the root. 

`fdisk /dev/sda` opened the partition editor, which told me I had no partition tables, and automatically created an MBR table. I pressed g to create a GPT table instead. I think pressed n to create a new partition. I set the start to the default and the end to +1G. This created a partition of 1 GiB. I then pressed t and changed the type of the partition to EFI System by typing uefi.

I then pressed n again to make another partition, used the default start and +4G for the end, creating another partition of 4 GiB. I then changed the type of this one to swap by typing t, 2, and then swap, changing the type to linux swap.

Finally, I created a partition, used the deafult for start and end to fill the rest of the disk, and changed the type to be Linux root (x86-64), using the number 23.

My boot partition is sda1, swap is sda2 and root is sda3

## Step 8: format the partitions

This part was fairly straightforward. I used `lsblk -f` to check my progress as I went.

First I set the root partition to be Ext4 using `mkfs.ext4 /dev/sda3`. I had a small issue here where I accidentally made it swap because I copied the command without changing sda3 to sda2 but I just ran the original command again and I think that fixed it but we'll see if that causes issues

Next I set the swap partition to be swap with `mkswap /dev/sda2`

Finally I set the EFI partition to be FAT32 using `mkfs.fat /dev/sda1`

`lsblk -f` gave the following result:



I also reran the command `mkfs.fat -F 32 /dev/sda1` just to make sure it was FAT32 but I don't think that changed anything.

## Step 9: Mount the file systems

This part was the most straightforward, as I just copied commands from the wiki while using `lsblk -f` to make sure everything seemed right.

`mount /dev/sda3 /mnt` mounted my root filesystem

`mount --mkdir /dev/sda1 /mnt/boot` mounted my boot filesystem

`swapon /dev/sda2` set my swap filesystem to be swap.

# Part 2: Actually install arch

This is where I realized why all the previous steps were under "pre-install". This was just the stuff that any operating system needs, now it was time to actually put arch on the disk I had set up. 

## Step 1: Select mirrors

For this step, I opened the `/etc/pacman.d/mirrorlist` in vim and copied the US mirrors under the worldwide mirrors. Maybe this will make it a little faster and hopefully it wont break it.

## Step 2: install essential packages

I've heard that this step is often the most precarious, as installing the right packages makes or breaks your install. In particular, I will make sure I install a software for networking. 

I ran the command `pacstrap -K /mnt base linux linux-frimware NetworkManager`, which resulted in an error because the package is actually "networkmanager". 

I then ran `pacstrap -K /mnt base linux linux-frimware networkmanager` and everything seemed to work.

# Part 3: Configure system

This section involves actually going into the filesystem that has been setup

## Step 1: Fstab

This file is necessary to mount the filesystems on startup. I simply copied the command from the wiki to do this: `genfstab -U /mnt >> /mnt/etc/fstab`

## Step 2: chroot

To actually interact with the new filesystem, I typed the command `arch-chroot -S /mnt`. Then, to make sure networkmanager had installed I did `NetworkManager --version` and got `1.58.1-2`. 

## Step 3: time

To set the local time, I ran `ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime`

I also ran `hwclock --systohc` according to the wiki, I will figure out time synchronization later.

## Step 4: localization

To generate the locales I used `locale-gen`

Then I downloaded vim using `pacman -S vim` and edited the file "/etc/locale.conf" to have "LANG=en_US.UTF-8" and "KEYMAP=us".

## Step 5: network

I cded into /etc and used `vim hostname` to create the hostname file and made the name "coles-vm"

## Step 6: intramfs

I ran this command because it said so `mkinitcpio -P`. 

## Step 7: set password

I set the password (im not telling you)

## Step 8: download bootloader

First, I downloaded the packages "grub" and "efibootmgr"

Then I ran the command `grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=GRUB`, and it said the installation was complete and there were no errors, very quickly. So I did `ls /boot` to make sure everything was right, and it seemed like everything was there

I then generated the main config file using `grub-mkconfig -o /boot/grub/grub.cfg`

## Step 9: reboot

IT WORKED!!!

I successfully booted into arch and logged into root.

I then ran `systemctl start NetworkManager` which automatically setup a network connection allowing me to ping google.com.

# Part 4: post install

Since I was doing this for an assignment, this part came with 7 specific deliverables:

1. Install a DE
2. Create 2 user accounts with sudo permissions
3. Install another shell
4. install ssh
5. add color coding to the terminal
6. Set the system to boot into the DE
7. create a few aliases

## Step 1: install a desktop environment

I decided to install gnome, as it was the environment I was most familiar with. To do this, I ran `pacman -Syu gnome` which should also check for system updates. This took a little while as it was quite a large install but it was eventually successful. I would leave this alone for now and go through the other steps defore booting into the DE.

## Step 2: Create user accounts

I created a couple of user accounts with sudo permissions by using `useradd -mG wheel`. I then set each of their passwords using `passwd`. However, to actually make the wheel group have the sudo ability, I had to download sudo and edit the sudoers file, which I did with `visudo`. This command didn't initially work, so I set the editor for visudo using `ln /usr/bin/vim vi` while in the /etc/bin directory. With this, visudo was successful and I changed the sudoers file to allow the wheel group to use sudo.

For the rest of the steps, I will operate in the user account I made for myself.

## Step 3: Install a different shell

Since I was in the user account, I used `sudo pacman -Syu fish` to install the fish shell. This worked and I tested it by creating and editing a file in the shell before exiting.

## Step 4: install ssh

I ran `sudo pacman -Syu openssh` which was successful. I tested this by using ssh to connect to a VM of mine and it was successful.

## Step 5: add color to the terminal

To do this, I had to edit the PS1 variable in my .bashrc file. I made it so my username and device name were green and my home directory was blue. The line in .bashrc looked like this: `PS1=\[\033[01;32m\]\u@\h\[\033[01;37m\]:\[\033[01;34m\]\W\$ \\[\033[01;37m\]]"`

## Step 6: Configure to boot into DE

For gnome, this is quite easy. You just have to use the command `systemctl enable gdm.service` and it will boot into GNOME upon reboot. All this actually does is create a link between the systemd display manager file to gdm.service.

## Step 7: Set a few aliases

To add aliases I added the following lines to my .bashrc file

- `alias c='clear'`
- `alias update='sudo pacman -Su'`
- `alias bashrc='vim .bashrc'`
- `alias new='exec $SHELL'`