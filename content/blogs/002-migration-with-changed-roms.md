---
title: "Live Migration with changing Roms"
date: 2018-02-15T13:51:46+01:00
tags: ["qemu", "migration"]
draft: true
---

# Live Migration with changing Roms #
## a.k.a size matters ##

### Background on Qemu roms ###

Virtual Hardware still has rom to represent it.
The size of these is implied by the pci spec to be a power of two.
In qemu the size allocated for such a rom is defined by the size of the file backing up the rom.

So as an example for virtio network card the `virtio-net-pci` driver defined `efi-virtio.rom` as default rom file in `hw/virtio/virtio-pci.c:2361`.
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

This space is visible as pci mapping from the guest, so obviously it is not allowed to change at runtime.

### Implications on Migration ###

All of the above works fine as long as you have just one system and sizes of the rom's do not change.
But remember that size is defined by loading from a file, so on a migration the target loads it's local file.
Things become complex if that file has a different size, like my bionic file of the same as above has:

```-rw-r--r-- 1 root root 294K Feb  5 14:09 /usr/lib/ipxe/qemu/efi-virtio.rom```

Now the target of a migration will allocate a power of 2 area that fits 294k which means 512k.
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

This is a tricky question. To some extend the roms come from different projects than qemu.
So qemu can't perfectly map versions to roms.

"But hey they bundle roms on the release tarballs" you might say.
That is correct, but won't help.
On one hand there are [policies](https://www.debian.org/social_contract#guidelines) which do not allow to pick up those pre-built roms for various reasons.
Furthermore different distributions might want to add or enable/disable different features in those roms.
For example Ubuntu had https enabled for quite a while which makes our roms having a different size to Debian.

With all that in mind we can agree that upstream can't handle it as it isn't fully under their control.
The actual projects of the various roms have even less control where/how they are bundled.
Qemu upstream can provide some generic mechanisms to handle those cases nicely thou.

```lang-none
                   +---------------+
                   | qemu bundling |
                   |               |
                   | Versions      |
                   +-------+-------+
+--------------+           |           +-----------------------+
| Rom projects |           |           | Distributions         |
|              +-----------------------+ might                 |
| code         |           |           | bundle other Versions |
+--------------+           |           +-----------+-----------+
                           |                       |
                           |                       |
+--------------+           |           ------------+-----------+
| Distribution |           |           | Distributions         |
| toolchain    +-----------------------+ enable/disable feaures|
| on build     |           |           | modifying code        |
+--------------+           |           +-----------------------+
                   +-------v-------+
                   | Size of rom   |
                   |               |
                   | in a release  |
                   +-------+-------+
                           |
                +----------v----------+
                | Distribution decides|
                | which qemu version  |
                | will it be aligned  |
                | with?               |
                +---------------------+
```

So eventually this is something that falls back on the Distributions as part of the integration of those projects and the copatibility for upgrades, migrations and similar.

### Changing sizes but retaining migratability ###

When I first understood all the implications the runway was too short (close to [Ubuntu 17.10](https://wiki.ubuntu.com/ArtfulAardvark/ReleaseSchedule) being released).
So I reverted my ipxe changes and took a look how it is handled so far.
And to my surprise it seems that mostly people just hope it doesn't happen - there isn't a very clear solution to it.

So first of all I added a [build time check](https://git.launchpad.net/~paelzer/ubuntu/+source/ipxe/commit/?id=6eb7e240412930d64878fcedd23969c929ae51b1) that will prevent any fix/change to accidentially change the sizes.
But at least for the next release I needed a solution to not be forced to keep and outdated ipxe.

I discussed with several people and the obvious options were:

* disable some features to again fit (you never know when it breaks, also I want Features :-))
* strip/shrink/... all that is done as part of the build process anyway
* declare live migrations not supported (obvious no-go)

I reached out further and after a while upstream discussion revealed that a few had solved it with [mapping older machine types to files of a different size](http://lists.nongnu.org/archive/html/qemu-devel/2017-11/msg00484.html).
And after no better solution was found I [proposed](https://paste.ubuntu.com/25925120/) such a change [in sync with Debian](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=881263).
This wasn't picked up yet, but the discussions triggered around it were good.

Eventually for Ubuntu 18.04 I needed a solution and implemented the file mapping on the machine types.
This solution consists of TODO pieces:
* a [compat package](https://launchpad.net/ubuntu/+source/ipxe-qemu-256k-compat) having roms of the former size under different file names
* a patch to qemu that uses HW_COMPAT to map to the old filenames for older types

That way on a migration it will allocate based on the compat file - the size matches - and the content gets transferred.
Migration working just fine now.

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
 | file |                      new roms         | compat-file|<-incoming machine
 | 237k |                                       | 237k       |  type selects
 +------+                                       +------------+
```

Even with all the effort to have it working, please never forget that in general it is recommended to [upgrade the machine type](https://wiki.ubuntu.com/QemuKVMMigration#Upgrade_machine_type) when you can.

### Further thoughts ###

Even on the current solution I think I should move up to the non arch dependent compat.h at least.
But thinking further I have had some ideas, but no time yet to consider or RFC code any of them.

* Maybe we should pad roms to a way higher size to never hit the issue again
  We could do so, but that action on itself will hit this issue so I want a clear solution on handling those differences before doing so.
* I wondered if distributions should in general carry all released versions of the roms all the time.
  That way spawning a guest of a trusty machine type would get exactly the rom on any releases.
  But that is packaging, dependency and bug fixing nightmare - so I'm not votring for this.
* While it is a downstream issue, a more upstream defined way "how to handle" might be nice.
  It might have a huge flaw I don't see yet, but why not allocating based on the incoming migration rom size instead of file.
  For the latter I assume it might have to do with the spawning happen before the incoming migration actually takes place, so one might need a two stage migration (1 pass info, 2 (re)-spawn and do it).
  On the other hand the machine type is known while spawning as well, maybe we could at least just hard set the sizes instead of the indirection via files.
  Especially for the rom size issues this seems nice, since:
 * the content is migrated anyway
 * the correct size is passed on migration

Given enough time I'd think something like that will be a better solution.
But until then mapping via the machine types seems to be the best we can do to provide clean migrations to users.
