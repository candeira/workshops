# Install NixOS

## 📖 Overview

This workshop provides a walk through installing, configuring and administrating
the GNU/Linux operating system NixOS. Each module contains some background
information on major features and concepts in NixOS, and includes activities to
apply what you have learned.

By the end of this module you will have:

* Booted into NixOS
* Generated an initial
  [/etc/nixos/configuration.nix][nixos-generate-default-config] config and
  minimally customized it.
* Installed NixOS onto real hardware or into a VirtualBox virtual machine.

## ✋ Before You Begin

It is assumed that you have downloaded the [prerequisites and configure your
computer][prerequisites] before.

If you are running the tutorial on a VirtualBox VM, then you're almost set.

<details><summary>✋🎯️ At the Melbourne :: C◦mp◦se 2019 Workshop we are running
a NixOS proxy-cache for the workshop so that the class doesn't become bottle
necked by internet access speeds. The proxy-cache server name is discoverable by
the mdns name <code>dymaxion.local</code>. Expand this section for configuration
instructions. </summary> <p>

You'll need to do two steps:

1. Activate the proxy-cache in the temporary in-memory operating system
   configuration

```bash
chmod 644 /etc/nixos/configuration.nix
vi /etc/nixos/configuration.nix
```

Add the following lines:

```nix
services.avahi.enable = true;
services.avahi.nssmdns = true;

nix.binaryCaches = [
   "http://dymaxion.local"
   "https://cache.nixos.org"
];
```

Exit your editor and, still as root, activate the configuration:

```bash
nixos-rebuild switch
```
Now run `ping dymaxion.local` to confirm that avahi is functioning and the
proxy-cache is accessible.

</details>

If you are installing NixOS on your computer as its primary operating system then
you'll need to [burn the installation ISO to a USB drive or DVD][burn-the-iso]
and be comfortable with downloading upwards of 4Gb of data.


## 🎯 Installing NixOS from scratch

### Networking

Having internet access during an OS install can be handy to pull in configs. In
the case of NixOS, if you want anything more than a very bare bones system to
boot into, you’re going to want internet access to pull in system packages.

If you are installing into VirtualBox then you don't need to do anything here.

If installing on a computer as its primary operating system, the first thing
you'll need to do after booting is to plug in an ethernet cable or configure WiFi:

```bash
# Generates the actual key used to authenticate on your WPA secured network
wpa_passphrase $SSID $PASSPHRASE > /etc/wpa_supplicant.conf

# Restarts WPA Supplicant, which gives us WiFi for now
systemctl restart wpa_supplicant.service
```

### Partitioning

Time to destroy some valuable data! Just kidding. You won’t make a mistake, and
more importantly, you have 3 copies of your data on at least 2 different types
of storage media and in 2 different physical locations that are unlikely to be
hit by the same disaster right? Right?!

Jokes aside, this process will wipe anything on the disk. Consider yourself
warned.

A UEFI boot device requires a GUID partition table (GPT). Hence we’ll be using
`gdisk` instead of the venerable `fdisk`. If you’re installing on a system that
doesn’t use UEFI, then you can do a similar job with good ’ol `fdisk`.

To start, delete any existing partitions and start with a clean slate:

```bash
# Identify the disk to install NixOS on - something like /dev/nvme0n1 or /dev/sda.
# We'll refer to it as $DISK.
lsblk

# Open gdisk on the disk we're installing on
gdisk $DISK

# print the partitions on the disk
Command: p

# Delete a partition. Select the partition number when prompted.
# Repeat for all partitions.
Command: d
```

Now you'll need to create the partitions we need: an EFI boot partition, and an
LVM partition. LVM (logical volume management) allows us to more easily change
our partitions (size and layout) should we need. In our case, the LVM partition
will contain our root and swap partitions.

This code block assumes you’re still at a gdisk prompt.

```bash
# Create the EFI boot partition
Command: n
Partition number: 1
First sector: <enter for default>
Last sector: +1G       --  make a 1 gigabyte partition
Hex code or GUID: ef00 -- this is the EFI System type

# Create the LVM partition
Command: n
Partition number: 2
First sector: <enter for default>
Last sector: <enter for default - rest of disk>
Hex code or GUID: 8e00 -- Linux LVM type

# Write changes and quit
Command: w
```

#### Encryption and LVM

Your partition table and primary partitions are now in place. Now we can encrypt
the partition that will contain your LVM partitions. This is the second
partition that was created above - so should be something like `/dev/nvme0n1p2`
or `/dev/sda2`. We’ll refer to it as `$LVM_PARTITION` below.

Note that your boot partition won’t be encrypted but your swap partition will be
encrypted. You don’t have any control over what’s moved into your swap space, so
it is important to encrypt it as it could end up containing all sorts of private
stuff in the clear - for example passwords copied from a password manager.

In the example below, we’re creating a swap space that is the same size as our
RAM (16GB), and filling the rest of the disk with our root filesystem. You might
want to tweak these sizes for your machine.

```bash
# You will be asked to enter your passphrase - DO NOT FORGET THIS
cryptsetup luksFormat $LVM_PARTITION

# Decrypt the encrypted partition and call it nixos-enc. The decrypted partition
# will get mounted at /dev/mapper/nixos-enc
cryptsetup luksOpen $LVM_PARTITION nixos-enc

# Create the LVM physical volume using nixos-enc
pvcreate /dev/mapper/nixos-enc 

# Create a volume group that will contain our root and swap partitions
vgcreate nixos-vg /dev/mapper/nixos-enc

# Create a swap partition that is 4G in size - the amount of RAM configured
# in your virtual machine or the total amount of memory on your machine.
# Volume is labeled "swap"
lvcreate -L 4G -n swap nixos-vg

# Create a logical volume for our root filesystem from all remaining free space.
# Volume is labeled "root"
lvcreate -l 100%FREE -n root nixos-vg
```

### Create Your filesystems

In the below snippet, `$BOOT` refers to the boot partition created above -
something like /dev/sda1.

```bash
# Create a FAT32 filesystem on our boot partition
mkfs.vfat -n boot $BOOT

# Create an ext4 filesystem for our root partition
mkfs.ext4 -L nixos /dev/nixos-vg/root

# Make your swap partition to be a swap
mkswap -L swap /dev/nixos-vg/swap

# Turn the swap on before install
swapon /dev/nixos-vg/swap
```

### Mount file-systems and prep for install

You're almost there. Now it’s time to mount the partitions you created, put your
system configuration in place, and finally, install NixOS.

The snippet below uses `$BOOT_PARTITION` as a placeholder for the UEFI boot
partition we created earlier. This was the first partition on the disk, and will
probably be something like `/dev/sda1` or `/dev/nvme0n1p1`.

```bash
mount /dev/nixos-vg/root /mnt
mkdir /mnt/boot
mount $BOOT_PARTITION /mnt/boot
```

Now that you have file-systems that can be written, let’s generate our initial config.

```bash
nixos-generate-config --root /mnt
```

### Configuration

NixOS is primarily configured by `/etc/nixos/configuration.nix`. Given that your
pre-installation root filesystem is mounted at `/mnt`, you'll find the
configuration at `/mnt/etc/nixos/configuration.nix` for now.

Let’s open it up and set some important options.

```bash
# Or, you know, use `nano` or whatever else you might prefer.
vim /mnt/etc/nixos/configuration.nix
```

If anything is broken in your config, installation will fail with an error
message to help diagnose your problem. Furthermore, because NixOS is the way it
is, you can radically reconfigure your system later knowing that you can
fallback to a known good configuration, and once you’re confident everything
works, clean up packages you no longer need. In short, don’t stress too much
about installing and configuring absolutely everything. It’s fine to start with
a small but working system and build up as you learn what you want.

It is of critical importance that you tell NixOS about the LUKS encrypted
partition which was created as needs to be decrypted before it's possible to use
to any LVM partitions. We do that like so.

```nix
boot.initrd.luks.devices = [
  { 
    name = "root";
    device = "/dev/nvme0n1p2";
    preLVM = true;
  }
];
```

NixOS also needs to know that we’re using EFI, however it may have been
correctly configured for you automatically.

```nix
boot.loader.systemd-boot.enable = true
```

If you are installing NixOS as your primary operating system, now would be a
good time to enable WiFi support via wpa_supplicant.

```nix
networking.wireless.enable = true;
```

<details><summary>✋🎯️ At the Melbourne :: C◦mp◦se 2019 Workshop we are running
a NixOS proxy-cache for the workshop so that the class doesn't become bottle
necked by internet access speeds. The proxy-cache server name is discoverable by
the mdns name <code>dymaxion.local</code>. Expand this section for configuration
instructions. </summary> <p>

You'll need to do two steps:

1. Activate the proxy-cache in the temporary in-memory operating system
   configuration

```bash
chmod 644 /etc/nixos/configuration.nix
vi /etc/nixos/configuration.nix
```

Add the following lines:

```nix
services.avahi.enable = true;
services.avahi.nssmdns = true;

nix.binaryCaches = [
   "http://dymaxion.local"
   "https://cache.nixos.org"
];
```

Activate the configuration:

```bash
nixos-rebuild switch
```

2. Activate the proxy-cache in the final configuration

```bash
vi /mnt/etc/nixos/configuration.nix
```

Add the following lines:

```nix
services.avahi.enable = true;
services.avahi.nssmdns = true;

nix.binaryCaches = [
   "http://dymaxion.local"
   "https://cache.nixos.org"
];
```

Now run `ping dymaxion.local` to confirm that avahi is functioning and the
proxy-cache is accessible.

> 🛈 You can declare your `binaryCaches` on the command line. The option is called `substituters`. During the workshop, have a play around with:
>
> ```bash
> nixos-rebuild --option substituter https://cache.nixos.org/ test
> nixos-rebuild --option substituter $LOCAL_CACHE_PROXY_IP test
> ```
>
> Knowing this can be a lifesaver if you stuff up your `binaryCaches` configuration, or if you don't stuff it up but the caches change after you configure them. For instance, you might need to run `nixos-rebuild --option substituter https://cache.nixos.org/ switch` after you leave the workshop, as the `dymaxion.local` proxy-cache will no longer be accessible.
</p>
</details>

### Start the install

Once you’re happy with your configuration, we can pull the trigger on an install.

```bash
nixos-install
# IT'LL ASK YOU FOR YOUR ROOT PASSWORD - DON'T FORGET IT
# IT'LL ASK YOU FOR YOUR ROOT PASSWORD - DON'T FORGET IT
# IT'LL ASK YOU FOR YOUR ROOT PASSWORD - DON'T FORGET IT
reboot
```

Go get a coffee while everything installs, and hopefully you’ll reboot to your new system. If something has gone wrong, don’t worry. You can always boot back into the installation media, mount your partitions, update the configuration, and install again. To mount existing partitions, you’ll need to decrypt the LVM partition and then activate its volume group.

```bash
cryptsetup luksOpen $LVM_PARTITION nixos-enc
lvscan
vgchange -ay
mount /dev/nixos-vg/root /mnt
vim /mnt/etc/nixos/configuration.nix
nixos-install
```

## 📚 Additional reading material

* [NixOS Manual: Chapter 2. Installing NixOS](https://nixos.org/nixos/manual/index.html#sec-installation).
* [Instructions for booting from an Ubuntu liveCD and installing NixOS on a machine](https://gist.github.com/chris-martin/4ead9b0acbd2e3ce084576ee06961000).
* [Installating NixOS with encrypted root](https://gist.github.com/martijnvermaat/76f2e24d0239470dd71050358b4d5134).
* [Installing NixOS on Hetzner bare metal with ZFS](https://ghuntley.com/notes/hetzner/).

## ⏭️ What's next

<!-- in-line links -->
[burn-the-iso]: https://nixos.org/nixos/manual/index.html#sec-booting-from-usb
[prerequisites]: ./00-prerequisites/README.md
[nixos-generate-default-config]: ../../configurations/nixos-generate-default-config/configuration.nix
