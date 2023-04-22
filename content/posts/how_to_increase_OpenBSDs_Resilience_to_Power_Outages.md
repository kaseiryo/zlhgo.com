---
title: "How to increase OpenBSDs Resilience to Power Outages"
date: 2022-01-19T12:34:05+09:00
draft: false
---

Most of the OpenBSD systems I am in charge of are deployed in data centres, powered by UPSs which provide them with electrical power during periods of public grid power outages. But there is also a number of OpenBSD systems I administer, which are deployed in much less favourable conditions; where frequent power outages last longer than UPS batteries do, or where there are no UPSs at all (such as branch office routers in godforsaken places where having electricity and Internet access at all is considered a lucky circumstance). These latter systems are likely to have high rate of unclean shutdowns caused by prolonged or unexpected power outages, which in turn increase the probability of their inability to boot without human intervention. This article describes steps to make OpenBSD system more resilient to unexpected power outages by minimising the possibility of inconsistent file systems after unclean shutdowns, which is achieved by mounting all disk partitions in read-only mode. Filesystems which have to be writable - `/var, /dev and /tmp` - are mounted as writable memory file systems.

## A few words of caution

This is "works for me" kind of tutorial. Although I use described setup on more than 30 production instances, that doesn't mean it will suit your purpose as well. Primary goal of this article is to educate readers about some of the less-known functionalities of OpenBSD and deepen their understanding of underlying components so they can become more creative with their setups. Keep in mind that - talking in LEGO terms - being a master builder who doesn't rely on standard templates means you're mostly on your own when something goes bad with your design. Keeping `/var` in memory file system has been characterized as "wierd tweak" by Theo de Raadt, founder and leader of OpenBSD. He also said "we are not writing software for your crazy tweak". Truth to be told, I don't take any offense in this, as both statements are correct. Quite the contrary, I am flattered to be recognized by OpenBSD leader as someone who does wierd and crazy tweaks.

Now, some OpenBSD developers are more supportive to non-standard setups - for example my question on (AFAIK unofficial) #OpenBSD IRC channel regarding tcpdump(8) not being able to start with read-only `/` filesystem has been taken into account and identified as unveil(2) bug. This lead to kernel patch which fixed the problem in OpenBSD 6.6. Thanks (I guess) @semarie, @beck, possibly others, who did not dismiss this with "read-only `/` is not supported". On the other hand, some OpenBSD developers are less supportive to non-standard setups. For example, when informed that syspatch(8) has problem patching stuff in `/var` mounted as memory file system, its creator and maintainer @ajacoutot replied he "will make that check stronger so that syspatch fails right away on MFS-mounted /var" (no worries, I have already worked around this as you will be able to read below).

Taking all above into account, please think twice before asking OpenBSD developers and users for help when using setup described in this article. Keep in mind that number of people using read-only `/` and `/var` on `mfs` is insignificant compared to whole OpenBSD user base. Problems you are encountering could be, and most probably are, problems with your way of using OpenBSD, and not problems with OpenBSD itself. This doesn't mean you must never ask anything on OpenBSD mailing lists if you use described setup, only that it would be fair to declare which on-disk file systems you keep read-only, and which ones you mount as memory file systems.

## Hardware and software used

I use PC Engines apu4c4 router board because of the following characteristics:

* It features 4 GbE ports which is ideal for my branch offices - two ports are for two different ISPs, third port for local LAN, fourth for pfsync(4) interface in high availability setups which include carp(4).
* Its CPU - 1 GHz quad Jaguar core with 64 bit and AES-NI support - does good job encrypting IPSEC flows
* It has 4Gb of RAM, which is more than enough for basic routing and firewalling tasks, but also for memory file systems

As for local SD card storage, I use 32Gb SanDisk Extreme PRO. It's relatively pricey, but my experiments with lower quality SD cards resulted in prolonged branch office downtime and 500 kilometer trip, which in the end proved much more expensive than initial savings.

I am connecting to PC Engines apu4c4's serial interface from ThinkPad T440 via DIGITUS USB 2.0 serial adapter, which runs FreeBSD 12.1-RELEASE as main OS (although I sometimes boot to OpenBSD as well as - yes, I confess - Windows 10 thanks to GPT, UEFI and ReFind). My primary desktop environment is Xfce, and I am using GNU Screen to connect to PC Engines apu4c4's serial console.

I use dd to flash install68.img onto any USB flash drive lying around my desk. I connect PC Engines apu4c4's ethernet port closest to the serial port - em0 to my LAN switch so it can obtain IP parameters from ISC DHCP Server, and access resources on the Internet via router running OpenBSD set up by following this article almost to the word.

As of OpenBSD 6.6 I don't need any third party package, as I use openrsync(1) to periodically synchronize memory-based `/var` filesystem to `/altvar` UFS-formatted disk partition.

## Installation

I assume you are already familiar with simple install, so I won't write here about preparation of install media, choosing boot device, setting serial port's baud rate and com port, system hostname, IP parameters, root password and creation of non-privileged system user - you can see these in video tutorial (currently outdated, check back soon for update).

After completion of above install stages, we arrive at the most important part for our read-only setup, which is appropriate manual partitioning. When asked, Allocate (W)hole disk to OpenBSD, choose (C)ustom layout, and partition as follows:

~~~
label   mountpoint      size
a       /                 1g
b       <none>            1g
d       /var              1g
e       /usr              2g
f       /usr/X11R6        1g
g       /usr/local        1g
h       /home             4g
i       /var/syspatch     1g

~~~

Make sure to inspect with p m if you partitioned correctly:

~~~

sd0> p m
OpenBSD area: 64-62332200; size: 30435.6M; free: 18112.4M
#                size           offset  fstype [fsize bsize   cpg]
  a:          1027.6M               64  4.2BSD   2048 16384     1 # /
  b:          1027.6M          2104512    swap
  c:         30436.5M                0  unused
  d:          1027.6M          4209056  4.2BSD   2048 16384     1 # /var
  e:          2055.2M          6313536  4.2BSD   2048 16384     1 # /usr
  f:          1027.6M         10522560  4.2BSD   2048 16384     1 # /usr/X11R6
  g:          1027.6M         12627072  4.2BSD   2048 16384     1 # /usr/local
  h:          4102.5M         14731584  4.2BSD   2048 16384     1 # /home
  i:          1027.6M         23133600  4.2BSD   2048 16384     1 # /var/syspatch

~~~

...before you hit w followed with q to write changes and quit.

After that, continue with installation of sets - at this point I recommend to install all of them.

>For a long time I used to install only basic sets and man set on my network devices, not only to save disk space (SD cards used to be of smaller capacity just a few years ago), but also to minimize possible attack area by reducing software components available on the system to bare minimum. While I am still very supportive of an idea to keep various system components in separate sets where users can nitpick them according to their needs, reality seems a bit more complicated than it appears in theory. For example, a few days ago I wanted to take a look at sendsyslog(2) manpage on one of my routers, but got "no entry in the manual". I know I always install manXX.tgz, so I suspected a bug. To cut a long story short, after a few mails exchanged with @schwarze, an OpenBSD developer who I respect not only for his work on mandoc(1), but also for being a great person, I learned that library manual pages reside in compXX.tgz, not in manXX.tgz. It went against my assumption that man pages reside in man set. But it made sense once I looked at it from different angle - people who hack libraries will usually have compiler set installed on their machines, whereas people who opt to remove compiler set will rarely read about libraries. Anyway, after I heard his opinion that "everything should be in baseXX.tgz. People shooting themselves in the foot by deselecting sets, then getting confused, is a recurring pain...", I decided I'll try to ease some of the OpenBSD developers' pain by installing all the sets :)

Continue to the end of the install and reboot when instructed so, but not before removing install media USB flash disk.

## First boot (still vanilla)

Once system boots up, let's first observe our mountpoints, and free disk space on them:

~~~
robsd66# mount
/dev/sd0a on / type ffs (local)
/dev/sd0h on /home type ffs (local, nodev, nosuid)
/dev/sd0e on /usr type ffs (local, nodev)
/dev/sd0f on /usr/X11R6 type ffs (local, nodev)
/dev/sd0g on /usr/local type ffs (local, nodev, wxallowed)
/dev/sd0d on /var type ffs (local, nodev, nosuid)
/dev/sd0i on /var/syspatch type ffs (local, nodev, nosuid)

robsd66# df -h
Filesystem     Size    Used   Avail Capacity  Mounted on
/dev/sd0a     1008M   96.3M    862M    10%    /
/dev/sd0h      3.9G   18.0K    3.7G     0%    /home
/dev/sd0e      2.0G    955M    964M    50%    /usr
/dev/sd0f     1008M    202M    756M    21%    /usr/X11R6
/dev/sd0g     1008M    218K    958M     0%    /usr/local
/dev/sd0d     1008M    5.9M    952M     1%    /var
/dev/sd0i     1008M    2.0K    958M     0%    /var/syspatch

~~~

At this point, our system is almost identical to one created with automatic partitioning. All the partitions are on-disk, UFS-partitioned, mounted with default options - all of them are writeable, and /usr/local is the only one to have wxallowed flag, which is the main reason why I created it in the first place, although as of OpenBSD 6.6 no files will reside there, thanks to introduction of openrsync(1). Wikipedia has some basic info about W^X.

Main difference between automatic partitioning and our custom partitioning is absence of /tmp (we will use memory file system), /usr/obj/ and /usr/src (we won't be compiling stuff from sources), and introduction of separate /var/syspatch partition in order not to clutter our RAM (packages containing binary patches can consume quite a few megabytes over lifetime of a release).

## The road to madness

If you are - like me - an admirer of H. P. Lovecraft's literature, you will presumably understand why I chose to give such weird name to this section. The setup is being described as wierd and crazy, remember? I have deliberately been not-strictly-technical, in hope to deter readers who seek arid technical solution as opposed to readers who seek (perhaps terrifying) adventure. In any case, don't blame me for any damages, mental or otherwise, you might suffer as a consequence of reading this article, much less following its instruction.

Let's create folder which will serve as mountpoint for on-disk /altvar partition into which contents of mfs-mounted /var will be periodically synchronized:

~~~
robsd66# mkdir /altvar

~~~

We'll now be editing our fstab so that on-disk UFS partition which was originally mounted under /var gets mounted under /altvar. Also, we will introduce three memory file systems - /var, /dev and /tmp, of which /var and /dev are going to be populated from /altvar on-disk UFS partition, and /dev folder from on-disk UFS / partition, respectively. Be sure to give a thorough look at all available options of mount_mfs(8), maybe you'll get even wilder ideas than I got :)

Before editing, it is good idea to backup original fstab:

~~~
# cp /etc/fstab ~/fstab.ORIG
~~~

Here's how my edited fstab looks like:

>Do not copy/paste fstab below - my DUUIDs won't work for you! Lines starting with swap should be ok to copy/paste.

>Order of fstab entries is important - don't declare sub-mountpoint (for example /var/syspatch) above mountpoint into which it mounts (e.g. /var)!

~~~
2a250ffa1d19cd07.a / ffs rw 1 1
2a250ffa1d19cd07.b none swap sw
2a250ffa1d19cd07.d /altvar ffs rw,nodev,nosuid 1 2
2a250ffa1d19cd07.e /usr ffs rw,nodev 1 2
2a250ffa1d19cd07.f /usr/X11R6 ffs rw,nodev 1 2
2a250ffa1d19cd07.g /usr/local ffs rw,wxallowed,nodev 1 2
2a250ffa1d19cd07.h /home ffs rw,nodev,nosuid 1 2

swap /var mfs rw,nodev,nosuid,-s512m,-P=/altvar 0 0
swap /dev mfs rw,nosuid,noexec,-s4m,-P=/dev,-i128 0 0
swap /tmp mfs rw,nodev,noexec,nosuid,-s512m 0 0

2a250ffa1d19cd07.i /var/syspatch ffs rw,nodev,nosuid 1 2

~~~

Let's reboot once again, this time into system which has /var, /dev and /tmp partitions mounted as memory file systems.

## Further down the rabbit hole

Once system is rebooted, let's observe once again our mountpoints and free space on them:

~~~
robsd66# mount
/dev/sd0a on / type ffs (local)
/dev/sd0d on /altvar type ffs (local, nodev, nosuid)
/dev/sd0e on /usr type ffs (local, nodev)
/dev/sd0f on /usr/X11R6 type ffs (local, nodev)
/dev/sd0g on /usr/local type ffs (local, nodev, wxallowed)
/dev/sd0h on /home type ffs (local, nodev, nosuid)
mfs:34347 on /var type mfs (asynchronous, local, nodev, nosuid, size=1048576 512-blocks)
mfs:4302 on /dev type mfs (asynchronous, local, noexec, nosuid, size=8192 512-blocks)
mfs:98547 on /tmp type mfs (asynchronous, local, nodev, noexec, nosuid, size=1048576 512-blocks)
/dev/sd0i on /var/syspatch type ffs (local, nodev, nosuid)

robsd66# df -h
Filesystem     Size    Used   Avail Capacity  Mounted on
/dev/sd0a     1008M   96.3M    862M    10%    /
/dev/sd0d     1008M    5.9M    952M     1%    /altvar
/dev/sd0e      2.0G    955M    964M    50%    /usr
/dev/sd0f     1008M    202M    756M    21%    /usr/X11R6
/dev/sd0g     1008M    218K    958M     0%    /usr/local
/dev/sd0h      3.9G   18.0K    3.7G     0%    /home
mfs:34347      495M    5.7M    464M     1%    /var
mfs:4302       3.4M   32.0K    3.2M     1%    /dev
mfs:98547      495M    5.0K    470M     0%    /tmp
/dev/sd0i     1008M    2.0K    958M     0%    /var/syspatch
~~~

At this stage we are on shaky grounds - our /var filesystem is on a volatile file system, which means we will lose all of the changes made to it since it was initially populated from /altvar. To make things worse, we aren't even resilient to unexpected power outage yet, as all of our on-disk UFS file systems are mounted as writable. Let's start fixing these issues.

Let's do initial sync manually, check if everything went well, and script it if it did.

>Be very careful with trailing slashes in source and destination directories, you could end up with /var synced into /altvar/var instead of mirroring /var content in /altvar!

~~~
# /usr/bin/openrsync --rsync-path=/usr/bin/openrsync -avx --delete /var/ /altvar
Transfer starting: 258 files
backups/etc_passwd.current
cron/log
db/kernel.SHA256
log/messages
Transfer complete: 4.1 KB sent, 1.98 MB read, 8.15 MB file size
~~~

Some of the files on /var partition are quite important. For example, db/kernel.SHA256, which is re-created after every successful reordering of kernel (see KARL). Unless we synchronized the change, important security feature of kernel reordering would fail without manual intervention on next, or any other consequent boots. Same goes for backups/ directory which backs up important config files, package and other databases under db/, logs under log/ etc. Actually, we don't want to lose anything below /var mountpoint, which is why we synchronize all of its contents.

Let's put this command into a /root/syncvar.sh script now, along with some others. As you will see later, our on-disk UFS partitions will be mounted read-only, which prevents writing to /altvar, therefore we will want to temporarily re-mount all of our on-disk partitions as writable. Next, we will want to update our random.seed file so our system gets more enthropy, become more unpredictable to possible attackers, and ultimately get more secure (those lines are copy/paste from random_seed() function in rc). After that we synchronize contents of /var to /syncvar and finally we re-mount our on-disk partitions as read-only. Here's how that looks in a shell:

~~~
#!/bin/sh

/sbin/mount -uwA -t nomfs
dd if=/var/db/host.random of=/dev/random bs=65536 count=1 status=none
chmod 600 /var/db/host.random
dd if=/dev/random of=/var/db/host.random bs=65536 count=1 status=none
dd if=/dev/random of=/etc/random.seed bs=512 count=1 status=none
chmod 600 /etc/random.seed
/usr/bin/openrsync --rsync-path=/usr/bin/openrsync -avx --delete /var/ /altvar
/sbin/mount -urA -t nomfs
~~~

We will sometimes want to execute above script interactively from command line, and in order to do it we need to make it executable:

~~~
robsd66# chmod 755 /root/syncvar.sh
~~~

Above script should be invoked periodically - not too often because of SD card wear and vulnerability of the system to unclean shutdowns during the time of its running, but also not too rarely as we want to retain as fresh information from logs as possible. Running it daily, at midnight, from cron, and also on intended shutdowns and reboots seems OK, so we append the following to root's crontab:

@daily /root/syncvar.sh >/dev/null 2>&1

We also create rc.shutdown file with the following contents:

~~~
#!/bin/sh

/root/syncvar.sh
~~~

Last, but not the least, we will add custom function to rc.local which will wait for kernel to get reodered, and once it is done, synchronize mfs-mounted /var to on-disk UFS-formatted /altvar, and mount on-disk file systems in read-only mode.

~~~

mount_ro() {
  while sleep 5; ([ -n "$(pgrep -f 'reorder_kernel')" ]); do done
  /usr/bin/openrsync --rsync-path=/usr/bin/openrsync -avx --delete /var/ /altvar
  /sbin/mount -urA -t nomfs
}

mount_ro &
~~~

>I was initially modifying rc directly in order to achieve late mount of disk file systems in read-only mode, which is a terrible idea. I asked around on @misc mailing list and got some answers publicly, however the above nice solution was forwarded to me off-list, by another list member who also obtained it off-list. I will be more than happy to give credit here, just drop me a line if you are an author of above solution.

We can now reboot into OpenBSD system which is hopefully much more resilient to unexpected power outages without sacrificing any of its security or functionality, assuming we don't use additional third party packages.

>In order to make changes to files residing on disk partitions you need to mount them in writeable mode with mount -uwA -t nomfs.

>Mount disk partitions back to read-only mode with mount -urA -t nomfs afterwards.

## syspatch and sysupgrade

In order to patch and upgrade our resilient OpenBSD systems with syspatch(8) and sysupgrade, we need to revert temporarily to "vanilla" installation.

Before any patching or upgrading, we will simply activate our original fstab, and move rc.shutdown and rc.local scripts away from /etc:

~~~
# cp /etc/fstab ~/fstab.MFS
# cp ~/fstab.ORIG /etc/fstab
# mv /etc/rc.shutdown ~/
# mv /etc/rc.local ~/
~~~

After reboot our system will have all the partitions mounted from disk in writable mode, with all the modifications for synchronization and read-only mounting disabled. It is now safe to proceed with patching or upgrading as usual.

System upgrades, as well as most system patches (those that touch kernel) require reboot. Only after reboot to standard, non-reslient system will we make it "resilient" again. This will require two stages. In first stage we will only activate "resilient" fstab:

~~~
# cp ~/fstab.MFS /etc/fstab
# reboot
~~~

Reboot is needed so that /dev, /tmp and /var partitions are mounted as memory file system.

After reboot, /dev, /tmp and /var partitions are mounted as memory file system, but all the on-disk partitions are still writable, and procedures for syncing changes to /altvar on clean shutdowns and reboots are not yet active. In order to activate them we just need to copy appropriate scripts, which we backed up to root's home directory earlyer, back to /etc, and reboot:

~~~
# cp ~/rc.shutdown /etc/
# cp ~/rc.local /etc/
# reboot
~~~

If everything went well, after reboot we are once again in "resilient" mode.

