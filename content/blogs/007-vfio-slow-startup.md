---
title: "Slow VFIO guest startup"
date: 2024-04-28T12:08:27+02:00
tags: ["vfio", "iommu", "qemu", "passthrough"]
draft: false
---

# Slow guest startup caused by using VFIO #
## a.k.a being lost without a map(ing) ##

Disclaimer: for now this is about the debugging where the time sink is, it mentions ways to mitigate some effects but not a fix.
If later a fix comes up I'll update.

### 2024 Update ###

To overcome this Qemu added and evolved pre-allocation quite a bit as you can
often see in the "Memory backends" of the [changelogs](https://wiki.qemu.org/ChangeLog/5.0) of 5.0 and later.
This got further improved, stabilized and in [Libvirt 8.2](https://libvirt.org/news.html#v8-2-0-2022-04-01)
got exposure to higher management levels. There as well several fixups landed in
later releases. Look for the details on [memory backing configuration](https://libvirt.org/formatdomain.html#memory-backing) to try it.
IMHO all that today (April 2024) improves the situation a lot and one can
indeed use those features to see it starting much faster in a recent
Ubuntu 24.04 Noble than what I found back in 2019 when analyzing this the
first time. Proper numa config was always important for guests of this size
and also is important for this initialization.

### Symptom: classic linear scaling at work ###

Starting a KVM guest usually is in the range of a few seconds. Obviously the guest then needs time to boot up, but the preparation phase is rather short.
This isn't true if you use VFIO passthrough. If you do so the memory needs to be allocated, pinned and mapped to the iommu before the guest can really start.
These actions are known to be quite slow/expensive, but there are a few things that make this worse.

The first time I used passthrough guests had the size of probably a half to a few GiB.
But these days guests can grow up to terabytes as in [bug 1838575](https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1838575) when it was reported to me.
I wanted to make sure I really know where the bottlenecks are.

It turns out it all was just [linear scaling of O(n)](https://en.wikipedia.org/wiki/Big_O_notation).
The kernel task that allocates and maps the memory is single threaded, so no matter how big your nice CPU count grows 1TiB is ~1000 time slower than the 1GiB guest of the old days.

Initially it wasn't clear if the guest preparation, the bootloader or the boot inside the guest consumed the time.
Therefore I tracked time in three sections prep (until the console connects) / bootloader (until you see the kernel to initialize) / boot-up (until ready for login).

```plaintext
| Guest / Time (sec)         | Prep | bootloader | boot-up |
| 512MB 1CPU no-VFIO         |    4 |         12 |     17  |
| 1.2TB 1CPU no-VFIO         |    6 |         21 |     16  |
| 1.2TB 1CPU    VFIO         |  253 |         20 |     18  |
```

So the extra time consumed clearly was in the preparation stage when qemu/kvm sets up things
Running [perf](https://perf.wiki.kernel.org/index.php/Tutorial) along that showed work was mostly in the memory management, in the setup I had around transparent huge page handling.

```
  73.91% [kernel] [k] clear_page_erms
   9.53% [kernel] [k] clear_huge_page
   1.65% [kernel] [k] follow_trans_huge_pmd
```

### How to make it worse ###

In the debugging I found that a mid sized 128GiB guest might actually take from a few seconds or up to 10 minutes to start up.
I later realized this was due to fragmentation of the memory which made it need more allocation calls, more locking to get these resources and finally more calls to map things in the [IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit).
We will come later to the details of this, but on top of the linear scaling for raw guest size this could add up to a factor of ```x15``` in he experiments that I have run.

That way the 1.x TiB guest can take more than half an hour to start which is over patience limit of even the most relaxed admin and it will have shot it dead before that happens.

### Immediate workarounds that come to mind ###

If you did not come here for a nice debugging story, but only want this hilarious long guest start time to go down then I'd strongly recommend to use huge pages.
Especially with guest sizes >128GiB in my humble opinion 1G [huge pages](https://libvirt.org/formatdomain.html#elementsMemoryBacking) pay off huge.
This is due to multiple reasons in this particular case:
- more memory per page = less mappings
- huge pages are only consumed explicitly, which usually means never unless you do so. Due to that:
  - the system more easily finds free pages
  - almost no fragmentation

To be clear, the initialization speed using 1G huge pages still is linear to the guest size.
But you can save an order of magnitude and to a huge extend avoid the further delay amplification by the fragmentation.

Extending the table above
```plaintext
| Guest / Time (sec)         | Prep | bootloader | boot-up |
| 512MB 1CPU no-VFIO         |    4 |         12 |     17  |
| 1.2TB 1CPU no-VFIO         |    6 |         21 |     16  |
| 1.2TB 1CPU    VFIO         |  253 |         20 |     18  |
| 1.2TB 1CPU    VFIO noTHP   |  476 |         31 |     20  |
| 1.2TB 1CPU    VFIO 1G-HP   |   59 |         21 |     18  |
```

So hugepages help a lot in this case, but all of you who already set up huge pages know it.
Pre-allocating and distributing them among guest is admin/management effort that has its costs in other places.

### Check where time is lost - part I: Userspace ###

The obvious next step is to wonder where so much time is lost to optimize it.
To be without pre-judgment to the kernel lets start in Userspace.

Old, simple but still helpful is `strace` and it clearly pointed to the VFIO ioctl.
Here a case with a ~55 second hang on this call for a 128GiB guest:

```
[...]
0.000041 openat(AT_FDCWD, "/dev/vfio/vfio", O_RDWR|O_CLOEXEC) = 32 <0.000019>
[...]
0.000058 ioctl(32, VFIO_DEVICE_PCI_HOT_RESET or VFIO_IOMMU_MAP_DMA, 0x7ffd774f7f20) = 0 <0.000037>
0.000060 ioctl(32, VFIO_DEVICE_PCI_HOT_RESET or VFIO_IOMMU_MAP_DMA, 0x7ffd774f7f20) = 0 <0.334458>
0.334593 ioctl(32, VFIO_DEVICE_PCI_HOT_RESET or VFIO_IOMMU_MAP_DMA, 0x7ffd774f7f20) = 0 <0.000055>
0.000088 ioctl(32, VFIO_DEVICE_PCI_HOT_RESET or VFIO_IOMMU_MAP_DMA <unfinished ...>
[...]
45.903415 <... ioctl resumed> , 0x7ffd774f7f20) = 0 <55.558237>
```

To be thorough I checked if this was special to my machine, but it affected the new/huge boxes of two major x86 chip manufacturers.
And to be fully complete I checked non x86 on a ppc64 box which was still very similar.

```
0.000037 ioctl(17, VFIO_SET_IOMMU, 0x7) = 0 <0.000039>
0.000070 ioctl(17, VFIO_IOMMU_SPAPR_REGISTER_MEMORY <unfinished ...>
[...]
276.894553 <... ioctl resumed> , 0x7fffe3fd6b70) = 0 <286.974436>
```

### Check where time is lost - part II: which ioctl exactly ###


As you see in  the former section there were multiple such ioctl calls. I wanted to see what made the slow one special.
Fortunately `gdb` can easily grab ioctls with `catch syscall 16`.

After a bit of checking for the arguments I came up with this simple `qemu` and `gdb` setup that quickly got me directly *in front* of the slow ioctl.
First I freed the device and started gdb:

```
$ virsh nodedev-detach pci_0000_21_00_1 --driver vfio
$ gdb /usr/bin/qemu-system-x86_64
```

And then in gdb we just wait for the right size to come by (my 128GiB example).
From there we catch the next ioctl which always is the right one:

```
(gdb) b vfio_dma_map
(gdb) command 1
Type commands for breakpoint(s) 1, one per line.
End with a line saying just "end".
>silent
>if size != 134217728000
 >cont
 >end
>end
(gdb) run -m 131072 -smp 1 -no-user-config -device vfio-pci,host=21:00.1,id=hostdev0,bus=pci.0,addr=0x7 -enable-kvm
(gdb) catch syscall 16
(gdb) c
```

A backtrace from here looks like this

```
(gdb) bt
#0  in ioctl () at ../sysdeps/unix/syscall-template.S:78
#1  in vfio_dma_map (container=0x555557608430, iova=4294967296, size=134217728000,
                     vaddr=0x7fe01fe00000, readonly=false)
#2  in vfio_listener_region_add (listener=0x555557608440, section=0x7fffffffcad0)
#3  in listener_add_address_space (listener=0x555557608440, as=0x5555565ca5a0)
#4  in memory_listener_register (listener=0x555557608440, as=0x5555565ca5a0)
#5  in vfio_connect_container (group=0x5555576083b0, as=0x5555565ca5a0,
                               errp=0x7fffffffdda8)
#6  in vfio_get_group (groupid=45, as=0x5555565ca5a0, errp=0x7fffffffdda8)
#7  in vfio_realize (pdev=0x555557600570, errp=0x7fffffffdda8)
#8  in pci_qdev_realize (qdev=0x555557600570, errp=0x7fffffffde20)
#9  in device_set_realized (obj=0x555557600570, value=true, errp=0x7fffffffdff0)
[...]
#17 in main (argc=14, argv=0x7fffffffe3d8, envp=0x7fffffffe450) at vl.c:4387
```

With that we were set up checking what happens in the kernel right at this call.

### Check where time is lost - part III: Kernel tracing ###

From perf we knew it must be in some loops for memory management, but I wanted to know more which higher level calls the time was lost.
Eventually I realized the best for my checks would be:

```
$ sudo trace-cmd record -p function_graph -l vfio_pin_pages_remote -l vfio_iommu_map
```

I ran that twice, once with a 12 second guest start right after boot (almost no fragmentation).
And then restarting the same guest which took ~175 seconds.

I generally got long repeating sequences like these:

```
funcgraph_entry:      ! 428.925 us |        vfio_pin_pages_remote();
funcgraph_entry:        0.771 us   |        vfio_iommu_map();
funcgraph_entry:      ! 426.510 us |        vfio_pin_pages_remote();
[...]
```

Comparing the slow and fast cases I found that the slow case had faster but more of these sequences.
The assumption was that fragmentation might cause this and due to more contention in the memory management to find free consecutive areas and more calls to pin and map these it would take more time.

### Check where time is lost - part IV: systemtap ###

I wanted to check the count and size of the `vfio_pin_pages_remote` and `vfio_iommu_map` activities.
With a systemtap script like this we get all we need:

```
probe module("vfio_iommu_type1").function("vfio_iommu_type1_ioctl") {
    printf("New vfio_iommu_type1_ioctl\n");
    start_stopwatch("vfioioctl");
}
probe module("vfio_iommu_type1").function("vfio_iommu_type1_ioctl").return {
    timer=read_stopwatch_ns("vfioioctl")
    printf("Completed vfio_iommu_type1_ioctl: %d\n", timer);
    stop_stopwatch("vfioioctl");
}
probe module("vfio_iommu_type1").function("vfio_pin_pages_remote") {
    timer=read_stopwatch_ns("vfioioctl")
    printf("%ld: %s\n", timer, $$parms);
}
```

The stopwatch is like a monotonic "add cpu time consumption" clock for vfio_iommu_type1_ioctl.
The overall amount of calls we can directly get from the entries and the most important bit was the npage argument that lets us know how big a chunk of consecutive pages it found.
Thereby the trace contained all the information we wanted.

```
1154838: dma=0xffff8a65b28e7300 vaddr=0x7fe0a8400000 npage=0x1f3fa00 pfn_base=0xffffac19160a3d48 limit=0x4000
1764663: dma=0xffff8a65b28e7300 vaddr=0x7fe0a8800000 npage=0x1f3f600 pfn_base=0xffffac19160a3d48 limit=0x4000
2375581: dma=0xffff8a65b28e7300 vaddr=0x7fe0a8c00000 npage=0x1f3f200 pfn_base=0xffffac19160a3d48 limit=0x4000
2980276: dma=0xffff8a65b28e7300 vaddr=0x7fe0a9000000 npage=0x1f3ee00 pfn_base=0xffffac19160a3d48 limit=0x4000
3590913: dma=0xffff8a65b28e7300 vaddr=0x7fe0a9400000 npage=0x1f3ea00 pfn_base=0xffffac19160a3d48 limit=0x4000
4200148: dma=0xffff8a65b28e7300 vaddr=0x7fe0a9800000 npage=0x1f3e600 pfn_base=0xffffac19160a3d48 limit=0x4000
```

In general both logs begin to pin and map with some rather large allocations.
But then they drop off to smaller sizes. That is the [loop in vfio_pin_map_dma](https://github.com/torvalds/linux/blob/master/drivers/vfio/vfio_iommu_type1.c#L1003) that does that.

With that I was able to compare the fast and slow cases (due to fragmentation).
And it confirmed the theory. Not only - despite all trace overhead - did it still take almost twice as long.
Further it confirmed that the slow case had more and smaller calls.

```plaintext
case / log(10) of size | 3   | 4    | 5     | 6      | 7     | 8     |
Fast                   | 108 | 1293 | 12133 | 113330 | 27794 | 1119  |
Slow                   | 194 | 1738 | 17375 | 143411 |    55 |    3  |
```

This shows for the traced duration that the fast case has a few much larger
calls which the slow case had to make up with much more smaller calls.

P.S. I had no lost events reported, but I assume I have aborted the slow case too early expecting another few thousands of <=6 allocations to come in.
But since I only compared relative behavior and changes over time that should still be valid data IMHO.


### Outlook - how to break that linear scaling ###

As mentioned there are several flat improvements possible by the use of huge pages.
But as the time goes on very large guests will become more and more normal.
And to speed up those growing sizes at some point we need to scale better than O(n).

The obvious thought is to do that in multiple threads, but as this is already lock heavy the benefits might be negligible.
Also iommu's are known to be full of quirks like needing the mappings in order or even in a single call - furthermore each of them tends to be slightly different so a solution for one might fail on the other.
We'd need to find a way to divide and conquer this keeping these constraints in mind - at least it is only really an issue in very huge sizes so we don't need to split into small chunks (where locks are more of a problem).

After the analysis above I knew in which area to look and - as usual as - soon as you know that you find others with the same issues.
It seems kernel development here is already ongoing, but might be stalled as the last update is from late 2018.

- [Series implementing some related improvements via ktask](https://lwn.net/Articles/770826/)
- [LWN Article about this series](https://lwn.net/Articles/771169/)
- [Similar issue reported on memory hot add with vfio](https://www.redhat.com/archives/vfio-users/2018-April/msg00020.html)
- [Plumbers presentation on ktask](https://linuxplumbersconf.org/event/2/contributions/159/attachments/19/146/ktask_lpc_2018_daniel_jordan.pdf)
- [particular change interesting for my case](http://lkml.iu.edu/hypermail/linux/kernel/1811.0/03361.html)
- [Blog post about ktask](https://blogs.oracle.com/linux/making-kernel-tasks-faster-with-ktask,-an-update)
