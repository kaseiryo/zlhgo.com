---
title: "Building Readonly OpenBSD System"
date: 2022-01-19T11:20:26+09:00
draft: false
---

This is how I set up a readonly OpenBSD 6.8 system.

The system is a router running off an SD card. I want to minimise wear on the SD card, and I want the system to come up clean after a power failure. For times when I need to make a quick tweak, I can still go read-write temporarily.

This is unfortunately not a supported OpenBSD setup, and small tweak to /etc/rc is necessary.

## Overview

The technique here is the same as some others (for example http://www.volkerroth.com/tecn-obsd-diskless.html). Root is mounted readonly, and filesystems like `/dev, /var and /tmp` are mounted as mount_mfs(8) memory filesystems.

mount_mfs(8) has a nice feature to build the memory filesystem by copying the contents of an existing directory. So we build /var and /dev from the existing on-disk directories. /tmp is created empty.

We maintain two versions of /etc/fstab. One version has entries for the readonly system with mfs filesystems, and the other version has the normal read-write entries. There is a script to put the appropriate version in place before the next reboot.

If the system is running readonly, you can temporarily remount root read-write to make changes.

## Think about

You could probably convert an existing OpenBSD systems to readonly, but consider:

*the system will enable an initial swap partition even if it is not in fstab
*bookkeeping is easier with fewer filesystems

The first point means you cannot stop the system from enabling a swap partition just by removing it from /etc/fstab. Not what you want if you want a readonly SD card. There might be a way to disable this behaviour, but it was easier for me to avoid creating a swap partition in the first place.

The second point has to do with managing entries in /etc/fstab. With the readonly system described here, two different versions of /etc/fstab must be maintained. It’s just easier if you only have a few entries to keep in sync.

## Procedure

First do a normal install onto the SD card, but partition it as a single filesystem with no swap. Install packages, run syspatch(8), and configure things as normal.

(If your system is so small that it wouldn’t be able to relink the kernel without swap, you might be able to build the SD card on other hardware.)

Then follow these steps:

## Move /var/syspatch

All of /var will end up in ram, so we don’t want the contents of /var/syspatch to be included.

~~~
# mv /var/syspatch /var-syspatch
# ln -s /var-syspatch /var/syspatch
~~~

## Disable library and kernel relinking

ASLR and KARL won’t work with a readonly root, so they need to be disabled:

~~~
# rcctl disable library_aslr
# chmod -x /usr/libexec/reorder_kernel
~~~

## Install /root/bootmode.sh

bootmode.sh is a script to safely copy an fstab into place before a reboot. download

For example, if you need to do something that requires read-write, you could do this:

~~~
# /root/bootmode.sh rw
temporarily remounting / rw
remounting / ro
# reboot
~~~

When the system comes up, the filesystems will be normal read-write.

As seen in the example above, if the system is currently readonly, root has to be remounted read-write temporarily in order to install a new fstab. The script will try to restore root to read-only if it had to remount. If the mount was already read-write, the script won’t change it.

After you have made your change, you can restore the readonly fstab like this:

~~~
# /root/bootmode.sh ro
# reboot
~~~

Copy bootmode.sh to a convenient location, and don’t forget to chmod +x.

Test it like this:

~~~
# /root/bootmode.sh
filesystem: rw
fstab: 
~~~

It correctly detects the current mount mode, but we haven’t configured any fstabs yet.

## Create /etc/fstab.rw

You want two versions of fstab: one normal read-write, and one for readonly.

These are named /etc/fstab.rw and /etc/fstab.ro. bootmode.sh copies the appropriate one into place.

To avoid unnecessary copies, we add a special identifying comment to each file. bootmode.sh looks at this comment.

Create /etc/fstab.rw by adding the comment:

~~~
# echo '#bootmode rw' > /etc/fstab.new
# cat /etc/fstab >> /etc/fstab.new
# mv /etc/fstab.new /etc/fstab.rw
~~~

So my /etc/fstab.rw looks like this now:

~~~
# cat /etc/fstab.rw
#bootmode rw
f30b29e96a3044f5.a / ffs rw,wxallowed,noatime 1 1
~~~

And bootmode.sh will install it:

~~~
# /root/bootmode.sh rw
# /root/bootmode.sh
filesystem: rw
fstab: rw
~~~

## Create /etc/fstab.ro

My readonly fstab looks like this:

~~~
#bootmode ro
f30b29e96a3044f5.a / ffs ro,wxallowed 1 1
swap /tmp mfs rw,nodev,-s10m 0 0
swap /var mfs rw,nodev,nosuid,-s10m,-P=/var 0 0
swap /dev mfs rw,nosuid,noexec,-s4m,-i128,-P=/dev 0 0
~~~

Root is obviously marked readonly.

/tmp starts out as an empty 10MB filesystem.

/var is also 10MB, and starts with the contents of /var on disk.

/dev is similar, but note the -i128 value.

## Patch /etc/rc

/etc/rc always mounts root read-write. This is seen around line 379:

~~~
# Re-mount the root filesystem read/writeable. (root on nfs requires this,
# others aren't hurt.)
mount -uw /
chmod og-rwx /bsd
ln -fh /bsd /bsd.booted
~~~

Since there is no configurable way to change that, we need to make some small changes before and after those lines. First save the original:

~~~
# cp /etc/rc /etc/rc.ORIG
~~~

Then change that section of /etc/rc to look like this:

~~~
# Re-mount the root filesystem read/writeable. (root on nfs requires this,
# others aren't hurt.)
case `awk '$1 == "#bootmode" {print $2}' < /etc/fstab` in
rw)
mount -uw /
chmod og-rwx /bsd
ln -fh /bsd /bsd.booted
;;
esac
~~~

This will ensure root will only be mounted read-write if /etc/fstab has the special comment indicating read-write.

## Usage

After the above steps are done, you can reboot readonly:

~~~
# /root/bootmode.sh ro
# reboot
~~~

When the system comes up, you will see messages like this:

~~~
pax: /tmp/mntrL5pZoPXL3/./run/ntpd.sock skipped. Sockets cannot be copied or extracted
~~~

and

~~~
dd: /etc/random.seed: Read-only file system
chmod: /etc/random.seed: Read-only file system
rm: /etc/nologin: Read-only file system
~~~

and

~~~
/etc/rc[624]: /usr/libexec/reorder_kernel: cannot execute - Permission denied
~~~

That’s all ok.

If you botched /etc/rc you can take out your SD card or boot disk and mount it on another OpenBSD system and fix it.

You can verify the state:

~~~
# /root/bootmode.sh
filesystem: ro
fstab: ro
~~~

If you want to go back to read-write:

~~~
# /root/bootmode.sh
# reboot
~~~

To temporarily mount read-write for minor changes:

~~~
# mount -uw /
# touch /etc/junk
# mount -ur /
# rm /etc/junk
rm: /etc/junk: Read-only file system
~~~

It is best to keep the read-write time period relatively short to avoid the chances of some process latching onto the filesystem and preventing remounting. For example, you might see this:

~~~
# mount -ur /
mount_ffs: /dev/sd0a on /: Device busy
~~~

That means somebody has a file open for writing on that filesystem. You’ll have to find the process and kill it, or else reboot if you really want to go back to readonly.

