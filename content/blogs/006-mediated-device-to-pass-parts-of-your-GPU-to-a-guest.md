---
title: "Mediated Device GPU passthrough"
date: 2019-04-17T12:08:27+02:00
tags: ["libvirt", "qemu", "passthrough"]
draft: false
---

# Mediated Device based GPU passthrough #
## a.k.a passing part of a whole ##

### The many faces and names of vGPU ###

Having GPU resources in the guest has various use cases. From a windows gaming VM to GPGPU assisted AI learning.
Due to that there are many things people commonly call a "virtual Graphics Processing Unit" => `vGPU`.
Unfortunately it doesn't help at all that many guides, forum posts and blogs call all of them vGPU.
Also PCI-passthrough has many facets e.g. using VFIO and virtual-functions or not.

Up until recently we have had three types to make GPU somehow power available in a guest.

[PCI-passthrough](https://www.ibm.com/developerworks/linux/library/l-pci-passthrough/) => Usually used for GPGPU (but could be almost any PCI device)

* + Uses *normal* guest driver
* + Needs not really support by the Device
* - 1:1 Device-Guest mapping
* - Needs IOMMU / VT-d
* - Not migratable
* - IOMMU group restrictions

[VF-Passthrough](https://wiki.libvirt.org/page/Networking#PCI_Passthrough_of_host_network_devices) => Usually used for Network

* + Uses *normal* driver
* + 1:N Device-Guest mapping
* - Needs FW&Driver to create VFs
* - Needs IOMMU / VT-d
* - Not migratable
* - IOMMU groups restrictions

[VirtGL](https://virgil3d.github.io/) => some 3D in guest without passthrough

* + 1:N Device-Guest mapping
* + Needs no IOMMU / VT-d
* - Guest needs VirtGL driver
* - Not migratable
* - only for rendering
* - not as fast as device passthrough

Now there is another contender in the vGPU arena - [mediated devices](https://www.kernel.org/doc/Documentation/vfio-mediated-device.txt)

* + 1:N Device-Guest mapping
* + Needs no IOMMU / VT-d
* + Uses *normal* driver
* / Migratable (future)
* - Needs Driver to create MDEVs
* - Needs IOMMU / VT-d

If you compare the four of them from 10.000 feet they differentiate at the level where they are "split"

```plaintext
|                 | HW  | Firmware  | Kernel/Driver | Userspace    | Guest |
| PCI-passthrough | 1:1 | --------- | ------------- | ------------ | --x   |
| VF-Passthrough  |     | split VFs | ------------- | ------------ | --x   |
| Mediated Device |     |           |  split MDEVs  | ------------ | --x   |
| VirtGL          |     |           |               | VirtGL split | --x   |
```

### Choose your Ubuntu release ###

To a different extend mediated devices were usable before (since 18.04 Bionic Beaver) already.
But in Ubuntu 19.04 ([Disco Dingo](https://wiki.ubuntu.com/DiscoDingo/ReleaseNotes#qemu)) many things finally come together:

 * automatic apparmor rules for GL devices
 * automatic apparmor rules for Mediated Devices
 * Kernel bits for i915 (>=4.17)
 * Kernel bits for Nvidia (>=4.15)
 * Enablement of GL support in qemu
 * Many minor fixes in kernel, qemu and libvirt for this use case in 19.04

Not needed for mediated devices, but also often requested VirtGL finally is enabled in the virtualization stack of Ubuntu 19.04

For those that want to stick to the latest LTS release even for such experiments there is a way out using:

* [HWE Kernel](https://wiki.ubuntu.com/Kernel/LTSEnablementStack) with kernel 5.0 from Disco
* [Ubuntu Cloud Archive](https://wiki.ubuntu.com/OpenStack/CloudArchive) with qemu 3.1 and libvirt 5.0 from Disco

### Prepare the Mediated Device ###

This example uses the Intel i915 GPU that is less powerful but much more widely available than the high profile [Nvidia cards](https://docs.nvidia.com/grid/latest/grid-vgpu-release-notes-red-hat-el-kvm/index.html) supporting mediated devices.
First of all you need:

* load VFIO modules on boot
* enable IOMMU
* enable GVT mode for i915 driver

If you follow the references in the last section you'll find that it seems these days every hardware needs at least some different quirks still.
The ones used and listed here are those I needed.

 * set `drm.debug=0`
 * set `intel_iommu=igfx_off` ([bug in my HW](https://bugs.freedesktop.org/show_bug.cgi?id=110238))

```shell
$ printf "kvmgt\nvfio-iommu-type1\nvfio-mdev" | sudo tee /etc/initramfs-tools/modules
$ printf "i915.enable_gvt=1" | sudo tee /etc/modprobe.d/i915.conf
$ sed -e 's/^GRUB_CMDLINE_LINUX_DEFAULT="/&intel_iommu=on drm.debug=0 intel_iommu=igfx_off /' /etc/default/grub
$ sudo update-initramfs -u
$ sudo update-grub
```

After reboot your IOMMU should be up and running

```shell
$ dmesg | grep -i -e iommu -e i915 -e gvt                                                          
...
[    0.108458] DMAR: IOMMU enabled
[    0.173256] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
[    0.883736] iommu: Adding device 0000:00:00.0 to group 0
...
[   11.758688] i915 0000:00:02.0: MDEV: Registered
```

The next thing you need to do is to actually *carve out* a mediated device from your real GPU.
On the i915 card that is done via sysfs.
You want to:

* check which PCI ID your GPU has
* create a new UUID to use
* register a new mediated device



```shell
$ lspci -v | grep -i vga
00:02.0 VGA compatible controller: Intel Corporation HD Graphics 6000
        (rev 09) (prog-if 00 [VGA controller])

# on 02 we can then see supported types
$ ll /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types
drwxr-xr-x  3 root root 0 Apr 17 15:07 i915-GVTg_V4_4/
drwxr-xr-x  3 root root 0 Apr 17 15:07 i915-GVTg_V4_8/

# register a new one
$ uuid
4dd50f26-ec08-11e8-b838-4bc3356865b6
$ cd /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V4_4
$ echo 4dd50f26-ec08-11e8-b838-4bc3356865b6 | sudo tee create

# check the new device exists
ll /sys/bus/mdev/devices/4dd50f26-ec08-11e8-b838-4bc3356865b6
lrwxrwxrwx 1 root root 0 Apr 17 15:08
  /sys/bus/mdev/devices/4dd50f26-ec08-11e8-b838-4bc3356865b6 ->
  ../../../devices/pci0000:00/0000:00:02.0/4dd50f26-ec08-11e8-b838-4bc3356865b6/
```


### Pass the Mediated Device to the guest ###

Eventually it is qemu that will do this job and in qemu command line that would look like:

```shell
...
-nodefaults
-M graphics=off
-display gtk,gl=on
-device vfio-pci,sysfsdev=/sys/bus/pci/devices/0000:00:02.0/4dd511f6-ec08-11e8-b839-2f163ddee3b3,display=on,rombar=0
...
```

[Kraxels blog](https://www.kraxel.org/blog/2018/04/vgpu-display-support-finally-merged-upstream/) explains the reason for each of those nicely, no need to repeat that.

But only a few people really prefer qemu command line, instead let use use libvirt to pass this device.
And in addition I wanted to have it run by libvirt to benefit from the [apparmor profile support that got added for GL and MDEV features](https://wiki.ubuntu.com/DiscoDingo/ReleaseNotes#libvirt).
Use whatever tool you like, I usually edit directly in `virsh`, but you could most likely could change the same in `virt-manager`.
Essentially what you need to do is:

* enable GL on a spice graphic
* disable listening (incompatible with GL)
* add the mediated device as hostdev

In libvirt XML for my example that looks like:

```xml
<graphics type='spice'>
  <listen type='none'/>
  <gl enable='yes'/>
</graphics>
<hostdev mode='subsystem' type='mdev' managed='no' model='vfio-pci'>
  <source>
    <address uuid='4dd50f26-ec08-11e8-b838-4bc3356865b6'/>
  </source>
</hostdev>
```

### Use the GPU in the guest ###

Other than a full PCI passthrough of a GPU the hosts - where the guest will output graphics on the external ports - my Host still runs all it's UI on the HDMI out of the graphics card we just passed to the guest.
I have not yet had the pleasure to use actual Graphics on those devices in the guest (needs to be arbitrated, but I have yet to do so - maybe one can assign ports one day).

But they can already be used to run some accelerated open-CL load which is much more interesting to me anyway.
First of all take a look at the card in the guest, it is a *normal* PCI device.

```shell
$ lspci -s 07
00:07.0 VGA compatible controller: Intel Corporation HD Graphics 6000 (rev 09)
```

Using beignet for i915 open-CL shows the card as supported

```shell
$ sudo apt install beignet
$ clinfo
Number of platforms                               1
  Platform Name                                   Intel Gen OCL Driver
  Platform Vendor                                 Intel
  Platform Version                                OpenCL 2.0 beignet 1.3
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_byte_addressable_store cl_khr_3d_image_writes cl_khr_image2d_from_buffer cl_khr_depth_images cl_khr_spir cl_khr_icd cl_intel_accelerator cl_intel_subgroups cl_intel_subgroups_short cl_khr_gl_sharing
  Platform Extensions function suffix             Intel
...
Number of devices                                 1
  Device Name                                     Intel(R) HD Graphics 6000 BroadWell U-Processor GT3
...
```

And for the sake of a trivial (read non full AI stack) test program you might look at [jcupitt](https://github.com/jcupitt/opencl-experiments/blob/master/README.md).

```shell
$ sudo apt install beignet-dev
$ wget https://codeload.github.com/hpc12/tools/tar.gz/master
$ tar xvf master
$ cd tools-master/
$ make

$ ./cl-demo 1000000 1000
NAME: Intel(R) HD Graphics 6000 BroadWell U-Processor GT3
...
VERSION: OpenCL 1.2 beignet 1.3
EXTENSIONS: cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_byte_addressable_store cl_khr_3d_image_writes cl_khr_image2d_from_buffer cl_khr_depth_images cl_khr_spir cl_khr_icd cl_intel_accelerator cl_intel_subgroups cl_intel_subgroups_short cl_khr_gl_sharing cl_khr_fp16
...
---------------------------------------------------------------------
0.001428 s
8.404181 GB/s
GOOD
```

I'm sure with better GPUs and real use cases that can be very helpful.
Especially since there are no Virtual Functions for GPUs that I'd know of that finally provides a 1:N relation for GPU passthrough.

More on that topic next time when I got hold of more powerful devices.
Until then I really like that my 4 year old NUC with integrated graphics already allowed me to experiment on `mediated devices`

### Outlook and next steps ###

If you watch the mailing lists you will see that there are more types of mediated devices discussed, I'm sure more of them are coming.
Even very different things like [s390x vfio-ap](https://patchwork.kernel.org/cover/10634859/) was implemented using mediated devices.
It is exiting to see what other devices and what mdev features will be added next - [mdev+vfio](https://events.linuxfoundation.org/wp-content/uploads/2017/12/Hardware-Assisted-Mediated-Pass-Through-with-VFIO-Kevin-Tian-Intel.pdf) or [vhost-mdev](https://events.linuxfoundation.org/wp-content/uploads/2017/12/Cunming-Liang-Intel-KVM-Forum-2018-VDPA-VHOST-MDEV.pdf), who knows?

### Further References: ###

* [Kraxels blog when the feature hit upstream](https://www.kraxel.org/blog/2018/04/vgpu-display-support-finally-merged-upstream/) (great and brief reference)
* [Intel GVT setup](https://github.com/intel/gvt-linux/wiki/GVTg_Setup_Guide)
* [Kernel vfio mdev doc](https://www.kernel.org/doc/Documentation/vfio-mediated-device.txt)
* [vGPU intro Slide deck](https://www.linux-kvm.org/images/5/59/02x03-Neo_Jia_and_Kirti_Wankhede-vGPU_on_KVM-A_VFIO_based_Framework.pdf)
* early Docs on [mediated devices](https://www.infradead.org/~mchehab/kernel_docs/unsorted/vfio-mediated-device.html)
* [Nvidia documentation](https://docs.nvidia.com/grid/latest/grid-vgpu-release-notes-red-hat-el-kvm/index.html)
