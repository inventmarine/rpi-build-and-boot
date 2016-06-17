
# Introduction

This is a fork of https://github.com/twobitcircus/rpi-build-and-boot with modifications to provide a Virtualbox VM, to cross compile 
 nodejs native modules for ARM processor, more specifically for the RaspberryPI.
 
*Note*: Read the original project before you decide if this fork is for you.
 

## Differences from the orignal project:
- Use 2016-05-27-raspbian-jessie-lite.img (headless) rather than  2015-09-24-raspbian-jessie.img 
- Remove references to OpenFrameworks.
- Add folder serialport-node-test with aa sample nodejs app, that depends on serialport, which can be used to test node-gyp cross compilation.
- Add CC and CXX environment variables, to allow node-gyp to be cross compiled.

-----------------------------

This package uses Vagrant and Ansible to spin up a virtual machine that runs on Mac, Linux, or Windows(?).  
In no time at all, you'll have an environment that:

- Cross-compiles for the Raspberry Pi armv7l architecture.
- NFS boots one or more Raspberry Pis.  The root partition is loop-mounted from a .img file, so you can later dd it to an SD card for standalone operation.

Here, Vagrant automates the process of setting up a Virtualbox virtual machine and Ansible automates the process of setting up the cross-compiler, the NFS server, and OpenFrameworks.

## Why cross-compile?

The Raspberry Pi is slow.  This environment will let you compile applications on a faster computer/VM.

Though I built this virtual machine with nodejs in mind (and the original author with Openframeworks in mind), it'll work just fine for any cross-compiling task.

## Why NFS-boot?

If you're writing code that runs on a single Raspberry Pi, NFS-booting lets you edit code locally on your desktop.  There's no need to SSH into a Pi to edit and compile.

Because the root partition is loop-mounted from a .img file, you can later create a standalone SD card by merely dd'ing it SD cards.

The magic comes when you're building a cluster of Raspberry Pis.  There's no need to rsync or to reflash a stack of SD cards.  The latest code is accessible on every Pi at all times.  At Two Bit Circus, a common design pattern is to have a cluster of NFS-booted Raspberry Pis all choreographed by a single Linux server.  This virtual machine serves as the starting point for that design pattern.

# Instructions

## Prerequisites

1. Install [VirtualBox](https://www.virtualbox.org/). Or Parallels.  I actually use this with Parallels.
1. Install [vagrant](http://www.vagrantup.com/).
1. Install [Ansible](http://ansible.com).  I used pip.

## Other Dependencies

1. Clone this repository and cd into it.
1. Download your preferred Raspberry Pi SD card image.  I'm using [2016-05-27-raspbian-jessie-lite.img](http://downloads.raspberrypi.org/raspbian_latest).


## Get the image ready

_If you're looking for a cross-compiler solution, chances are you already have an SD card with oF installed.  But just in case..._

1. Burn `2016-05-27-raspbian-jessie-lite.img` to an SD card and boot a Raspberry Pi.
1. Remove the card from the Raspberry Pi and use `dd` to make an image file.
1. _Only do this if your card isn't based on 2016-05-27-raspbian-jessie-lite.img_.  Calculate the offsets to the boot and root partitions on the file.  I've included a tool to calculate these for you automatically (only works on OS X).  Run `./tool.py offsets <my_image.img>`.
1. Copy the output of this tool to the top of `playbook.yml`.

## Create the virtual machine

### Want to NFS boot multiple Raspberry Pis?

_if not, then skip this section_

1. Edit `Vagrantfile` and uncomment the `config.vm.network` line.
1. Create a _wired ethernet_ network.  I use a USB Ethernet adapter on my Macbook.  On my Mac (host machine) the IP/netmask is `10.0.0.2/255.0.0.0`.  The IP address of the virtual machine is hard-coded to `10.0.0.1` (in the Vagrantfile).

### Bring up the virtual machine

1. Type `vagrant up`.  If you're netbooting, choose the wired adapter to bridge to.
1. The machine will start and provision itself.  If there's an error and the provisioning doesn't complete, you can type `vagrant provision` to retry the provisioning process.
1. Get a cup of coffee.  It'll take awhile.
1. Type `vagrant ssh` to connect to and begin using your new environment.

## Cross-compile nodejs native modules!

When you're ssh'ed into your virtual machine, you can access the root partition in /opt/raspberrypi/root.  

From your vagrant shell:

    cd /vagrant/serialport-node-test
    npm install #it will download the dependencies and compile for the host machine (X64)
    npm run rebuild_arm #will navigate into the serialport module and compile it for arm
    zip -r serialport-node-test.zip ./ #will give you a zip file ready to be unziop in a raspberrypi 


## For NFS booters only

The provisioning process in the preceding section has modified your SD card image to enable NFS booting.  We need to write the boot partition _and not the root partition_ to an SD card.  Insert an SD card into your computer.  It can be tiny.  We're only writing about 60MB to it.

1. On OSX I've written a tool to help make bootable cards: `./tool.py netboot image.img /dev/rdiskX [--ip=10.0.0.Y]`
1. On Linux: `dd if=image.img of=/dev/rdiskX ibs=512 obs=1m count=<root_offset_sectors>`.  The root offset is the offset of the root partiion from before, but divided by 512.  On my SD card, that number is 122880.  This particular incantation only copies the first partition (the boot partition) to the SD card.  We don't want a root partition on this card, because it'll be using the NFS share.
1. Examine the `cmdline.txt` file in the newly minted SD card.  It assigns the static IP address 10.0.0.101 (which you can change for subsequent cards) and designates 10.0.0.1 as the NFS server

It's worth noting that the ansible provisioning process has already altered the /etc/fstab on the root partition on the SD card image to inhibit mounting the root partition from the SD card.

## NFS boot your Pi

1. Put the card in a Pi, connect it to the hard-wired network, and turn it on.

## Many Raspberry Pis
Want to run a network of Raspberry Pis all with the same codebase, or with access to the same shared media?  Flash some more SD cards.  Change `cmdline.txt` to set a different IP address for each.  They'll all boot up and have access to that same root partition.

## Standalone Raspberry Pi
Are you finished developing and want to flash a stand-alone SD card that doesn't require NFS booting?  Simply `dd` the _entire_ image.img file to an SD card.  You've been editing that image all along!  

There's some things you'll have to do first:

1. On the Raspberry Pi _root_ partition, alter /etc/fstab and restore the mount point for /
1. On the Raspberry Pi _boot_ partition, remove the stuff after "rootwait"

## Simultaneously develop on your desktop and the Raspberry Pi

If you put an OpenFrameworks project in the rpi-build-and-boot directory, and change config.make to point the OpenFrameworks root at /opt/openframeworks, you can compile in XCode on the Mac side AND compile from the /vagrant directory.

### Notes ###

- *HACK* I create a file in `/etc/cron.d` that tries to mount `/opt/raspberrypi/root` every minute.  I tried to make an upstart job, but upstart is... difficult.
