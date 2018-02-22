---
title: "003 Chrony With Gps"
date: 2018-02-22T11:26:08+01:00
draft: true
---

### IMPORTANT ###
I was stopped by SDCard issues - keep this as draft until I find time again.
I can't upgrade to 18.04 anymore but really want to use all the new versions for this.
:-/




# Timeberry #
## a.k.a Quality time with a RaspberryPi2 + Chrony ##

### Introduction

In a culmination of coincidences I wanted to test [Chrony](https://bugs.launchpad.net/ubuntu/+source/chrony) with a [PPS device](https://www.kernel.org/doc/Documentation/pps/pps.txt) and the only thing I had around for that was an [Adafruit Ultimate GPS Hat](https://www.adafruit.com/product/2324) as well as a [Raspberry Pi2 B](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/).

This is based on my notes to get that to be a GPS backed local high quality NTP protocol source. This is for fun, I do not (try to) imply anything on the quality of this vs e.g. a good link to a high stratum of the [NTP pool project](http://www.pool.ntp.org)

TODO: add Photo

### Using GPSD

First of all I needed to verify that the GPS Hat actually works. The hardest part was waiting for hours to get a lock, but I didn't get out for better reception (it is cold atm).

On the setup itself adafruit has a nice overview [how to do](https://learn.adafruit.com/adafruit-ultimate-gps-hat-for-raspberry-pi/use-gpsd) so inlcuding checks to the basice function.

So it essentially is:
  $ sudo apt-get install gpsd gpsd-clients python-gps

TODO: Time stats

### Using Chrony

TODO show consumption of GPS in Chrony


TODO More to refer and use:
http://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html
https://learn.adafruit.com/adafruit-ultimate-gps-hat-for-raspberry-pi/use-gpsd
http://www.catb.org/gpsd/gpsd-time-service-howto.html#_feeding_chrony_from_gpsd
http://www.catb.org/gpsd/gpsd-time-service-howto.html#_chrony_performance_tuning
