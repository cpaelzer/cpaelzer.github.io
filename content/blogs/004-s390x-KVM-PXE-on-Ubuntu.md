---
title: "s390x KVM PXE on Ubuntu"
date: 2018-10-22T06:14:29+02:00
tags: ["s390x", "pxe", "qemu"]
draft: false
---

# s390x KVM PXE on Ubuntu #
## a.k.a behave like others just once ##

### Background on s390x PXE booting ###

[PXE booting](https://www.syslinux.org/wiki/index.php?title=PXELINUX) is nothing
new but so far to do so on s390x you had to jump quite a few [extra hoops](http://ubuntu-on-big-iron.blogspot.com/2017/12/pxe-netboot-kvm-s390x.html).

On x86 you had the package `pxelinux` that would provide a full featured pxlinux to boot.
You'd instruct a system to fetch this and it would implement all of the [PXE Linux configuration](https://www.syslinux.org/wiki/index.php?title=PXELINUX#Configuration)
from there.

But being x86 code that would not have worked on s390x. Due to that there initially was a way to [create your own
initial blob](https://github.com/ibm-s390-tools/s390-tools/tree/master/netboot)
that would consist of a kernel, an initrd and scripts to implement the very basics of PXE.
But not only did that miss extended PXE features, it also had a kernel/initrd bundled.
If you wanted to use it for something else you'd have to regenerate that blob.
And in general users/admins were required to create such a blob for themselves.

[PPC64LE](https://www.ubuntu.com/download/server/power) instead had all the PXE
features implemented via it's bios called [SLOF](https://github.com/aik/SLOF/).

In an effort to get more PXE features to s390x they reused some of that code and
also bundled it with their ROM. This is actually part of [qemu 3.0](https://wiki.qemu.org/ChangeLog/3.0#s390_firmware)
but to make it more usable for users and automation software it was
[SRU'ed](https://wiki.ubuntu.com/StableReleaseUpdates)
to [Bionic and Cosmic](https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1790901).
Due to that it is available on the last [Ubuntu LTS](https://wiki.ubuntu.com/LTS)
starting with qemu version `1:2.11+dfsg-1ubuntu7.7`

On one hand that lets you skip most of the [steps](http://ubuntu-on-big-iron.blogspot.com/2017/12/pxe-netboot-kvm-s390x.html)
that were required before and on the other hand you will gain more PXE
configuration features being supported.

### Step I - prepare netboot resources ##

There are plenty of ways to [install on s390x](https://wiki.ubuntu.com/S390X#Installation).
All of them usually seem odd to non System z people.
So lets add PXE based install into KVM as something that looks more familiar.

There is already a [s390x d-i image](http://ports.ubuntu.com/ubuntu-ports/dists/bionic/main/installer-s390x/current/images/generic/)
that can be used on [LPAR installations](https://wiki.ubuntu.com/S390X/Installation%20In%20LPAR).
And like the `ubuntu.ins` file there would tell the duo of
[LPAR](https://en.wikipedia.org/wiki/Logical_partition) and
[HMC](https://en.wikipedia.org/wiki/IBM_Hardware_Management_Console) how to load
that into memory, we will have the
[PXE config](https://www.syslinux.org/wiki/index.php?title=PXELINUX#Configuration)
do something very similar in our case.

The following provides all we need for a basic netboot implementing a
[PXE configuration file](https://www.syslinux.org/wiki/index.php?title=Config).

```shell
$ sudo mkdir -p /srv/tftp/pxelinux.cfg
$ cat << EOF | sudo tee /srv/tftp/pxelinux.cfg/default
label ubuntu
   kernel kernel.ubuntu
   initrd initrd.ubuntu
EOF
```

Then make the netboot kernel and initrd available in your tftp directory

```shell
$ sudo wget -P /srv/tftp http://ports.ubuntu.com/ubuntu-ports/dists/bionic/main/installer-s390x/current/images/generic/kernel.ubuntu
$ sudo wget -P /srv/tftp http://ports.ubuntu.com/ubuntu-ports/dists/bionic/main/installer-s390x/current/images/generic/initrd.ubuntu
```

### Step II - prepare a guest ###

I usually recommend using KVM through libvirt for most cases.
Actually most of the time through [uvtool](https://help.ubuntu.com/lts/serverguide/cloud-images-and-uvtool.html)
which helps you to work with [cloud images](https://cloud-images.ubuntu.com/) easily.

We don't need uvtool today (strictly speaking we don't need libvirt either, but it makes things more readable and easier), so just get libvirt and qemu-kvm.

```shell
$ sudo apt install -qq -y libvirt-daemon-system qemu-kvm
```

We have to tell the guest that it is supposed to netboot, the change diverging from a usual KVM Guest configuration is:

```xml
<boot dev='network'/>
```

And we also want to give it a new virtual disk to work on, so let us create one:

```shell
$ sudo qemu-img create -f qcow2 /var/lib/libvirt/images/blog-netboot.qcow2 8G
Formatting '/var/lib/libvirt/images/blog-netboot.qcow2', fmt=qcow2 size=8589934592 cluster_size=65536 lazy_refcounts=off refcount_bits=16
```

This disk is configured the usual way following the [libvirt domain format](https://libvirt.org/formatdomain.html).

An example of a full guest XML definition for this looks like [this](/004-guest.xml)

You can now define your guest from that XML with virsh:

```shell
$ virsh define blog-netboot.xml
```

### Step III - prepare guest network ###

Libvirt can provide the tftp service for you through the dnsmasq daemon that it will spawn anyway.
To do so add a snippet like the following to your default network configuration.

```xml
<tftp root='/srv/tftp'/>
```

Afterwards you will need to destroy and start your network to pick up those changes.
Warning: this will kill networking to any guests currently active on that network!

```shell
$ virsh net-destroy default
# Add the tftp line
$ virsh net-edit default
$ virsh net-start default
```

If you want to check, you will see the options `enable-tftp` and `tftp-root=/srv/tftp` in the dnsmasq config file `/var/lib/libvirt/dnsmasq/default.conf` due to that.

A full XML for an example default network would then look like [this](/004-network.xml)

### Step IV - start your guest ###

Now all you need is set up, you have:

- a netboot kernel/initrd to serve
- a libvirt network definition serving those files
- a PXE configuration that will make use of it
- a libvirt guest definition doing netboot

To see the early boot actions start it with --console.
```shell
$ virsh start --console blog-netboot
Domain blog-netboot started
Connected to domain blog-netboot
Escape character is ^]
done
  Using IPv4 address: 192.168.122.54
  Using TFTP server: 192.168.122.1
Trying pxelinux.cfg files...
  Receiving data:  0 KBytes
  TFTP: Received pxelinux.cfg/default (61 bytes)
Loading pxelinux.cfg entry 'ubuntu'
  Receiving data:  4092 KBytes
  TFTP: Received kernel.ubuntu (4092 KBytes)
  Receiving data:  13354 KBytes
  TFTP: Received initrd.ubuntu (13354 KBytes)
Network loading done, starting kernel...

[    0.437599] Linux version 4.18.0-10-generic (buildd@bos02-s390x-011) (gcc version 8.2.0 (Ubuntu 8.2.0-7ubuntu1)) #11-Ubuntu SMP Thu Oct 11 15:06:06 UTC 2018 (Ubuntu 4.18.0-10.11-generic 4.18.12)
[    0.437606] setup.289988: Linux is running under KVM in 64-bit mode
```

That is it, working fine following common PXE specifications to configure and use it.
From here it depends on your actual needs which way you want to go further with this.

### Outlook and next steps ###

To boot the [netboot installer](http://ports.ubuntu.com/ubuntu-ports/dists/bionic/main/installer-s390x/current/images/generic/) was just a nice example and quite close to compare it to older instructions.
But I recommend to become more cloudy and consider booting cloud images like that.
You can do so ephemeral or with a config that makes it install to an attached disk.
And this is not only a great use case, it also is [easy](https://tutorials.ubuntu.com/tutorial/create-kvm-pods-with-maas#0)
and already available via [MAAS KVM Pods](https://blog.ubuntu.com/2017/10/18/maas-kvm-pods).

I'm happy that backporting the changes to the [qemu 2.11](https://wiki.qemu.org/ChangeLog/2.11)
that is in [Ubuntu 18.04](http://releases.ubuntu.com/18.04/) will make this
available to all our [Ubuntu LTS](https://wiki.ubuntu.com/LTS) users.

### Further References: ###

* Old style upstream s390x KVM [netboot](https://github.com/ibm-s390-tools/s390-tools/tree/master/netboot)
* Old style Ubuntu s390x KVM [netboot](http://ubuntu-on-big-iron.blogspot.com/2017/12/pxe-netboot-kvm-s390x.html)
* Upstream Syslinux page about [PXE](https://www.syslinux.org/wiki/index.php?title=PXELINUX)
* Ubuntu community on [netboot](https://help.ubuntu.com/community/Installation/QuickNetboot) in general
