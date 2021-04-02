---
extends: default.liquid
title: nmap's idle scan with scapy
date: 2014-07-17T21:03:50Z
---

If you've used nmap before you are probably familiar with its idle scan feature. Should you not be, go and [skim the docs](https://nmap.org/book/idlescan.html). I'll quickly cover the basics here anyway.

Basically it will allow us to conduct a port scan of a remote host without revealing our IP address. This is done by making use of the fact that IP IDs are sequential. The images below (courtesy of the namp project documentation) explain this very well.

**Open Port**

[<img src="https://i.imgur.com/MajmRTH.png" alt="image" width="400" />](https://i.imgur.com/MajmRTH.png)

**Closed Port**

[<img src="https://i.imgur.com/2UALzzd.png" alt="image" width="400" />](https://i.imgur.com/2UALzzd.png)

**Filtered Port**

[<img src="https://i.imgur.com/Yv6hbwL.png" alt="image" width="400" />](https://i.imgur.com/Yv6hbwL.png)

Ok, so that's the short version of the theory and it's relatively painless to implement with scapy. Let's take a look at the script:

<script src="https://gist.github.com/Svenito/e377713b90525e842266.js"></script>

You provide the script with a `zombie ip`, `target ip`, and `target port`. These should be self explanatory. Then we use scapy to assemble our packets to conduct our scan. The `p1` SYN/ACK packet will be sent to the zombie so we can obtain a current IP ID from the RST response as a reference.

`p2` is a SYN packet which we send to the target with a return address of the zombie host. When the target receives this packet it will reply to the zombie instead of the source machine (us). The target will respond with a SYN/ACK to the `p2` packet, but send its response to the zombie instead of us. Thus increasing the zobmies IP ID value by one.

Then we probe the zombie again with a further SYN/ACK packet (this is `p3`) to find out what the new IP ID is.

Now we have enough information to determine if the remote port is open or not. We subtract the initial IP ID from the final one and print out the status of the port.

So how do you use it? The important thing is to find a relative inactive machine to use as a zombie, as you don't want the IP ID to change while you are conducting your probe. I've had the best results with using networked printers as zombie hosts. Unless there's a lot of print jobs going on, they are fairly idle. The printer is at `10.20.7.1` and I'm probing port 80 on `10.16.70.8`.

```sourceCode
$] sudo ./idle_scan.py 10.20.7.1 0.16.70.8 80
WARNING: No route found for IPv6 destination :: (no default route?)

[*] Scan 10.16.70.8 port 80 using 10.20.7.1 as zombie
[+] Zombie initial IP id 57319
[+] Zombie final IP id 57321
[+] Port 80 : open

sudo ./idle_scan.py 10.20.7.1 0.16.70.8 90
WARNING: No route found for IPv6 destination :: (no default route?)

[*] Scan 10.16.70.8 port 90 using 10.20.7.1 as zombie
[+] Zombie initial IP id 57341
[+] Zombie final IP id 57342
[+] Port 90 : closed
```

And to be sure, let's verify our findings with a proper nmap scan.

```sourceCode
$] nmap 10.16.70.8 -p80,90

Starting Nmap 5.51 ( https://nmap.org ) at 2014-07-17 13:07 BST
Nmap scan report for 10.16.70.8 (10.16.70.8)
Host is up (0.00013s latency).
PORT   STATE  SERVICE
80/tcp open   http
90/tcp closed dnsix

Nmap done: 1 IP address (1 host up) scanned in 0.04 seconds
```

Approved.
