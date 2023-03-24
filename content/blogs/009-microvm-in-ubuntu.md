---
title: "Microvm, qboot and feature reduced qemu in Ubuntu"
date: 2020-07-01T12:08:27+02:00
tags: ["kvm", "Ubuntu", "qemu", "microvm", "bios", "performance"]
draft: false
---

# "Microvm, qboot and feature reduced qemu in Ubuntu"
## a.k.a Throw everything over board, we need to get faster ##

QEMU/KVM is a very powerful, complex and feature rich software stack for
virtualization. But recently more and more people ask for virtualization style
isolation (compared to containers), but at the same time want it to be as
similar as possible to containers in terms of speedy initialization.

For these use cases a generic QEMU sometimes is considered "too fat" and
thereby ideas came up that a stripped-to-the-use-case QEMU most likely will be
as fast as some emerging competitors while at the same time be already mature
and compatible with a lot of other bits in the virtualization ecosystem.

To make that available to Ubuntu users there now is an alternative executable
for QEMU: `qemu-system-x86_64-microvm`.
You can get that with:

```shell
$ sudo apt install qemu-system-x86-microvm
```

# Inhibitors

Let us break down what usually consumes time when starting a guest:

* code: finding, loading and mapping of dynamic libraries; many features to initialize and check config for
* virtual-machine: plenty of virtual hardware to prepare, present and initialize
* bios: initializing all the (virtual) hardware
* boot: guest detecting and loading the kernel/initrd

The later coming virtio-fs will allow to make this even more container-like by
providing a modern and efficient way to share file system content.

Other than startup time (speed) there also can be less overhead due to the
reduced virtual-HW (efficiency) and size. The latter comes in two ways, for
memory/CPU footprint (guest density) as well as the active code base (smaller
security relevant profile).

In the following sections I'll outline what was done to speed up execution.

## Code-load

Before a program can even influence it's execution time it has to start. And
when starting your code doesn't run right away.
Dynamic linked shared objects need to be loaded and mapped as well before you
can start. If you want to optimize for the last few milliseconds you need to
"load less" which maps to disabling unwanted features.
This skips the loading, mapping as well as the initialization of these extra
features.

The features disabled in the `qemu-system-x86_64-microvm` binary are as
recommended on the [qboot](https://github.com/bonzini/qboot) page by QEMU
maintainer Paolo Bonzini.

## Virtual-machine startup

The common i440fx or q35 types try to resemble real computers. But there is
no strict need for that.
[microvm](https://github.com/bonzini/qemu/blob/master/docs/microvm.rst) is a
machine type inspired by [firecracker](https://firecracker-microvm.github.io/)
and constructed after its machine model - without PCI or ACPI, designed for
short lived guests.

That allows a minimalistic machine which reduces the amount of things the
hypervisor has to set up and emulate. Thereby the initialization phase in the
host gets faster, but also the guest kernel has less things to probe and
enumerate when booting again saving startup time.

## Bios

Just like the hypervisor prepares all virtual hardware the firmware usually
has to do a lot of enumeration, setup and initialization before the guest runs.
But just like with the `microvm` type the usual bios ROM's are meant to behave
like real computers.

By adopting a restricted feature set to just boot Linux one can optimize this
as well. [qboot](https://github.com/bonzini/qboot) does that and is available
as `/usr/share/qemu/bios-microvm.bin` provided by the package
`qemu-system-data` since Ubuntu Focal.

## Boot

A lot of time in a boot process is usually consumed by reading and processing a
myriad of bootloader specifications.
Therefore almost all implementations for fast startup switch to an external
kernel to load.
After all this is inspired by container-like workloads which have no control
of the kernel either.
Providing the kernel/initrd is something QEMU already can do, but in these use
cases it became the default way to run guests.

# In Action

Overall you could call QEMU like this to get all of the above:

```plaintext
$ sudo qemu-system-x86_64-microvm \                      # reduced binary
  -M microvm \                                           # reduced machine type
  -bios /usr/share/qemu/bios-microvm.bin \               # faster bios
  -kernel /boot/vmlinuz-5.4.0-39-generic \               # no internal bootloader
  -append "earlyprintk=ttyS0 console=ttyS0,115200,8n1" \
  -enable-kvm -cpu host -m 1G -smp 1 -nodefaults -no-user-config \
  -nographic -serial mon:stdio
```

My past as a performance engineer tells me that I neither have the right
hardware matrix nor the time to do a good performance evaluation of this.
But also this is meant to be a 5 minute read blog post and not a 140 pages
whitepaper. Therefore I beg you (and my old self) a pardon but I just couldn't
find a scientific enough, but fast way to satisfy myself in this short time.
I don't want to provide bad unreliable numbers to anyone - so no numbers today :-/

# Shared-FS

Since this usally aligns on container use-cases shared-directory is a common
pattern. To do so virtio-fs is available which has a server that can be spawned
like (do not do this on the hot path of your load, or you have wasted plenty of
time, have these ready beforehand):

```plaintext
/usr/lib/qemu/virtiofsd --socket-path=/tmp/myvhostqemu -o source=/tmp/testdir -o cache=always
```

And on the VM you can connect to it via:
```plaintext
... -chardev socket,id=char0,path=/tmp/myvhostqemu -device vhost-user-fs-device,queue-size=1024,chardev=char0,tag=myfs
```


# Drawbacks

This new class of feature-focused hypervisors fills a gap between system
containers and full hypervisors.
But you have to be sure that this is the spot your application design
needs/wants to be at.

The huge gains are mostly due to dropping features not needed for these use
cases. But if you end up then demanding pci-passthrough or migration support to
name a few, you'll end up needing a full featured hypervisor again.

# Next steps

So many things carved off in the Host/initialization means that the guest
content becomes even more important.
Even with a non optimized QEMU/KVM binary and configuration the guest boot and
initialization has consumed the majority of time.
By reducing the time the hypervisor needs to initialize this the focus to
further improve now is even more on the guest side.
You might consider running software directly from an initrd or from an
virtio-fs shared directory.
Also consider doing so without a full init system right into the workload that
matters to you (*matching non system-containers*).

The line between classic hypervisors like QEMU/KVM and containers already got
blurred by high quality system containers like
[LXD](https://linuxcontainers.org/lxd/introduction/).
These lines now get even more blurred, on one side with
container-use-case-optimized hypervisors (which this blog is about) and on the
other side containers starting to provide
[full virtualization](https://discuss.linuxcontainers.org/t/running-virtual-machines-with-lxd-4-0/7519)
as transparent alternative.
