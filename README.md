# MargosGentooInstallation
- My (unprofessional) basic approach to install Gentoo on a server.
- GPUs are not present
- Open rc
- uefi


# Prep
1. Download LiveGUI USB Image, or Minimal Installation CD is fine - since this is my version of the handbook, but I still recommand the GUI version since one can connect to Internet (either wireless or with ethernet cable, which is better) with ease, but I will still go with the gui approach on this handbook - from [https://www.gentoo.org/downloads/](https://www.gentoo.org/downloads/)
2. You could write the image .iso file into usb or dvd of your choice. I use Balena Etcher personally, and I love it. [https://etcher.balena.io/](https://etcher.balena.io/)
3. Boot into the live image, obviously.

# In the Live Environment
1. Go to terminal, type `passwd su` and enter. Change to a password of your wish, I usually go for `r`
2. Superuser by `su`, and enter the password.
3. Make partitions on gui by `gparted`, and make the following partitions:

| Partition         | Filesystem   | Size             | label (optional) |
|-------------------|--------------|------------------|------------------|
| /dev/nvme0n1p1    | fat32        | 500 MiB          |    efi           |
| /dev/nvme0n1p2    | xfs          | Everything else  |    root          |
| /dev/nvme0n1p3    | linux swap   | 8 GiB            |    swap          |

Note: different from the Gentoo handbook, imo 500 MiB of efi is enough, and I prefer putting the swap behind the root, but you can put it after efi too. My recommendation for swap is set a size between 0.5 - 1 times of your actual ram (if you have 16 GiB or something), and add swap file instead as needed in the future. Moreover, if swap is after root partition, you could change the size easily in the future, either add or delete the swap partition entirely in the future.

4. Enable and activate the swap partition!! Done by `mkswap /dev/nvme0n1p3` and `swapon /dev/nvme0n1p3`
5. Mount the future root partition temprarily on the live environment by `mkdir /mnt/gentoo` and `mount /dev/sda3 /mnt/gentoo`, we will change the root to there later.
6. Don't forget to create the efi folder too by `mkdir /mnt/gentoo/efi`

!Since the future root is mounted, everything changed inside /mnt/gentoo will be reflected to the future root!

7. Go to the future root by `cd /mnt/gentoo`.
8. Correct the time, so you can wget files online, `chronyd -q`.
9. `wget <find the link to a Stage 3 openrc archives>`.
10. Untar the file you just downloaded by `tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo`.
11. Edit the make.conf, `nano /mnt/gentoo/etc/portage/make.conf`.
12. add `-march=native` to COMMON_FLAGS, so it looks like `COMMON_FLAGS="-march=native -O2 -pipe`. Note: the second part is the letter "O", not the number "0"!
13. Maybe add the RUSTFLAGS? `RUSTFLAGS="${RUSTFLAGS} -C target-cpu=native"`.
14. Important, change MAKEOPTS, I have 8 cores 16 gb, so I use `MAKEOPTS="-j8 -l9"`.
15. I will use GRUB bootloader later, so add the USE flag of grub, dracut (as for initramfs) and maybe dist-kernel too, so the result is like `USE="grub dist-kernel dracut"`
16. save the file and exit by `control + o` and `control + x`.

IF your comupter shutted down before you finished installing, you can return and start here by copying the step 16 here, and resume on whereever you were earlier.

16. Copy the DNS info from the live environment to the new machine, `cp --dereference /etc/resolv.conf /mnt/gentoo/etc/`.

# Chrooting & More
1. Change the working root into the future root, and every file move & command you type here is directly reflected to the future root: `arch-chroot /mnt/gentoo`.
2. Using the new bash, `chroot /mnt/gentoo /bin/bash`
3. `source /etc/profile`
4. `export PS1="(chroot) ${PS1}"`
5. UEFI systems mount the efi partition to the efi folder by `mount /dev/nvme0n1p1 /efi`
6. Fetch the latest Gentoo snapshot with `emerge-webrsync`.
7. I prefer to change mirrors to one that is closer to me physically to maximize the downloading speed. Install mirrorselect by `emerge --ask --verbose --oneshot app-portage/mirrorselect`, and add mirror using `mirrorselect -i -o >> /etc/portage/make.conf`. One use `spacebar` to select mirror and press `enter` to confirm selection to be added to the make.conf. Or you could do the same thing manually.
8. Inspect profiles available with `eselect profile list`
9. Select a profile similar to "default/linux/amd64/23.0", remember the number and inserted into the command: `eselect profile set <choose a number here>`
10. Add a binary source here if you wish, but I prefer compile everything (which is very time consuming!!)
11. Set up the "CPU_FLAGS_", which I don't really care why atm, but it seems necessary. `emerge --ask --oneshot app-portage/cpuid2cpuflags`, and `echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags`
12. Set up the "VIDEO_CARD" by `echo "*/* VIDEO_CARDS: intel" > /etc/portage/package.use/00video_cards`
13. if you are bored, go for `emerge --ask --verbose --update --deep --changed-use @world`, following with `emerge --ask --depclean`
14. Inspecting timezone info with `ls -l /usr/share/zoneinfo`
15. Select a timezone with `ln -sf ../usr/share/zoneinfo/Europe/Berlin /etc/localtime`, change to a timezone you are in. I am in Berlin timezone rn
16. select what you need here, `nano /etc/locale.gen`. I only left "en_US ISO-8859-1" and "en_US.UTF-8 UTF-8" commented.
17. Refresh the locale with `locale-gen`
18. Inspect system-wide locale settings with `eselect locale list`, and change it with `eselect locale set <number>`. I chose the C.utf8 as mentioned in the official Gentoo handbook.
19. Update the locale after you have changed it with `env-update && source /etc/profile && export PS1="(chroot) ${PS1}"`

## Kernel Stuff
20. Install firmware and Sound Open Firmware (since I am on 11th gen i5), a little more tweaks are needed: `emerge --ask --autounmask sys-kernel/linux-firmware`, and run `dispatch-conf`, press `u` to use the new, and run the emerge again to install. Same logic goes to `emerge --ask --autounmask sys-firmware/sof-firmware`
21. I use installkernel and GRUB, so it goes like `emerge --ask sys-kernel/installkernel`
22. Install kernel with `emerge --ask sys-kernel/gentoo-kernel`. Warning, this is going to take a VERY long time to complete. Aim for 2 hours or something.
23. Maybe do a `emerge --depclean` after installing the kernel? Is that okay to you

## Filesystem Stuff
24. Let's do the fstab with `nano /etc/fstab`!

Tbh, you could just copy from here to the fstab. Nothing fancy:\
`/dev/nvme0n1p1   /efi         vfat   umask=0077           0 2`\
`/dev/nvme0n1p2   /            xfs    defaults,noatime     0 1`\
`/dev/nvme0n1p3   none         swap   sw                   0 0`

## Network Stuff
25. Set the hostname with `echo <yourhostname> > /etc/hostname`
26. Change the hosts file if needed `nano /etc/hosts`. I guess not?

## Users and Passwords
27. Change the root password with `passwd`
28. Add an administrative user via `useradd -m -G users,wheel,audio -s /bin/bash <username>`
29. Change the password of that user with `passwd <theusername>`

## Tools
29. Install molocate to enable file system indexing `emerge --ask sys-apps/mlocate`
30. Don't forget to add ssh to the startup runlevel with `rc-update add sshd default`
31. Shell completion, pretty important, install with `emerge --ask app-shells/bash-completion`
32. Use Chrony to sync the time when the OS boots, install with `emerge --ask net-misc/chrony`, and add it to the startup runlevel with `rc-update add chronyd default`
33. Install file system tools:\
XFS: `emerge -a sys-fs/xfsprogs`\
ext4: already included, but if you wish, `emerge -a sys-fs/e2fsprogs`\
VFAT (FAT32, etc.): `emerge -a sys-fs/dosfstools`\
Btrfs: `emerge -a sys-fs/btrfs-progs`\
F2FS: `emerge -a sys-fs/f2fs-tools`\
NTFS: `emerge -a sys-fs/ntfs3g`\
ZFS: `emerge -a sys-fs/zfs`\
bcachefs: e`merge -a sys-fs/bcachefs-tools

34. Scheduler for nvme devices and etc, pretty necessary, right? `emerge --ask sys-block/io-scheduler-udev-rules`
35. Install dhcpd for ethernet connections or so `emerge --ask net-misc/dhcpcd`, or NetworkManager, which uses nmcli to connect to wifi, and I found it easy to use. Install it with `emerge --ask net-misc/networkmanager`. !!I do not recommend install both. Just choose one!
36. Install wireless tools with `emerge --ask net-wireless/iw net-wireless/wpa_supplicant`

## Bootloader Installation
37. Every UEFI needs one extra command than BIOSes before being emerged: `echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf`, so GRUB would install for it, but not for BIOS.
38. Install GRUB with one command, `emerge --ask sys-boot/grub`
39. Use GRUB to generate a bootloader: `grub-install --efi-directory=/efi`
40. Generate the final GRUB configuration with `grub-mkconfig -o /boot/grub/grub.cfg`

## Clean Up
41. All finished up! Now `exit`, and `reboot`. Don't forget to remove the installation medium from the computer, whether the usb or a dvd.
42. Remove the downloaded tar file that is no longer needed with `rm /stage3-*.tar.*`


# Wrap Up
That's pretty much it, a clean installation of Gentoo, You probably want to install xfce4 for the minimized memory and cpu usage, but I guess I will leave that to another day.
Btw, I have not had time to test this work flow. There might be some unexpected situtions, but they are certainly fixable to my knowledge. lol unless either ethernet and wifi both failed (certainly not very possible since NetworkManager is there).
