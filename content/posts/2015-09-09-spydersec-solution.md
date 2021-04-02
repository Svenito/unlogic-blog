---
extends: default.liquid
title: SpyderSec solution
date: 2015-09-09T19:03:50Z
---

Another day, another VM. Today it's the [SpyderSec Challenge](https://www.vulnhub.com/entry/spydersec-challenge,128/)

So let me start it up and get on it.

![image](https://i.imgur.com/nah3Wah.gif)

As per usual I need the IP of the machine and the services it has running (if any). Straight from the Unlogic Cookbook

```sourceCode
root@kali:~/Downloads# nmap -sn 192.168.56.0/24

Starting Nmap 6.49BETA4 ( https://nmap.org ) at 2015-09-09 16:07 BST
Nmap scan report for 192.168.56.1
Host is up (0.00039s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100
Host is up (0.00017s latency).
MAC Address: 08:00:27:FF:57:41 (Cadmus Computer Systems)
Nmap scan report for 192.168.56.101
Host is up (0.00028s latency).
MAC Address: 08:00:27:56:11:10 (Cadmus Computer Systems)
Nmap scan report for 192.168.56.102
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 1.77 seconds

root@kali:~/Downloads# nmap -p- -sV 192.168.56.101

Starting Nmap 6.49BETA4 ( https://nmap.org ) at 2015-09-09 16:07 BST
Nmap scan report for 192.168.56.101
Host is up (0.00049s latency).
Not shown: 65533 filtered ports
PORT   STATE  SERVICE VERSION
22/tcp closed ssh
80/tcp open   http    Apache httpd
MAC Address: 08:00:27:56:11:10 (Cadmus Computer Systems)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 155.80 seconds
```

Lucky me, only one service running, and it's good old http at that. Let's take a look at that page then:

[<img src="https://i.imgur.com/IiA6MlY.png" alt="image" width="500" />](https://i.imgur.com/IiA6MlY.png)

Without wasting much time, let's get to the clue hunting. First things first: check the source. And there's clue number one, right between those script tags. It evals a function which seems to do some text processing. I'll open Firebug to see if that shows anything interesting, and sure enough

```sourceCode
SyntaxError: missing ; before statement
    61:6c:65:72:74:28:27:6d:75:6c:64:65:72:2e:66:62:69:27:29:3b
```

So that's the output from the eval. It throws an error because the result isn't valid javascript. It seems to me as though it might be HEX. I'll put it through Burp's decoder and sure enough it resolves to `a:l:e:r:t:(:':m:u:l:d:e:r:.:f:b:i:':):;`. That's not going to affect the page in any way, but I'm sure it's a clue. I'll note it down and carry on exploring.

The CSS contains a base64 encoded gif. I decided to investigate it by converting it to a file and opening it in Gimp to examine it. Nothing much of interest there either. Hexdump also shows nothing of note.

Ok then, apart from that there's nothing of interest in the source, so let me move onto the images on the page. There's two images: `Challenge.png` and `SpyderSecLogo200.png`. On first glance they appear to have nothing special about them, but once examined with `exiftool` I see something of interest

```sourceCode
root@kali:~/spydersec# exiftool Challenge.png
ExifTool Version Number         : 9.74
File Name                       : Challenge.png
Directory                       : .
File Size                       : 83 kB
File Modification Date/Time     : 2015:09:01 07:25:59+01:00
File Access Date/Time           : 2015:09:09 14:29:25+01:00
File Inode Change Date/Time     : 2015:09:09 14:29:19+01:00
File Permissions                : rw-r--r--
File Type                       : PNG
MIME Type                       : image/png
Image Width                     : 540
Image Height                    : 540
Bit Depth                       : 8
Color Type                      : RGB with Alpha
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Background Color                : 255 255 255
Pixels Per Unit X               : 2835
Pixels Per Unit Y               : 2835
Pixel Units                     : meters
Comment                         : 35:31:3a:35:33:3a:34:36:3a:35:37:3a:36:34:3a:35:38:3a:33:35:3a:
                                  37:31:3a:36:34:3a:34:35:3a:36:37:3a:36:61:3a:34:65:3a:37:61:3a:
                                  34:39:3a:33:35:3a:36:33:3a:33:30:3a:37:38:3a:34:32:3a:34:66:3a:
                                  33:32:3a:36:37:3a:33:30:3a:34:61:3a:35:31:3a:33:64:3a:33:64
Image Size                      : 540x540
```

Hex strings are the order of the day here at SpyderSec. So back to Burp's decoder once more after removing all the colons. The string decodes to another hex string. Same dance again, and I get a typical base64 string, decode once more and be rewarded with `A!Vu~jtH#729sLA;h4%`. Which is not encoded anymore. I make a note of it and carry on sleuthing.

Watch out, here comes the reliable `dirbuster`. Running it with the regular word list I discover the `v` subdirectory. Browsing to that however merely responds with a `403 Forbidden` reply. I've not exhausted all the nooks and crannies yet, there's still that cookie jar to poke at.

```sourceCode
Firebug -> Cookie tab -> URI /v/81JHPbvyEQ8729161jd6aKQ0N4/
```

Another clue.... leading me to a subdirectory under `v`. But that's also forbidden. Well, let me just plug some of the data we've found so far into it. The random characters from the exif data result in a 404, but the string from the javascript alert box however brings up a download dialog for a file called `mulder.fbi`.

```sourceCode
root@kali:~/spydersec# wget https://192.168.56.101//v/81JHPbvyEQ8729161jd6aKQ0N4/mulder.fbi
--2015-09-09 17:24:38--  https://192.168.56.101//v/81JHPbvyEQ8729161jd6aKQ0N4/mulder.fbi
Connecting to 192.168.56.101:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 13960421 (13M) [text/plain]
Saving to: ‘mulder.fbi’

mulder.fbi.1            100%[===============================>]  13.31M  5.52MB/s   in 2.4s

2015-09-09 17:24:41 (5.52 MB/s) - ‘mulder.fbi’ saved [13960421/13960421]

root@kali:~/spydersec# file mulder.fbi
mulder.fbi: ISO Media, MP4 v2 [ISO 14496-14]
```

A video file, which when I play it, is the song "Twilight Time" by "The Platters".

So here I hit another dead end.

Let me think

![image](https://i.imgur.com/CbfWCmv.gif)

I'll take inventory of the clues I have left now:

- A music video "Twilight time" titled _mulder.fbi_
- A seeming random string `A!Vu~jtH#729sLA;h4%`

So I ask myself, why is a video of a song named _mulder.fbi_? So I do a little research and after searching for `the platters "twilight time" x files` I hit this section in a [Wikipedia article](<https://en.wikipedia.org/wiki/Kill_Switch_(The_X-Files)>)

```sourceCode
When he puts it into the car stereo, it plays "Twilight Time"
by The Platters. However, the agents take it to the Lone Gunmen,
who discover that the disc contains a large quantity of encrypted data
```

Well if that ain't a clue and a bit! Ok, so a little more research of what data you can hide in a video file (search for `hiding files video mp4`) I am directed to a [Lifehacker article](https://lifehacker.com/5771142/embed-a-truecrypt-volume-in-a-playable-video-file) describing the process of hiding Truecrypt volumes in MP4s. It mentions a few ways to detect such a volume in a video, but to be honest, I might as well just try and mount the volume. That should be the easiest and quickest way to see if I am on the right track.

Sure enough, there's a volume in the video, but it needs a password. Well there's only one unused piece of the puzzle left. I plug that in and there's our volume with the `flag.txt` file which contains:

```sourceCode
Congratulations!

You are a winner.

Please leave some feedback on your thoughts regarding this challenge.
Was it fun? Was it hard enough or too easy?
What did you like or dislike, what could be done better?

https://www.spydersec.com/feedback
```

Well that was a nice challenge, especially the truecrypt volume in the MP4. That's something new I learned from this. So thanks to [@SpyderSec](https://twitter.com/Spydersec) for the challenge, and thanks to you for stopping by to read this.
