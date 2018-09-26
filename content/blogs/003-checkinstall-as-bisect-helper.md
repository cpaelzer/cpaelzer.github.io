---
title: "Checkinstall as Bisect Helper"
date: 2018-09-26T16:14:29+02:00
tags: ["packaging", "bisect"]
draft: false
---

# Checkinstall as Bisect Helper #
## a.k.a grml get those bisect bits where I want them ##

### Background on Bisecting ###

If you try to hunt down a particular change that

* introduced an issue to debug it
* that fixed an issue to backport it

then [bisecting](https://git-scm.com/docs/git-bisect) is one of the most common tools to get it done.

And it generally works great as long as you can do a cycle like the following locally:

```lang-none
    perfect world          real world
   +-------------+       +-------------+
   | Bisect      |       | Bisect      |
   |   +         |       |   +         |
   | Build       |       | Build       |
   +----^----+---+       +----^----+---+
        |    |                |    |
        |    |                |    X WTF
        |    |                |    |
   +----+----v---+       +----+----v---+
   |             |       |             |
   | Test        |       | Test        |
   |             |       |             |
   +-------------+       +-------------+
```

But quite often there is a huge gap between building some code and being able to test it.
When bisecting for Ubuntu packaging I often faced two types of challenges which both can be overcome with checkinstall.

Challenges:

* upstream changes - Need to bisect the upstream repo, but that misses packaging bits
* complex deployment - makes it hard to get from build to test e.g. install on multiple systems

I'll elaborate what makes these challenges annoying and then how checkinstall can sometimes help to overcome them.

### Challenge I: you look for upstream changes ###

Ubuntu (and Debian) have almost all of their integration logic in the [debian subdir](http://packaging.ubuntu.com/html/debian-dir-overview.html) of a package.
And since [git-ubuntu](https://blog.ubuntu.com/2017/08/09/git-ubuntu-clone) we can easily switch and compare between different releases and versions or even trivially bisect between Ubuntu uploads.
But quite often you will look for a change that was made upstream instead of a packaging change.

Fortunately these days most upstreams have git repositories you can clone from to bisect.
But those lack the debian/ subdirectory and therefore won't easily build .deb packages as you would need them.

Sometimes you can get away with a recipe build, which merges the debian dir with the upstream repo - but TBH most of the time it fails and it is burdensome in any case.
You often end up wishing you could do just use an upstream projects `make` but that should gives you .deb files.

### Challenge II: complex deployment ###

In many cases you'd think I go to a container and `make install`, but the need for .deb files increases as soon as you are not able to run something locally.
If you need to distribute and install the compiled bits on multiple remote systems `make install` just won't do it without being too error prone and would require you to build on multiple systems or sync build dirs after every build hopeing the build system will not complain on changed file update time.
Again you often end up wishing you could do just use an upstream projects `make` but that should gives you .deb files.

### Checkinstall to the rescue ###

If you are stuck at one of the points above then [checkinstall](http://checkinstall.izto.org/) could help you.
It allows you to run any kind of comman `config + make` process, but instead of `make install` you'd run
`checkinstall -D`

That will call `make install` but track all files moved and create a .deb file for you.
You can configure it to add metadata for you, but most of the time you'd only set a package name and a version.

The steps to use that are

* Prepare environment
```
$ apt build-dep <package>
$ git clone <upstream project>
# common git bisect calls
$ git bisect start <bad> <good>
```

* iterate with bisect/configure/make
```
./configure <options>
make -j
```

* checkinstall to get a .deb and test
```
checkinstall -D
# use the .deb it creates the way you need it and iterate
```

That way I solved above challenges in multple cases, allowing me to find the offending upstream change on one hand using the upstream git `git-bisect`.
While on the other hand using common Ubuntu .deb packages for clean install/upgrade of the versions I iterate on making distributing that to multiple systems quite easy.

Also attaching such a deb to a bug report for a reporter to test - is much cleaner than a raw binary or a tarball.

Hope that might help somebody else as well!

References:

* Ubuntu Community on [checkinstall](https://help.ubuntu.com/community/CheckInstall)
* Debian Community on [checkinstall](https://wiki.debian.org/CheckInstall)
* Ubuntu man page of [checkinstall](http://manpages.ubuntu.com/manpages/cosmic/man8/checkinstall.8.html)
* an [example case](https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1783140/comments/32) bisecting qemu that needed to be pushed on multiple systems to then run migration workload
