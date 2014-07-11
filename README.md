# Introduction

This package uses Vagrant and Ansible to spin up a virtual machine that runs on Mac, Linux, or Windows(?).  In no time at all, you'll have an environment that:

- Cross-compiles for the Raspberry Pi armv6l architecture.
- NFS boots one or more Raspberry Pis.  The root partition is loop-mounted from a .img file, so you can later dd it to an SD card for standalone operation.
- Is ready to cross-compile OpenFrameworks applications and run them on a cluster of Raspberry Pis.

You won't have to do any of the things on [the Raspberry Pi Cross-Compiling Guide](http://www.openframeworks.cc/setup/raspberrypi/Raspberry-Pi-Cross-compiling-guide.html) page.  I have contributed to that guide to make the process more smooth, but it's still a mess.

Here, Vagrant automates the process of setting up a Virtualbox virtual machine and Ansible automates the process of setting up the cross-compiler, the NFS server, and OpenFrameworks.

## Why cross-compile?

The Raspberry Pi is slow.  This environment will let you compile OpenFrameworks applications on your fast desktop.

Though I built this virtual machine with OpenFrameworks in mind, it'll work just fine for any cross-compiling task.

## Why NFS-boot?

If you're writing code that runs on a single Raspberry Pi, NFS-booting lets you edit code locally on your desktop.  There's no need to SSH into a Pi to edit and compile.

Because the root partition is loop-mounted from a .img file, you can later create a standalone SD card by merely dd'ing it SD cards.

The magic comes when you're building a cluster of Raspberry Pis.  There's no need to rsync or to reflash a stack of SD cards.  The latest code is accessible on every Pi at all times.  At Two Bit Circus, a common design pattern is to have a cluster of NFS-booted Raspberry Pis all choreographed by a single Linux server.  This virtual machine serves as the starting point for that design pattern.

# Instructions

## Prerequisites

1. Install [VirtualBox](https://www.virtualbox.org/).
1. Install [vagrant](http://www.vagrantup.com/).
1. Install [Ansible](http://ansible.com).  I used [pip](https://devopsu.com/guides/ansible-mac-osx.html).

## Other Dependencies

1. Clone this repository and cd into it.
1. Download your preferred Raspberry Pi SD card image.  I'm using [2014-06-20-wheezy-raspbian](http://downloads.raspberrypi.org/raspbian_latest).  Unzip it, and symlink it with `ln -s 2014-06-20-wheezy-raspbian.img image.img`
1. Download a zip file of the Raspberry Pi [cross-compiler tools](https://github.com/raspberrypi/tools/archive/master.zip).  Make sure the zip file is named `tools-master.zip`.  Leave it compressed.
1. Download [OpenFrameworks for armv6](http://www.openframeworks.cc/versions/v0.8.1/of_v0.8.1_linuxarmv6l_release.tar.gz).  Leave it compressed.

## Get the image ready
_You only have to do this if you're not using 2014-06-20-wheezy-raspbian._

The NFS root with which you're booting the Pi lives in the image.img file.  You need to calculate the offsets to the boot and root partitions on that device.  On OSX you can just type `file image.img`.  The relevant information here is the start sector for the boot (1st) and root (2nd) partitions.  Multiply the start sector by the block size (512) to get the byte offset.  Put these numbers in `offset_boot` and `offset_root` in playbook.yml.

## Create the virtual machine

1. Configure a _wired ethernet_ network.  I use a USB Ethernet adapter on my Macbook.  Set the IP/netmask of the wired connection to `10.0.0.2/255.0.0.0`.  The IP address of the virtual machine is hard-coded to `10.0.0.1`.
1. Type `vagrant up`.  It will probably ask you to select the network interface to bridge to.  Select your wired connection.
1. The machine will start and provision itself.  If there's an error and the provisioning doesn't complete, you can type `vagrant provision` to retry the provisioning process.
1. Get a cup of coffee.  It'll take awhile.
1. Type `vagrant ssh` to connect to and begin using your new environment.

## Make a bootable card

The provisioning process in the preceding section has already modified your SD card image to enable NFS booting.  We need to write the boot partition _and not the root partition_ to an SD card.  Insert an SD card into your computer.  It can be tiny.  We're only writing about 60MB to it.

1. On OSX and Linux: `dd if=image.img of=/dev/rdiskX ibs=512 obs=1m count=<root_offset_sectors>`.  The root offset is the offset of the root partiion from before, but divided by 512.  On my SD card, that number is 122880.  This particular incantation only copies the first partition (the boot partition) to the SD card.  We don't want a root partition on this card, because it'll be using the NFS share. Remember to unmount the partition but not eject the disk.
1. Examine the `cmdline.txt` file in the newly minted SD card.  It assigns the static IP address 10.0.0.101 (which you can change for subsequent cards) and designates 10.0.0.1 as the NFS server

It's worth noting that the ansible provisioning process has already altered the /etc/fstab on the root partition on the SD card image to inhibit mounting the root partition from the SD card.

## NFS boot your Pi

1. Put the card in a Pi, connect it to the hard-wired network, and turn it on.
1. If this is a fresh card, you'll be dropped into raspi-config.
  1. Enable ssh in the advanced options
  1. Do _not_ "expand the filesystem."
1. Finish raspi-config and reboot.

## Configure the Pi for OpenFrameworks

The virtual machine is set up with the tools to do this automatically. 

1. Type `vagrant ssh` to drop into a shell on your virtual machine.
1. `ssh pi@10.0.0.101` and log in (the password is `raspberry`).  Immediately log back out by typing 'exit'. We did this to add the host to the known_hosts file.
1. From the vagrant shell, type `ansible-playbook --ask-pass -i "10.0.0.101," /vagrant/openframeworks-pi.yml` (the password is 'raspberry').  *Note: I may have gone overboard here.  This does the same as running install_dependencies.sh on the Raspberry Pi from /opt/openframeworks/scripts/linux/linux_armv6l.  If you go this route, there's an http_proxy running at http://10.0.0.1:8888*  
1. Another cup of coffee.

## Test it out!

When you're ssh'ed into your virtual machine, you can access the root partition in /opt/raspberrypi/root.  Dig deeper, and you'll find /opt/raspberrypi/root/opt/openframeworks.  This is the armv6 OpenFrameworks directory, uncompressed and ready to go.  It's symlinked to /opt/openframeworks for simplicity.

The virtual machine has an alias `rmake` which calls the cross-compiler with the appropriate options.

From your vagrant shell:

    cd /opt/openframeworks/apps/myApps/emptyExample
    rmake

From your Raspberry Pi:

    cd /opt/openframeworks/apps/myApps/emptyExample
    bin/emptyExample

# Here's where it gets interesting

Big deal, right?  Well, consider what you can do with this development environment.

## Many Raspberry Pis
Want to run a network of Raspberry Pis all with the same codebase, or with access to the same shared media?  Flash some more SD cards.  Change `cmdline.txt` to set a different IP address for each.  They'll all boot up and have access to that same root partition.

## Standalone Raspberry Pi
Are you finished developing and want to flash a stand-alone SD card that doesn't require NFS booting?  Simply `dd` the _entire_ image.img file to an SD card.  You've been editing that image all along!  

There's some things you'll have to do first:

1. On the Raspberry Pi _root_ partition, alter /etc/fstab and restore the mount point for /
1. On the Raspberry Pi _boot_ partition, remove the stuff after "rootwait"

## Simultaneously develop on your desktop and the Raspberry Pi

If you put an OpenFrameworks project in the rpi-build-and-boot directory, and change config.make to point the OpenFrameworks root at /opt/openframeworks, you can compile in XCode on the Mac side AND compile from the /vagrant directory.






