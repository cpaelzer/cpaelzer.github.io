---
title: "Live Migration with changing ROM sizes"
date: 2018-02-20T13:51:46+01:00
tags: ["qemu", "migration", "option-roms"]
draft: false
---

# Live Migration with changing Rom sizes #
## a.k.a size matters ##

### Background on QEMU ROM ###

Virtual PCI devices have Option ROM support just like physical PIC device. Option ROMs for PCI devices have been used to enable additional functionality, such as PXE Boot.
The size of these is implied by the PCI spec to be a power of two.
In QEMU the size allocated for such a rom is defined by the size of the file backing up the rom.

As an example for virtio network card the `virtio-net-pci` driver defined `efi-virtio.rom` as default rom file in `hw/virtio/virtio-pci.c:2361`.
And for example on a Xenial system this file is provided by the package `ipxe-qemu` at:

```-rw-r--r-- 1 root root 237K Nov 22 18:15 /usr/lib/ipxe/qemu/efi-virtio.rom```


```lang-none
             +------------------------------+
             | Virtual System               |
             |                              |
             |                              |
             |                              |
             |  +-----------+               |
             |  | Rom 2**x  |               |
             |  | 256k      |               |
             |  +---^-------+               |
             +------------------------------+
                    | file load defines size
                    |
             +------+
             | file |
             | 237k |
             +------+
```

This space is visible as PCI mapping from the guest, so obviously it is not allowed to change at runtime.

### Implications on Migration ###

All of the above works fine as long as sizes of the ROM do not change.
But remember that size is defined by loading from a file, so on a migration the target loads its local file.
Things become complex if that file has a different size.  For example, let’s look at the bionic version of the rom:

```-rw-r--r-- 1 root root 294K Feb  5 14:09 /usr/lib/ipxe/qemu/efi-virtio.rom```

When migrating between two virtual hosts,  the target virtual host will allocate the smallest power-of-two size that fits the target hosts version of the ROM file. From above, the file size is 294k which is larger than 256 (2 << 7) and the next power of two is 2 << 8, which results in a ROM space of 512k.
This will trigger an [issue](https://bugs.launchpad.net/ubuntu/+source/ipxe/+bug/1713490) on the migration like this:
`qemu-system-x86_64: Length mismatch: 0000:00:03.0/virtio-net-pci.rom: 0x40000 in != 0x80000: Invalid argument`

```lang-none
 +-------------------------+                 +-------------------------+
 | Virtual System          |                 | Virtual System          |
 |                         |                 |                         |
 |                         |                 |                         |
 |                         |                 |                         |
 |  +-----------+          |   Migration     |  +---------------------+|
 |  | Rom 2**x  |          | +----------->   |  | Rom 2**x            ||
 |  | 256k      |          |   breaks        |  | 512k                ||
 |  +---^-------+          |   as rom        |  +--^------------------+|
 +-------------------------+   size is       +-------------------------+
    file load defines size     guest visible    file load defines size
        |                      and not            |
 +------+                      allowed          +-----+------+
 | file |                      to change        | file       |
 | 237k |                                       | 294k       |
 +------+                                       +------------+
```

And you are safe on the content - as the actual content of the rom will be migrated over.
So even with a different content file this won't break on migration.

### Who shall fix it ###

This is a tricky question. To some extent the ROMs come from different projects than QEMU.
So QEMU can't perfectly map versions to ROMs as the ROM files may change independently of QEMU..

You might say, "But hey they bundle ROMs on the release tarballs".
That is correct, but won't help due to differing distribution packaging [policies](https://www.debian.org/social_contract#guidelines).
Some distribution have restrictions on bundling pre-build ROMs
Furthermore different distributions might want to add or enable/disable different features in those ROMs.
For example Ubuntu had https enabled for quite a while which makes our ROMs have a different size compared to Debian.

With all that in mind we can agree that upstream can't handle it as it isn't fully under their control.
The actual projects of the various ROMs have even less control where and how they are bundled.
QEMU upstream can provide some generic mechanisms to handle those cases nicely though.

```lang-none
                   +---------------+
                   | QEMU bundling |
                   |               |
                   | Versions      |
                   +-------+-------+
+--------------+           |           +------------------------+
| Rom projects |           |           | Distributions          |
|              +-----------------------+ might                  |
| code         |           |           | bundle other Versions  |
+--------------+           |           +-----------+------------+
                           |                       |
                           |                       |
+--------------+           |           ------------+------------+
| Distribution |           |           | Distributions          |
| toolchain    +-----------------------+ enable/disable features|
| on build     |           |           | modifying code         |
+--------------+           |           +------------------------+
                   +-------v-------+
                   | Size of rom   |
                   |               |
                   | in a release  |
                   +-------+-------+
                           |
                +----------v----------+
                | Distribution decides|
                | which QEMU version  |
                | will it be aligned  |
                | with?               |
                +---------------------+
```

Eventually this is something that falls back on the Distributions as part of the integration of those projects and the compatibility for upgrades, migrations and similar.

### Changing sizes but retaining migratability ###

When I first understood all the implications the runway was too short (close to [Ubuntu 17.10](https://wiki.ubuntu.com/ArtfulAardvark/ReleaseSchedule) being released).
I reverted my ipxe changes and took a look how it is handled so far.
And to my surprise it seems that mostly people just hope it doesn't happen - there isn't a very clear solution to it.

First of all I added a [build time check](https://git.launchpad.net/~paelzer/ubuntu/+source/ipxe/commit/?id=6eb7e240412930d64878fcedd23969c929ae51b1) that will prevent any fix/change to accidentally change the sizes.
But at least for the next release I needed a solution to not force us to keep an outdated ipxe.

I discussed with several people and the obvious options were all unacceptable:

* disable some features to again fit (you never know when it breaks, also people generally want Features :-))
* strip/shrink/... all that is done as part of the build process anyway
* declare live migrations not supported (obvious no-go)

I reached out further and after a while upstream discussion revealed that a few had solved similar cases with [mapping older machine types to files of a different size](http://lists.nongnu.org/archive/html/qemu-devel/2017-11/msg00484.html).
And after no better solution was found I [proposed](https://paste.ubuntu.com/25925120/) such a change [in sync with Debian](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=881263).
This wasn't picked up yet, but the discussions triggered around it were good.

### Proposed Solution for QEMU 2.11 ###

Eventually for Ubuntu 18.04 I needed a solution and implemented the file mapping on the machine types.
This solution consists of the following pieces:

* a [compat package](https://launchpad.net/ubuntu/+source/ipxe-qemu-256k-compat) having ROMs of the former size under different file names
* a [patch to QEMU](https://anonscm.debian.org/cgit/pkg-qemu/qemu.git/tree/debian/patches/ubuntu/pre-bionic-256k-ipxe-efi-roms.patch?id=refs/heads/ubuntu-bionic-2.11) that uses HW_COMPAT to map to the old filenames for older types

That way on a migration QEMU will allocate based on the compat file. The size will match, and the content will get transferred. Migration is working just fine now.

```lang-none
 +-------------------------+                 +-------------------------+
 | Virtual System          |                 | Virtual System          |
 |                         |                 |                         |
 |                         |                 |                         |
 |                         |                 |                         |
 |  +-----------+          |   Migration     |  +---------------------+|
 |  | Rom 2**x  |          | +----------->   |  | Rom 2**x            ||
 |  | 256k      |          |   works         |  | 256k                ||
 |  +---^-------+          |                 |  +--^------------------+|
 +-------------------------+   but at the    +-------------------------+
    file load defines size     same time        file load defines size
        |                      new guests         |
 +------+                      will use the     +-----+------+
 | file |                      new ROMs         | compat-file|<-incoming machine
 | 237k |                                       | 237k       |  type selects
 +------+                                       +------------+
```

Even with all the effort to have it working, please never forget that in general it is recommended to [upgrade the machine type](https://wiki.ubuntu.com/QemuKVMMigration#Upgrade_machine_type) when you can.

By the way - to help you on this there is an experimental [snap](https://snapcraft.io/) called [virt-machine-type](https://launchpad.net/virt-machine-type/+announcement/14486).

### Further thoughts ###

Even on the current solution I think I should move up to the non arch dependent compat.h at least.
I have had some ideas, but no time yet to consider or RFC code any of them.  Here they are for now:

* Maybe we should pad ROMs to a way higher size to never hit the issue again
  We could do so, but that action on itself will hit this issue so I want a clear solution on handling those differences before doing so.
* I wondered if distributions should carry all released versions of the ROMs all the time.
  That way spawning a guest of a trusty machine type would get exactly the rom on any releases.
  But that is packaging, dependency and bug fixing nightmare - so I'm not voting for this.
* While it is a downstream issue, a more upstream defined way of "how to handle this" might be nice.
  It might have a huge flaw I don't see yet, but why not allocate based on the incoming migration rom size instead of file.
  For the latter I assume it might have to do with the spawning happen before the incoming migration actually takes place, so one might need a two stage migration (1 pass info, 2 (re)-spawn and do it).
  On the other hand the machine type is known while spawning as well, maybe we could at least just hard set the sizes instead of the indirection via files.
  Especially for the rom size issues this seems nice, since:
 * the content is migrated anyway
 * the correct size is passed on migration

Given enough time I'd think something like that will be a better solution.
But until then mapping via the machine types seems to be the best we can do to provide clean migrations to users.
