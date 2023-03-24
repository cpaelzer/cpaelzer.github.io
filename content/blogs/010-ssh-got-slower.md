---
title: "SSH login got slower, be happy!"
date: 2023-03-24T11:55:27+02:00
tags: ["ssh", "Ubuntu", "performance", "security", "post-quantum"]
draft: false
---

# "SSH login got slower, be happy!"
## a.k.a.: a quantum slower against quantum attacks ##

We regularly gather all kind of metrics to ensure Ubuntu is/stays/gets
(depends on your POV) great. One of the metrics that I wanted to look at
for way too long, but the change was too small to do so earlier was our
check on non interactive ssh login speed.

In that test we check how quickly you can log into ssh, do nothing and return
which was a lessons learned from [a bug years ago](https://bugs.launchpad.net/ubuntu/+source/update-motd/+bug/1893716)
that it is better to watch for. To be fair it is a corner case as most people
do something after login and that outweighs things by several order of
magnitude.

So while no one complained and most likely only very few even noticed, I've
looked deeper into a regression Ubuntu 22.10 Kinetic of about 100ms-150ms.
Tracking that down wasn't hard, but interesting.


## The Problem

```shell
# Jammy
ubuntu:~$ hyperfine --style=basic --warmup 10 --runs=500 --export-json=results-warm.json "ssh -o StrictHostKeyChecking=accept-new localhost true"
Benchmark 1: ssh -o StrictHostKeyChecking=accept-new localhost true
  Time (mean ± σ):     250.9 ms ±  21.6 ms    [User: 27.3 ms, System: 4.7 ms]
  Range (min … max):   218.9 ms … 394.4 ms    500 runs

# Lunar
ubuntu:~$ hyperfine --style=basic --warmup 10 --runs=500 --export-json=results-warm.json "ssh -o StrictHostKeyChecking=accept-new localhost true"
Benchmark 1: ssh -o StrictHostKeyChecking=accept-new localhost true
  Time (mean ± σ):     400.1 ms ±  32.1 ms    [User: 164.2 ms, System: 6.1 ms]
  Range (min … max):   354.4 ms … 547.0 ms    500 runs
```

Along that slowdown there also is quite an increase in Userspace CPU
consumption by about x6.

Due to the old issue I was first looking at things like MOTD, but that was
unchanged, so what now ...?


## "Follow the Money" -> "Follow the CPU"

Every performance engineer will be happy to talk hours about why it is great
when a regression goes along a CPU consumption change :-) The tech to track
that is established and mature (and much nicer than these finicky jitter, delay
or similar timing issues.

In this case `perf` quickly pointed to the guilty being `ssh` and since
I ran server and host on the same system, I mean client side `ssh`, not the
servers `sshd` (that will be important later).

```
Jammy                      Lunar
43.58%  swapper            41.63%  ssh
17.60%  sshd               27.60%  swapper
11.77%  ssh                14.29%  sshd
 2.33%  systemd             1.61%  systemd
 2.14%  dbus-daemon         1.42%  dbus-daemon
 1.66%  systemd-logind      1.09%  systemd-logind
 1.47%  run-parts           0.92%  run-parts
```

## Did ssh betray us?

Wondering if `ssh` hates us and stabbed us in the back wasting precious CPU
cycles I decided to ask before judging (generally recommended).

Running the same test in with `-vv` set quickly shows, all is mostly the same
except one little but important detail - [KEX](https://acronyms.thefreedictionary.com/KEX).
If you are a confused banking expert, no - not the Kansai Commodities Exchange,
this is Key Exchange.

```
# Jammy
debug1: kex: algorithm: curve25519-sha256

# Lunar
debug1: kex: algorithm: sntrup761x25519-sha512@openssh.com
```

And you might think WTH sntr..what I can't even pronounce this, what is it?
First of all, this isn't entirely new - Jammy supports the same algorithm as
`ssh` is happy to tell you the same on both systems compared:

```shell
$ ssh -Q kex
diffie-hellman-group1-sha1
diffie-hellman-group14-sha1
diffie-hellman-group14-sha256
diffie-hellman-group16-sha512
diffie-hellman-group18-sha512
diffie-hellman-group-exchange-sha1
diffie-hellman-group-exchange-sha256
ecdh-sha2-nistp256
ecdh-sha2-nistp384
ecdh-sha2-nistp521
curve25519-sha256
curve25519-sha256@libssh.org
sntrup761x25519-sha512@openssh.com
```

Furthermore setting both environments to use the same KEX via
`-oKexAlgorithms=curve25519-sha256` or
`-oKexAlgorithms=sntrup761x25519-sha512@openssh.com` makes them both on deliver
the very same result in regard to timing and CPU consumption.

As usual once you actually know what you search it is easy to find.
The change comes from [this upstream commit](https://anongit.mindrot.org/openssh.git/commit/?id=32dc1c29a4ac)
that switched the default.
And this is the big deal, `ssh` of course did not betray us. It is an
intentional change that is important to be done now to be safe in the future.

## Matters of scale

I mentioned above that it will be important that these extra CPU cycles
are mostly consumed on the client side. The reason for that is that a server
potentially handles a lot of connections and thereby might be overloaded
by the extra effort.

But spending the time on the client(s) is great as that will naturally scale
with the amount of connections.

## "Post quantum" is not a letter

There has been a looming doom in IT due to the "quantum computer vs encryption"
fight. And while those shiny super-cool(ed) systems didn't win yet, the strategy
of [store-now-decrypt-later](https://www.siliconrepublic.com/enterprise/quantum-apocalypse-store-now-decrypt-later-encryption)
makes it important that you prevent it now to be safe in the future.

If you want a quick, well presented summary on the general topic you might
enjoy [the recent summary](https://www.youtube.com/watch?v=-UrdExQW0cs)
by the always great Veritasium channel.

And yeah, that is the reason that you should be happy to have a slightly slower
login now - enjoy the (usually short) feeling of safety.
