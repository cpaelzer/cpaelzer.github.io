---
title: "The Ubuntu Virtualization Stack Crystal Ball"
date: 2020-01-20T12:08:27+02:00
tags: ["libvirt", "ubuntu", "qemu", "version"]
draft: true
---

# The Ubuntu Virtualization Stack Crystal Ball #
## a.k.a which version of QEMU and libvirt will be in Ubuntu? ##


## Background

I am asked a lot of times in different contexts which version of QEMU, libvirt,
or other components the next Ubutuntu release will contain. What this post will
address how the upstreams manage versions and then how the Ubuntu project
chooses a version to ship.

## Upstream

First, consider how the two major upstreams of QEMU and libvirt handle
versioning.


### QEMU

QEMU publishes a release schedule with dates for each version:

*   [https://wiki.qemu.org/Planning/5.0](https://wiki.qemu.org/Planning/5.0)
*   [https://wiki.qemu.org/Planning/4.2](https://wiki.qemu.org/Planning/4.2)
*   [https://wiki.qemu.org/Planning/4.1](https://wiki.qemu.org/Planning/4.1)
*   [https://wiki.qemu.org/Planning/4.0](https://wiki.qemu.org/Planning/4.0)
*   [https://wiki.qemu.org/Planning/3.1](https://wiki.qemu.org/Planning/3.1)

The upstream defines a new major release (e.g. 4.0, 5.0, etc.) every year and
then a few minor releases (e.g. 4.1, 4.2) in that same year.

Using the 4.x release as an example, this resulted in the new major release
occurring in April and then two minor releases in August and December.
However, keep in mind these months might vary year to year.


### libvirt

In recent years, libvirt releases are even more regular. Each year will set a
new major number and there are almost monthly releases doing a minor release
increase. See the libvirt [release notes](https://www.libvirt.org/news.html)
for examples

## Ubuntu Planning

For each Ubuntu release, we check the plans of these projects and match it to
the [release schedule](https://wiki.ubuntu.com/FocalFossa/ReleaseSchedule),
specifically the feature freeze dates of Ubuntu.

With these dates and versions in hand the following constraints are taken
into account:

*   Versions should be as new as possible to get the latest features and fixes
    to our users
*   Include at least the libvirt version that _follows_ the QEMU release.
    This is a lesson learned from the past and ensures that most features and
    any quirks needed are in there.
*   Consider availability of other dependent packages like iPXE, slof,
    virt-manager, etc. for compatibility with the newer versions of QEMU and
    libvirt
*   Upstream release dates + packaging time + testing should be well before
    the Ubuntu feature freeze date

Once those considerations are taken into account the versions that need to be
targeted usually become clear.

## Ubuntu Focal Fossa

Using Ubuntu’s next LTS release as an example, the above releases and
constraints led us to choose QEMU version 4.2 (December 2019) and libvirt
version 6.0 (January 2020) as the versions to target.

We prefer providing the newer QEMU 4.2 over 4.1 to ensure the best possible
feature set to our users. However, while QEMU 5.0 may release before Focal it
is too late in the cycle to pick up.


## FAQ

There are a bunch of common further questions, let me try to answer them right
away as well.

Q: _Why does Ubuntu skip QEMU and libvirt versions?_

A: Releasing each version is overall more costly than beneficial. It causes
additional work that is better spent elsewhere to get other components fixed
and updated. Furthermore having “_several quick but potentially error prone
updates_” policy works for some components, but the virt-stack is not one of
them.

The virtualization stack is utilized through the Ubuntu project. Pushing
releases more often increases the chance to slow down or even bring to a
grinding halt the project. If new features require that we need to work on
them in advance we usually do so in
[PPAs](https://help.launchpad.net/Packaging/PPA) without affecting the rest
of Ubuntu.

Q: _Really, that means QEMU might be outdated by 3 months when Ubuntu 20.04
releases?_

A: That is not accurate. It is not outdated but stabilized for three months.
We continue testing more uncommon cases and ensure that features continue to
work. There might be an upstream stable release like 4.2.1 that we will pull
in, but even if there is not we usually go through recent git commits and
cherry pick fixes for stabilization of QEMU/Libvirt before an Ubuntu release
is finalized.

That way you get software at the best spot of the “_As recent as possible,
but also as stable as possible_” tradeoff and from there
[SRU Policy](https://wiki.ubuntu.com/StableReleaseUpdates) will kick in
which means:

*   In a given Ubuntu release updates will focus on fixes and stability
*   Just six months later there will be another Ubuntu release providing the
    newest features

Q: _I want my base system to stay stable (e.g. previous LTS) and I want the
latest virtualization packages. How can I get both?_

A: For exactly that use case in the wider context of Openstack there is
the [Ubuntu Cloud Archive](https://wiki.ubuntu.com/OpenStack/CloudArchive)
which usually backports the most recent virtualization stack from the latest
Ubuntu LTS to previous LTS releases.
