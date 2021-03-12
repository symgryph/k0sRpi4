# Notes on building a rpi 4 kubernetes cluster

This document explains the necessary processes for building a RPI4 kubernetes cluster from scrath using alpine with f2fs process. 

## Getting boot disk ready with alpine

This part is a little tricky. Basic Steps:

1. Get the RPI 4 image from alpine, [here](https://alpinelinux.org/downloads/) be sure to get the aarch64 version!
2. go to [this link](https://wiki.alpinelinux.org/wiki/Raspberry_Pi_4_-_Persistent_system_acting_as_a_NAS_and_Time_Machine) which shows basics of getting things on the working.

3. Finish up until the 'date' part. I didn't bother silencing the fan.
4. Add the f2fs utils to the image
5. Make sure the image boots after everthing is done.

At this point you should have an image ready for coversion to f2fs (its entirely optional, but WAY better than ext4).

## Converting working system to f2fs

Basic steps:

1. On a dedicated debian system, you will need to do these steps:
2. Create raw image of working Sd card from part 1 this will take about 20 minutes or so
3. Copy the image to a new sd card (to avoid erasing your original)
4. make a new f2fs partition using the mkfs.f2fs /dev/mmcblk0p2 (overwriting the ext4 version)
5. Loopback mount the 'origial' that you dded via losetup

```bash
losetup -v -f /mnt/myimage.img
losetup -a
partx -v --add /dev/loop0
mkdir /media/from
mkdir /media/to
mount /dev/loop0p2 /media/from
mount /dev/mmcblk0p2 /media/to
cp -a /media/from/ /media/to
apt-get install vim
vi /etc/fstab
```

Make sure you edit the FSTAB on the copy!!!!

Your fstab should look thus:

```ini
/dev/mmcblk0p2  /       f2fs    rw,noatime,discard 0 1
/dev/cdrom      /media/cdrom    iso9660 noauto,ro 0 0
/dev/usbdisk    /media/usb      vfat    noauto  0 0
/dev/mmcblk0p1  /media/mmcblk0p1 vfat   defaults 0 0
```

There is a very nasty bug with fsck that I had to iron out (found it in a single post!) when running fsck: you need to add the forcefsck paramater to the boot (cmdline.txt). For quicker (if somewhat unsafer) reboots you could just do 0 0 which would avoid fsck alltogether. I like to be 'safe' and since i don't reboot that often, its not a problem.

```bash
mkdir /media/dos
mount /dev/mmcblk0p1 /media/dos
vim /media/dos/cmdline.txt
```

Your cmdline.txt needs to look like this with the fix in it:

```ini
modules=loop,squashfs,sd-mod,usb-storage quiet console=tty1 root=/dev/mmcblk0p2 forcefsck
```

### Fix CPU stupidity

The PI has a terrible bug in it that it uses a 95% threshold for using reasonable 'cpu bursting'. This means that your cpus will be at 600Mhz until you hit 95% for a sustained period of time. By making the cpu on demand a more reasonable 10% before we kick up, we can still idle at 600Mhz when we aren't busy, but if we get to 10% cpu our system will kick into overdrive. 

This has the rather terrible effect (the default that is) of making the PI have a max of 20MBPS instead of 80MBPS or more when doing network traffic. When the cpu is at a proper level, it will do 80-90MBPS. So here is the script.

```bash
#!/usr/bin/env ash
logger "Starting CPU FIX"
echo "ondemand" | tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor | logger &
logger "Sleeping for 3 seconds"
sleep 3
logger "Starting on demand threshold to 10%"
echo "10" | tee /sys/devices/system/cpu/cpufreq/ondemand/up_threshold | logger &
```

In order for this to work, the script must be placed in the ```/etc/local.d`` directory. Additionally, the following commands have to be issued to make sure things 'start':

```bash
#!/usr/bin/env ash
logger "Starting CPU FIX"
echo "ondemand" | tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor | logger &
logger "Sleeping for 3 seconds"
sleep 3
logger "Starting on demand threshold to 10%"
echo "10" | tee /sys/devices/system/cpu/cpufreq/ondemand/up_threshold | logger &
```

Edit this file in your favorite directory. The filename should be something like ```fixcpu.sh.start``` the important part is the ```.start``` suffix on the end of the file as this tells the system to 'start'. Logger sends nice logs to our syslog server to tell us that things are O.K. 

Unmount the disks and you are ready to now have a 'pristine' image to clone. Be sure to test.  For the tryly impatient (and not afraid of heat death) the default governer can be set to 'performance', but this is a topic for a different day.

```bash
echo "Turning on burn your processor mode"
echo performance |  tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Just replace these inside of your script instead of the top ones.

Finally, to enable things to work after editing is done:

```bash
cd /etc/local.d
chmod +x fixprocessor.sh.start
rc-update add local default
```

Now you are ready to shut down the system in an orderly fashion:

```bash
poweroff
```



## Cloning the disk on a dedicated 'cloning' linux box with clonezilla

As you probably noticed from the first 'dd' it takes a really long time to do an entire 32gb of disks. I found a way to get a 'pristine' image burned (assuming you have an sd card writer on your linux box!) using clonezilla. I am not going to go through everything, just the necessary tools to get clonezilla working on our debian machine. Also I include a 'fix' for sd cards not properly detecting ejects on my duplicator appliance.

### Prerequisites

These are the minimum pre-requisites for things to work

1. Sd card reader recognized by system
2. large disk attached for images to live on (can be network or otherwise)
3. Debian installed and working

#### Install necessary packages to get cloning working

Install the following packages:

```bash
apt-get install clonezilla
apt-get install f2fs-utils
apt-get install f2fs-tools
apt-get install ncdu
apt-get install parted
apt-get install dosfstools
apt-get install lzop
apt-get install zstd
apt-get install bmap-tools
apt-get install partclone
apt-get install apt-file
apt-get install lvm2
```

#### Image Fixing

Before you clone, its wise to 'check' the dos partition for errors on a windows box or mac using 'repair' option. If the file system is marked as 'unclean' the system will not properly clone and you will get an error.

## Actual Clone process

Assuming that everything is up, you need to run clonezilla. I am not going to determine what command you should use, just offer some basic pointers.

1. You should have a large mount point called /home/partimag where all images will be stored. This can be an attached disk, nfs share, etc.

   1. You will run clonezilla to get a nice gui. I am going to assume that you have already mounted the disk to /home/partimag doing so will fill up root and you will be sad

   2. choose "device/image" when imaging. This will get you a full disk. Also the thing to be duplicated should NOT be mounted (/dev/mmcp0).

   3. I then choose 'skip' for the /home/partimag location you could optionally mount a remote share here etc.

   4. choose 'beginer'

   5. choose 'save disk'

   6. give it a name

   7. choose appropriate unmounted disk (in my case mmcblk0)

   8. skip checking

   9. yes check the saved image

   10. not to encypt the image

   11. SAVE the command if you ever want to do this again! in my case:

       ```
       /usr/sbin/ocs-sr -q2 -c -j2 -z1p -i 4096 -p true savedisk masterdiskf2fsalpinerpi4 mmcblk0
       ```

   12. my nice one with compression etc:

       ```bash
       /usr/sbin/ocs-sr -q2 -j2 -z3 -i 10000 -fsck-src-part-y -p true savedisk interactivetestclonef2fswithtimecorrected mmcblk0
       ```

       just say yes as its giong to image now

   13. More advanced options are possible (suhc as automatic compression etc buut I won't cover here)

## Duplicate Image Process

Now that the image has been duplicated, restoring is just a matter of pressing keys. I am not going to cover this. Just one thing: if you find that mmc cards won't eject properly w/o a reboot: I have a fix (for fitlets at least)

```bash
#!/usr/bin/env bash
rmmod sdhci_pci
modprobe sdhci_pci
```

create this file as 'fixsd' in /bin and make executable. remove disk, put in new disk, and run. should work w/o a reboot.

```bash
/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -p true restoredisk rpi-alpine-f2fs-master mmcblk0
```



## More advanced usage

You could optionally install a clonezilla server cd and use that. this was 'manual' method with straight debian.

# NFS exports

This section describes how to setup an nfs v4.2 server with optimal networking and other settings. I am able to get line rate (about 95Megaytes/sec) with a $54.00 hc2 from hardkernel. I am not going to describe everything, but the highlights.

1. Get yourself the 'armbian' variant for this board (you could of course, use ANY computer capable of running a linux server)
2. Get yourself a nice fast SSD of 512GB or 1TB and format with f2fs
3. Install the nfs utils and disable Portmapper

```bash
apt-get update
apt-get install nfs-kernel-server
portmap
systemctl mask rpcbind.socket
systemctl mask rpcbind.service
```

4. I found an EVEN BETTER way to get rid of all the nasty daemons from [this link](https://peteris.rocks/blog/nfs4-single-port/). The file is /etc/default/nfs-kernel-server.

```ini
RPCNFSDCOUNT=8

# Runtime priority of server (see nice(1))
RPCNFSDPRIORITY=0

# Options for rpc.mountd.
# If you have a port-based firewall, you might want to set up
# a fixed port here using the --port option. For more information,
# see rpc.mountd(8) or http://wiki.debian.org/SecuringNFS
# To disable NFSv4 on the server, specify '--no-nfs-version 4' here
RPCMOUNTDOPTS="--no-nfs-version 2 --no-nfs-version 3 --nfs-version 4 --no-udp"
RPCNFSDOPTS="--no-nfs-version 2 --no-nfs-version 3 --nfs-version 4 --no-udp"

# Do you want to start the svcgssd daemon? It is only required for Kerberos
# exports. Valid alternatives are "yes" and "no"; the default is "no".
NEED_SVCGSSD=""

# Options for rpc.svcgssd.
RPCSVCGSSDOPTS=""
```

The important bits are the 'RPCMOUNTD opts'. When we do this we only have to worry about port 2049 no nasty mountd or nfslock crap. Very nice.  This file is ```/etc/common/nfs-common```

5. This one may or may not be necessary, adding in case world ends:

```ini
# If you do not set values for the NEED_ options, they will be attempted
# autodetected; this should be sufficient for most people. Valid alternatives
# for the NEED_ options are "yes" and "no".

# Do you want to start the statd daemon? It is not needed for NFSv4.
NEED_STATD="no"

# Options for rpc.statd.
#   Should rpc.statd listen on a specific port? This is especially useful
#   when you have a port-based firewall. To use a fixed port, set this
#   this variable to a statd argument like: "--port 4000 --outgoing-port 4001".
#   For more information, see rpc.statd(8) or http://wiki.debian.org/SecuringNFS
STATDOPTS=

# Do you want to start the idmapd daemon? It is only needed for NFSv4.
NEED_IDMAPD="yes"

# Do you want to start the gssd daemon? It is required for Kerberos mounts.
NEED_GSSD=
RPCNFSDOPTS="-N 2 -N 3"
RPCMOUNTDOPTS="--manage-gids -N 2 -N 3"
```

After I do this I am getting 95 Megabytes/sec on $59.00 hardware. Its really quite impressive. I may or may not add kerberos authentication at some point, but I am relatively happy with this.

## Sysctl Settings for better network performance

In order to get better network performance, its important to increase the size of our recieve and send buffers for nfs:

```ini
net.core.wmem_max=262144
net.core.rmem_max=262144
net.core.wmem_default=26144
net.core.rmem_default=26144
```

Simply cat these values to /etc/sysctl.conf and reboot.

## Ansible Scripts

The ansible scripts are basically set to fix a lot of this automatically. They are mostly for the 'cpu' issues mentioed. Here are approxymate purposes of each, and explanation of directory structure.

```ini
docs=Documentation
nfs=nfs server setup
scripts=ansible scripts
scripts/binary = k0s binary for creating our cluster
```

### Ansible Scripts brief description

```ini
fixcpucopy.yml= fixes the cpu issues by changing defualt governor to on demand and changes the threshold to 10% if cpu is busy, also copies the k0s binary

inventory.file= inventory file for ansible, populate appropriate node ip / names here. I assume you have working DNS. Node ips necessary for 'groupings'

pi1/2/3/4.yml = change hostnames as appropiate pi4m is the 'conductor node'

fixcpu.sh.start = rc script to 'fix' ondemand governor.
```

### Ansible setup for proper script invocation

Need to download k0s current, and put in the 'scripts/binary' with name k-s. Be sure to get the arm64 version. Also for a sightly smaller binary (about 40 megs or so) you can `upx --lzma k0s`. UPx is the 'ultimate packer for executables' its completely optional.

### Ansible invocation for 'fixing' cpu issues/copying

```bash
ansible-playbook -i inventory.file fixcpucopyk0s.yml -K
```

This lets you specify the 'sudo' password.

## SSH setup for correct ansible funtioning

You will need to setup passwordless ssh access via modifying the `authorized_keys` file in appropriate users directory. Also this user will need sudo permission. 

## Host Naming Convention

```ini
1.2.3.4 pi1 pi1.your.domain
1.2.3.5 pi2 pi2.your.domain
1.2.3.6 pi3 pi3.your.domain
1.2.3.7 pi4m pi4m.your.domain
```

First three are worker nodes, pi4m is the main node. It does the orchestration etc. I generally make sure my DNS is working with these, but this will make sure the ansible scripts are happy.

### Purpose of "NFS" server

This serves as the 'storage' option inside of kubernetes using the 'nfs' driver. I use nfsv4 because its WAY faster than the sd cards (and doesn't wear them down as fast!). It also separates my 'data' from my 'system'. I use a hardkernel HC2 running Arabian.

## Final Provisioning:

Once everything is done, simply follow [this](https://docs.k0sproject.io/latest/install/) skip the 'download' part since you have already done this with the 'ansible' prepare script. I like having 'everything' done w/o running curl | sh as root. It scares me (I am an old security nerd, so I just can't do that!)

## Some 'shortcuts'

If you aren't fanatical about filesystem performance, you can skip the f2fs conversion. I just like it because it makes 'life' in system 'seem faster' since its an atomic file (non journaling) filesystem. 

Be sure to 'set' and not forget your root password. Also create a 'non' privileged user who can login as 'non root' and then sudo/doas to perform privileged ops.

Copy your 'authorized' public ssh key to the user you create PRIOR to cloning! This will save setup 4 times! Make sure your user has 'sudo' permissions to become 'root'.

Fix your 'ssh' so its securly configured, I use [ssh-audit](https://github.com/jtesta/ssh-audit) to do this.  You also might want to enable 'auto patching' to keep your system up to date.

```bash
#!/bin/sh

# SPDX-License-Identifier: GPL-2.0-only
#
# cron job for automatic software updates
# Copyright (c) 2014 Kaarle Ritvanen

set -eu

sleep $(expr $RANDOM % 7200)
exec apk -U upgrade
```

- Put this file in /etc/periodic/daily
- Reboot once per week:

```bash
#!/usr/bin/env sh
/sbin/reboot

```

- put in /etc/periodic/weekly
- This is also in the ansible. More of reference if anyone is interested.

# Hardware list

- [ ] [Rack for Pi4](https://www.amazon.com/Raspberry-Rack-Mount-inch-Units/dp/B085LQT67P/ref=sr_1_6?dchild=1&keywords=pi+4+rack&qid=1615515453&sr=8-6)
- [ ] Cables for pi4 [hdmi](https://www.amazon.com/Panel-Connector-Detachable-Cable-Micro-HDMI/dp/B08FMVMR1S/ref=sr_1_1?dchild=1&keywords=B08FMVMR1S&qid=1615515532&sr=8-1) [lan passthrough](https://www.amazon.com/VANDESAIL-Keystone-Coupler-Female-20Pack/dp/B07KK5CSJY/ref=pd_rhf_se_s_sspa_dk_rhf_search_pt_sub_0_6/134-7856411-9046734?_encoding=UTF8&pd_rd_i=B07KK5CSJY&pd_rd_r=30aae818-b945-4e5e-9d17-f8511b5e8681&pd_rd_w=R04LH&pd_rd_wg=j91In&pf_rd_p=43fcab85-1376-4307-ada1-0576b19ad47e&pf_rd_r=HV6NER76RCKA0S12H33Q&psc=1&refRID=HV6NER76RCKA0S12H33Q&spLa=ZW5jcnlwdGVkUXVhbGlmaWVyPUEyQkhCNkhSQjIxVlZLJmVuY3J5cHRlZElkPUEwMTU0NTAyR1RTUUxFUURINUgwJmVuY3J5cHRlZEFkSWQ9QTA5MzIxOTYxSVdIOTNYQ1kyTElPJndpZGdldE5hbWU9c3BfcmhmX3NlYXJjaCZhY3Rpb249Y2xpY2tSZWRpcmVjdCZkb05vdExvZ0NsaWNrPXRydWU=)
- [ ] [4x POE Hats for pi4](https://www.canakit.com/raspberry-pi-poe-hat.html?cid=usd&src=raspberrypi)
- [ ] [3x PI4 with 8gb (worker)](https://www.canakit.com/raspberry-pi-4-8gb.html)
- [ ] [1x pi4 with 4gb (boss)](https://www.canakit.com/raspberry-pi-4-4gb.html)
- [ ] [cheap 8 port rack-mountable kvm switch](https://www.amazon.com/TESmart-Enterprise-Control-Rackmount-Keyboard/dp/B06XKXTV5T/ref=sr_1_6?dchild=1&keywords=TESmart+8+Port+KVM+HDMI+Switch+Switcher+Box+Support+4K+Rack+Ears+Standard+1U&qid=1615516081&sr=8-6)
- [ ] [Nice managed switch with POE](https://www.provantage.com/hpe-jl683a-aba~7HEWN64X.htm) (and after fan replacement quiet!)
- [ ] [Hardkernel HC2 nfs server](https://ameridroid.com/products/odroid-hc2?_pos=1&_sid=9e9015d6f&_ss=r)
- [ ] [good sd card](https://www.bhphotovideo.com/c/product/1375051-REG/sandisk_sdsqxaf_032g_gn6ma_extreme_microsd_32gb.html?sts=pi&pim=Y)
- [ ] [sd card 'usb-c/usb-3 reader/writer'](https://www.amazon.com/gp/product/B07416LQVM/ref=ppx_yo_dt_b_asin_title_o04_s00?ie=UTF8&psc=1)
- [ ] [good burning program (light)](https://gitlab.com/bztsrc/usbimager/-/tree/master)
- [ ] [good burning program (fat)](https://www.balena.io/etcher/)



