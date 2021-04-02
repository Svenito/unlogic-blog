---
title: Change MAC automatically in linux
date: 2017-05-07T21:03:50Z
tags:
  - linux
  - networking
  - howto
path: /2017/05/07/linux-mac-address
authors: Sven Steinbauer
summary: Today I'll go over how to automatically change your MAC address in linux and how to do a few other things related to this
---

A little background first. I got a new work laptop and as with a lot of modern laptops, this one also only has an external
ethernet adapter. It kinda worked ok, but the issue with the USB interface is that the MAC address on the dongle, for the
ethernet device is not the one that appears in the OS. You actually receive a pass through MAC address. This in itself is
no big issue unless you are on a network that only dishes out IP addresses based on known MAC addresses, and said engineering
department also has the MAC address from the dongle in the records. Asking engineering to update this is trivial, but involves
waiting. Plus, this is linux, I can work around this right? Sure enough, you can just update your MAC address with

    ifconfig <iface> hw ether XX:XX:XX:XX:XX:XX

Where the `XX:XX:XX:XX:XX:XX` is the desired MAC address. However, I have to keep running that each time I plug in the
device. This wasn't ok. So let's leverage `udev` for this. In a file called `/etc/udev/rules.d/75-mac-spoof.rules` I added
the following:

    ACTION=="add", SUBSYSTEM=="net", ATTR{address}="YY:YY:YY:YY:YY:YY", RUN+="/sbin/ip link set dev %k address XX:XX:XX:XX:XX:XX"

This will replace the `YY` MAC address (original one) with the `XX` address. All ok right? Well no, the `%k` resolves to `eth0`
which is what the OS helpfully renames to something else before this rule runs. Game's not over yet, so let's fix that.

Creating and editing `/etc/udev/rules.d/70-my-net-name.rules` fill it with:

    SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="XX:XX:XX:XX:XX:XX", NAME="eth0"

The MAC address here is the original MAC address, becuase this will run _before_ the rule above.

So tail `syslog` and replug the network device, we should see our device being named `eth0` and then its MAC address change.
Excellent, we're in business and don't have to manually change the MAC address each time. Plus we don't have a stupid name for
our network interface anymore :)
