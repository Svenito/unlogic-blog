---
extends: default.liquid
title: Let's crack Bandit Part 2
date: 2015-03-15T21:03:50Z
summary: Continues on from Let's crack Bandit Part 1
---

Continues on from [Let's crack Bandit Part 1](/posts/2015-03-13-lets-crack-bandit-part1/)

## Level 15 -&gt; 16

[Level 15](https://overthewire.org/wargames/bandit/bandit16.html)

Eventhough this is very similar to the previous level, it's a little more complicated as we need to connect with SSL. The simplest way is using `openssl` with `s_client`. Once connected it's the same dance as above

```sourceCode
bandit15@melinda:~$ openssl s_client -quiet -connect localhost:30001
depth=0 CN = li190-250.members.linode.com
verify error:num=18:self signed certificate
verify return:1
depth=0 CN = li190-250.members.linode.com
verify return:1
BfMYroe26WYalil77FoDi9qh59eK5xNr
Correct!
cluFn7wTiGryunymYOu4RcffSxQluehd

read:errno=0
```

## Level 16 -&gt; 17

[Level 16](https://overthewire.org/wargames/bandit/bandit17.html)

Here we have a choice. We run a simple ping scan across the port range and then figure out which port is the right one by trying each one. Depending on the number of ports open this could take a while or not. Let's see how we're going to handle this by seeing which ports are open

```sourceCode
bandit16@melinda:~$ nmap localhost -p 31000-32000

Starting Nmap 6.40 ( https://nmap.org ) at 2015-03-20 14:54 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00080s latency).
Not shown: 996 closed ports
PORT      STATE SERVICE
31046/tcp open  unknown
31518/tcp open  unknown
31691/tcp open  unknown
31790/tcp open  unknown
31960/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.08 seconds
```

Not too bad. Because it's a short list, we can try them one by one, or we run a service discovery on them. Service discovery in nmap takes a while, so I only scan the ports we are interseted in:

```sourceCode
bandit16@melinda:~$ nmap -sV -p 31046,31518,31691,31790,31960 localhost

Starting Nmap 6.40 ( https://nmap.org ) at 2015-03-20 14:51 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00015s latency).
PORT      STATE SERVICE VERSION
31046/tcp open  echo
31518/tcp open  msdtc   Microsoft Distributed Transaction Coordinator (error)
31691/tcp open  echo
31790/tcp open  msdtc   Microsoft Distributed Transaction Coordinator (error)
31960/tcp open  echo
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Now we only have two ports to try, as the others are clearly just echo ports. Eliminating one we go ahead and

```sourceCode
bandit16@melinda:~$ openssl s_client -quiet -connect localhost:31790
depth=0 CN = li190-250.members.linode.com
verify error:num=18:self signed certificate
verify return:1
depth=0 CN = li190-250.members.linode.com
verify return:1
cluFn7wTiGryunymYOu4RcffSxQluehd
Correct!
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAvmOkuifmMg6HL2YPIOjon6iWfbp7c3jx34YkYWqUH57SUdyJ
imZzeyGC0gtZPGujUSxiJSWI/oTqexh+cAMTSMlOJf7+BrJObArnxd9Y7YT2bRPQ
Ja6Lzb558YW3FZl87ORiO+rW4LCDCNd2lUvLE/GL2GWyuKN0K5iCd5TbtJzEkQTu
DSt2mcNn4rhAL+JFr56o4T6z8WWAW18BR6yGrMq7Q/kALHYW3OekePQAzL0VUYbW
JGTi65CxbCnzc/w4+mqQyvmzpWtMAzJTzAzQxNbkR2MBGySxDLrjg0LWN6sK7wNX
x0YVztz/zbIkPjfkU1jHS+9EbVNj+D1XFOJuaQIDAQABAoIBABagpxpM1aoLWfvD
KHcj10nqcoBc4oE11aFYQwik7xfW+24pRNuDE6SFthOar69jp5RlLwD1NhPx3iBl
J9nOM8OJ0VToum43UOS8YxF8WwhXriYGnc1sskbwpXOUDc9uX4+UESzH22P29ovd
d8WErY0gPxun8pbJLmxkAtWNhpMvfe0050vk9TL5wqbu9AlbssgTcCXkMQnPw9nC
YNN6DDP2lbcBrvgT9YCNL6C+ZKufD52yOQ9qOkwFTEQpjtF4uNtJom+asvlpmS8A
vLY9r60wYSvmZhNqBUrj7lyCtXMIu1kkd4w7F77k+DjHoAXyxcUp1DGL51sOmama
+TOWWgECgYEA8JtPxP0GRJ+IQkX262jM3dEIkza8ky5moIwUqYdsx0NxHgRRhORT
8c8hAuRBb2G82so8vUHk/fur85OEfc9TncnCY2crpoqsghifKLxrLgtT+qDpfZnx
SatLdt8GfQ85yA7hnWWJ2MxF3NaeSDm75Lsm+tBbAiyc9P2jGRNtMSkCgYEAypHd
HCctNi/FwjulhttFx/rHYKhLidZDFYeiE/v45bN4yFm8x7R/b0iE7KaszX+Exdvt
SghaTdcG0Knyw1bpJVyusavPzpaJMjdJ6tcFhVAbAjm7enCIvGCSx+X3l5SiWg0A
R57hJglezIiVjv3aGwHwvlZvtszK6zV6oXFAu0ECgYAbjo46T4hyP5tJi93V5HDi
Ttiek7xRVxUl+iU7rWkGAXFpMLFteQEsRr7PJ/lemmEY5eTDAFMLy9FL2m9oQWCg
R8VdwSk8r9FGLS+9aKcV5PI/WEKlwgXinB3OhYimtiG2Cg5JCqIZFHxD6MjEGOiu
L8ktHMPvodBwNsSBULpG0QKBgBAplTfC1HOnWiMGOU3KPwYWt0O6CdTkmJOmL8Ni
blh9elyZ9FsGxsgtRBXRsqXuz7wtsQAgLHxbdLq/ZJQ7YfzOKU4ZxEnabvXnvWkU
YOdjHdSOoKvDQNWu6ucyLRAWFuISeXw9a/9p7ftpxm0TSgyvmfLF2MIAEwyzRqaM
77pBAoGAMmjmIJdjp+Ez8duyn3ieo36yrttF5NSsJLAbxFpdlc1gvtGCWW+9Cq0b
dxviW8+TFVEBl1O4f7HVm6EpTscdDxU+bCXWkfjuRb7Dy9GOtt9JPsX8MBTakzh3
vBgsyi/sN3RqRBcGU40fOoZyfAMT8s1m/uYv52O6IgeuZ/ujbjY=
-----END RSA PRIVATE KEY-----

read:errno=0
```

Now copy that key into a new file and use `chmod go-rw key` to remove group and other read/write. ssh refuses to accept a key that is read/write by anyone other than the user who owns the file. Then simply

```sourceCode
bandit16@melinda:~$ ssh -i /tmp/k.key bandit17@localhost
```

## Level 17 -&gt; 18

[Level 17](https://overthewire.org/wargames/bandit/bandit18.html)

We remain logged in as bandit17 from the previous level. To compare two files we need to do a `diff`

```sourceCode
bandit17@melinda:~$ diff passwords.old  passwords.new
42c42
< BS8bqB1kqkinKJjuxL6k072Qq9NRwQpR
```
