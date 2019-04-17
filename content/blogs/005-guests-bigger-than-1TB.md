---
title: "KVM Guests bigger than 1TiB"
date: 2019-04-17T10:08:27+02:00
tags: ["libvirt", "qemu"]
draft: false
---

# KVM Guests bigger than 1TiB #
## a.k.a size matters ##

### Background addressing memory ###

More memory is almost always good, but to be able to use it it needs to be addressable as well.
CPU's are different and you can check yours how many bits are really supported.

```shell
$ cat /proc/cpuinfo | grep '^address sizes'
...
# an older server with a E5-2620
address sizes   : 46 bits physical, 48 bits virtual
# a laptop with an i7-8550U
address sizes   : 39 bits physical, 48 bits virtual
```

And ignoring all other constraints, to address 1TiB you'll need 40 bits,
so if you want to have more than 1TiB of guest memory it needs more than 40 bits to address it.


### Qemu, Libvirt and addressing bits exposed to the guest ###

Ideally you'd want your virtual CPU's in the guest to match the Hosts physical bits.
But then you clearly do not want to change the amount of physical bits when e.g. migrating between two hosts.

Therefore qemu by default chose to [expose 40 bits](https://git.qemu.org/?p=qemu.git;a=blob;f=target-i386/cpu.c;h=89ca3268d4e998eb4877b123391630652bb5d300;hb=11f6fee5766#l3020)
on x86 which also is what [TCG](https://wiki.qemu.org/Documentation/TCG) supports.

As we learned above that will not be enough to have guests >1TiB.
To fix that qemu can be told to either use a defined amount of physical bits using the `phys-bits` attribute of the CPU definition.

```shell
$ qemu-system-x86_64 ... -cpu ...,phys-bits=42
```

As mentioned above you'd actually best want host and guest phys bits to match and this is where [`host-phys-bits`](https://git.qemu.org/?p=qemu.git;a=blob;f=target-i386/cpu.c;h=89ca3268d4e998eb4877b123391630652bb5d300;hb=11f6fee5766#l3020)
comes into play automatically passing the amount of bits that your host CPU has.

```shell
$ qemu-system-x86_64 ... -cpu ...,host-phys-bits
```

OK, problem solved you might think - why not make host-phys-bits the default then?
Well, as mentioned before in virtualization you want predictability of how the guest is constructed.
You want no difference if you start the same guest the same way on two systems and neither do you want to change things when migrating.

Still you'd think then you'd just configure your libvirt, OpenStack or whatever else is managing your guests to set `host-phys-bits` or `phys-bits` as needed.
And there is the problem, as of today libvirt (and due to that all higher level stacks that manage guests through libvirt) can not express `[host-]phys-bits`.
This is a known [Ubuntu](https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1769053) and [upstream](https://bugzilla.redhat.com/show_bug.cgi?id=1578278) problem.

### Special machine types to the - interim - rescue ##

Long term everyone would prefer it to be implemented in libvirt and adopted
in upper layers of virtualization management, but what can you do asap if you want to
run rather large guests.

Ubuntu >=18.04 defines special x86 machine types with the `-hpb` suffix for
`Host-Phys-Bits`.
Ubuntu usually provides custom Machine types anyway to encapsulate any
downstream or portability cases.
And since CPU attributes can be modified in those it is easy to provide
*one more* machine type.

Qemu will tell you about all current known machine types and there you can
see it:

```shell
$ qemu-system-x86_64 -M ? | grep hpb
pc-i440fx-bionic-hpb Ubuntu 18.04 PC (i440FX + PIIX, +host-phys-bits=true, 1996)
pc-q35-bionic-hpb    Ubuntu 18.04 PC (Q35 + ICH9, +host-phys-bits=true, 2009)
```

And in libvirt that would look like
```xml
...
  <os>
    <type arch='x86_64' machine='pc-i440fx-bionic-hpb'>hvm</type>
...
```

This is a neat trick as defining the machine type is a rather old feature and therefore already supported in most consumers of libvirt.
For example in OpenStack that can be set via:
* Global via [nova config](https://docs.openstack.org/nova/pike/configuration/config.html)
* Per image via [metadata](https://docs.openstack.org/image-guide/image-metadata.html)

### Outlook and next steps ###

As stated in the [NEWS file](https://git.launchpad.net/ubuntu/+source/qemu/tree/debian/qemu-system-x86.NEWS?h=ubuntu/disco-devel#n3) the intention is to stop adding `-hpb` types
once this can be expressed in libvirt and is adopted enough in higher level tools to be no more needed.

But that comes down to actually implement this [upstream](https://bugzilla.redhat.com/show_bug.cgi?id=1578278) first.
Until then it is a simple helper to ease the life of Ubuntu users using very large guests.
