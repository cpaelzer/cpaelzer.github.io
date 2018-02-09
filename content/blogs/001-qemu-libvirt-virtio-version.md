---
title: "query and control virtio version"
date: 2018-02-07T12:51:26+01:00
draft: true
---

# virtio: modern/legacy = ??? #
### a.k.a puzzlement about inquiring or controlling virtio revisions via libvirt ###

When looking around how to check for the current setting of modern/legacy virtio,
or how to control it via libvirt I only found dead ends at first.

 * If you search how to get some info about it you will find [query-virtio/info virtio][1], but after 5 versions being discussed nothing is merged still.
 * If instead you search how to better control the version you'll find [virtio revision][2], but also 5 versions discussed until declared postponed.

You might find the baked-in defaults changing in qemu 2.7 in include/hw/compat.h by this [commit][3].
But this has so many special cases and triggers that might change it, that at least I wasn#t sure what effectively is used.

So what could you do to:

 * check how a device is configured in regard to modern/legacy at the moment?
 * control the modern/legacy settings via libvirt for those unable to run qemu via cmdline?

It is not nice and clean due to above mentioned changes not having landed, but at least there is some way to do it.
To query you can at least find the info in the qtree.
using [virsh qemu-monitor-command][4] as an easy way to access the guests monitor you can for example run.

`$ virsh qemu-monitor-command --hmp <guestname> 'info qtree'`

That gave me the following per device virtio-blk-pci that I was looking for:

```
          dev: virtio-blk-pci, id "virtio-disk0"
            [...]
            disable-legacy = "on"
            disable-modern = false
```

Obviously this lets you check way more attributes and devices, so take a look even if you are looking for something else.

In my case I now knew that disable-legacy was "on".
But how could I change that when - as in my case - I can't modify the qemu cmdline directly?

There is [qemu:commandline][5] to control qemu commandline arguments from libvirt, but in my case I needed to inject a driver property.
And that the disks part of the commandline is rendered independent to qemu:commandline.
Remember that due to the lack of [virtio revision control][2] you can't just tell the libvirt to render it the way you want it per device.
Also in addition I also had hotplug devices which would use a completely different path than the commandline anyway.

The solution is to combine [qemu:commandline][5] with -global of the [qemu standard options][6].
Combining all that a change to my guest looked like:

```
    > <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
    [...]
    >   <qemu:commandline>
    >     <qemu:arg value='-global'/>
    >     <qemu:arg value='virtio-blk-pci.disable-legacy=off'/>
    >   </qemu:commandline>
```

That made the qemu commandline have: `-global virtio-blk-pci.disable-legacy=off`
And that in turn made my devices supporting modern and legcay as I wanted it for a test that I was doing.

          dev: virtio-blk-pci, id "virtio-disk0"
            [...]
            disable-legacy = "off"
            disable-modern = false

I hope that might help somebody else out there as well.

Dusma bleiwe ...

[1]: https://lists.gnu.org/archive/html/qemu-devel/2017-10/msg00393.html
[2]: https://www.redhat.com/archives/libvir-list/2016-September/msg00227.html
[3]: https://git.qemu.org/?p=qemu.git;a=commit;h=9a4c0e220d8a4f82b5665d0ee95ef94d8e1509d5
[4]: http://blog.vmsplice.net/2011/03/how-to-access-qemu-monitor-through.html
[5]: https://libvirt.org/drvqemu.html#qemucommand
[6]: https://qemu.weilnetz.de/doc/qemu-doc.html#Standard-options
