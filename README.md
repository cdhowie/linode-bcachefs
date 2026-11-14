# linode-bcachefs

This repository contains scripts that can be used to build
[Linode](https://www.linode.com/)-compatible [disk
images](https://techdocs.akamai.com/cloud-computing/docs/images) that contain a
bcachefs root filesystem (usually also with an ext4 `/boot` filesystem).

## debian-bcachefs

Builds an image based on Debian.  Usage is `./debian-bcachefs $SUITE`.  Right
now the only supported suite is `forky`, which is currently the testing release
of Debian.

In order to run this script, you will need the `debootstrap` package installed
as well as a working `bcachefs` binary and kernel module.

The script will create a raw 3GB disk image in the working directory, bind it
to a loopback device, set up a GPT partition table with BIOS boot, `/boot`, and
`/` partitions, mount `/` and `/boot`, debootstrap Debian into the new root,
install bcachefs, GRUB, and other system tools, and finally tear down the
mounts and loopback device.  Finally, per Linode requirements, the image is
gzipped.

## Booting images on Linode

After uploading the image, you can create Linodes using it.  Note that, by
default, Linode will select a direct kernel boot configuration using a
Linode-supplied kernel when you use the web UI to create a Linode from an
image.  This will fail because their kernel does not contain the bcachefs
module.  You will need to change the configuration's "kernel" setting to
"direct disk."  This will instruct Linode to run the bootloader contained in
the disk image, which will run the kernel and load the initrd from the
contained `/boot` volume.

If you use the Linode API to create a Linode, you can supply the correct boot
configuration there to avoid the extra reboot to change the boot configuration
after Linode creation.

Note that Linode's "filesystem/boot/network helpers" are tools that run before
each boot of your Linode and require an ext4 root filesystem to work.
Therefore, they will not work on a bcachefs image.  This is especially
important if you use Linode's legacy "Configuration Profile Interfaces"
networking option, which requires static networking configuration usually
handled by the network helper.  You will have to manually update the network
configuration in this case as the helper will not be able to.  If you use the
newer "Linode Interfaces" option, Linode will provide the network configuration
via DHCP, so networking will be configured automatically.

The default root password is `root`.  Obviously, you should change this.  sshd
is not included in the image; you will need to change the password and then
install sshd using Lish to gain ssh access to the Linode.  In the future, the
build scripts may be modified to copy an ssh key into the image and disable
root's password, making sshd safe to bundle.

You will likely want to resize the partition containing the root filesystem to
fill the disk and then grow the filesystem itself with `bcachefs device resize
$(findmnt -n -o SOURCE --target /)`, otherwise you will be limited to a 3GB
root filesystem.
