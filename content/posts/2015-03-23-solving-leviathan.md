---
extends: default.liquid
title: Solving Leviathan
date: 2015-03-23T21:03:50Z
---

After having done [Bandit](https://overthewire.org/wargames/bandit/), let's move on to [Leviathan](https://overthewire.org/wargames/leviathan/). None of the levels have hints, so there won't be any links to each of the levels.

Without further ado, let's get cracking.

## Level 0 -&gt; 1

Logging in with `leviathan0:leviathan0` we take a quick look around to see what we have to work with, if anything

```sourceCode
leviathan0@melinda:~$ ls -la
total 24
drwxr-xr-x   3 root       root       4096 Nov 14 10:32 .
drwxr-xr-x 167 root       root       4096 Mar 21 06:46 ..
drwxr-x---   2 leviathan1 leviathan0 4096 Feb 10 18:08 .backup
-rw-r--r--   1 root       root        220 Apr  9  2014 .bash_logout
-rw-r--r--   1 root       root       3637 Apr  9  2014 .bashrc
-rw-r--r--   1 root       root        675 Apr  9  2014 .profile
```

Let's take a look in that `.backup` folder.

```sourceCode
leviathan0@melinda:~$ ls .backup/
bookmarks.html
```

Checking the contents of the file, we see it really is a list of bookmarks. Going on the assumption that the password for the next level is in there, let's grep for _leviathan_ in this file

```sourceCode
leviathan0@melinda:~$ grep leviathan .backup/bookmarks.html
<DT><A HREF="https://leviathan.labs.overthewire.org/passwordus.html | This will be fixed later, the password for leviathan1 is rioGegei8m" ADD_DATE="1155384634" LAST_CHARSET="ISO-8859-1" ID="rdf:#$2wIU71">password to leviathan1</A>
```

And there it is

## Level 1 -&gt; 2

As usual we login, do a `ls -la` and see a `check` binary that is setuid `leviathan2`. Let's run it and see what happens

```sourceCode
leviathan1@melinda:~$ ./check
password: test
Wrong password, Good Bye ...
```

Trying the usual `strings check` shows us a couple of interesting strings: `secret` and `love`. Neither one or both will return success. Hrmm. Ok, let's take a look at the disassembly

```sourceCode
(gdb) disass main
Dump of assembler code for function main:
   0x0804852d <+0>: push   %ebp
   0x0804852e <+1>: mov    %esp,%ebp
   0x08048530 <+3>: and    $0xfffffff0,%esp
   0x08048533 <+6>: sub    $0x30,%esp
   0x08048536 <+9>: mov    %gs:0x14,%eax
   0x0804853c <+15>:    mov    %eax,0x2c(%esp)
   0x08048540 <+19>:    xor    %eax,%eax
   0x08048542 <+21>:    movl   $0x786573,0x18(%esp)
   0x0804854a <+29>:    movl   $0x72636573,0x25(%esp)
   0x08048552 <+37>:    movw   $0x7465,0x29(%esp)
   0x08048559 <+44>:    movb   $0x0,0x2b(%esp)
   0x0804855e <+49>:    movl   $0x646f67,0x1c(%esp)
   0x08048566 <+57>:    movl   $0x65766f6c,0x20(%esp)
   0x0804856e <+65>:    movb   $0x0,0x24(%esp)
   0x08048573 <+70>:    movl   $0x8048680,(%esp)
   0x0804857a <+77>:    call   0x80483c0 <printf@plt>
   0x0804857f <+82>:    call   0x80483d0 <getchar@plt>
   0x08048584 <+87>:    mov    %al,0x14(%esp)
   0x08048588 <+91>:    call   0x80483d0 <getchar@plt>
   0x0804858d <+96>:    mov    %al,0x15(%esp)
   0x08048591 <+100>:   call   0x80483d0 <getchar@plt>
   0x08048596 <+105>:   mov    %al,0x16(%esp)
   0x0804859a <+109>:   movb   $0x0,0x17(%esp)
   0x0804859f <+114>:   lea    0x18(%esp),%eax
   0x080485a3 <+118>:   mov    %eax,0x4(%esp)
   0x080485a7 <+122>:   lea    0x14(%esp),%eax
   0x080485ab <+126>:   mov    %eax,(%esp)
   0x080485ae <+129>:   call   0x80483b0 <strcmp@plt>
   0x080485b3 <+134>:   test   %eax,%eax
   0x080485b5 <+136>:   jne    0x80485c5 <main+152>
   0x080485b7 <+138>:   movl   $0x804868b,(%esp)
   0x080485be <+145>:   call   0x8048400 <system@plt>
   0x080485c3 <+150>:   jmp    0x80485d1 <main+164>
   0x080485c5 <+152>:   movl   $0x8048693,(%esp)
   0x080485cc <+159>:   call   0x80483f0 <puts@plt>
   0x080485d1 <+164>:   mov    $0x0,%eax
   0x080485d6 <+169>:   mov    0x2c(%esp),%edx
   0x080485da <+173>:   xor    %gs:0x14,%edx
   0x080485e1 <+180>:   je     0x80485e8 <main+187>
   0x080485e3 <+182>:   call   0x80483e0 <__stack_chk_fail@plt>
   0x080485e8 <+187>:   leave
   0x080485e9 <+188>:   ret
End of assembler dump.
```

So it uses `strcmp` to compare our input to whatever the right pass is (`0x080485ae`) I'm going to use `ltrace` to trace through the library call and see what that reveals

```sourceCode
leviathan1@melinda:~$ ltrace ./check
__libc_start_main(0x804852d, 1, 0xffffd794, 0x80485f0 <unfinished ...>
printf("password: ")                                       = 10
getchar(0x8048680, 47, 0x804a000, 0x8048642password: test
)               = 116
getchar(0x8048680, 47, 0x804a000, 0x8048642)               = 101
getchar(0x8048680, 47, 0x804a000, 0x8048642)               = 115
strcmp("tes", "sex")                                       = 1
puts("Wrong password, Good Bye ..."Wrong password, Good Bye ...
)                       = 29
+++ exited (status 0) +++
```

And there we have it.

```sourceCode
leviathan1@melinda:~$ ./check
password: sex
$ whoami
leviathan2
$ cat /etc/leviathan_pass/leviathan2
ougahZi8Ta
```

## Level 2 -&gt; 3

This time we are given a file called `printfile` that is setuid `leviathan3`. Initially you'd think we can just print the contents of the `leviathan3` password file. Nope, there's a check in the binary preventing us from doing so.

Let's find out what that check is with `ltrace` once again

```sourceCode
leviathan2@melinda:~$ ltrace ./printfile /tmp/unlogic
__libc_start_main(0x804852d, 2, 0xffffd764, 0x8048600 <unfinished ...>
access("/tmp/unlogic", 4)                                  = 0
snprintf("/bin/cat /tmp/unlogic", 511, "/bin/cat %s", "/tmp/unlogic") = 21
system("/bin/cat /tmp/unlogic"testing
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                     = 0
+++ exited (status 0) +++
```

We can see that it checks access to the file, then runs `cat` on the file if it's ok for us to access it. Symlinks won't work here, as the `access` call dereferences the symlink. So what can we do? We exploit spaces. By creating a file that is a symlink to the `leviathan3` password file, along with another file, that has the same name followed by a space and another name, we can trick `access` into allowing it to carry on, and then `cat` to print the files. let me show you

```sourceCode
leviathan2@melinda:~$ ln -s /etc/leviathan_pass/leviathan3 /tmp/levpass3
leviathan2@melinda:~$ touch /tmp/levpass3\ other
leviathan2@melinda:~$ ./printfile /tmp/levpass3\ other
Ahdiemoo1j
/bin/cat: other: No such file or directory
```

So access checks `/tmp/levpass3\ other` and deems it ok. Then that string gets passed to `cat` which interprets it as two files, hence the `/bin/cat: other: No such file or directory`

## Level 3 -&gt; 4

Another program that prompts for a pass. Usual approaches of `strings` and checking the _disass_ doesn't reveal much, but the function `do_stuff` does call `strcmp` and we know now that we can use `ltrace` to help us out

```sourceCode
leviathan3@melinda:~$ ltrace ./level3
__libc_start_main(0x80485fe, 1, 0xffffd794, 0x80486d0 <unfinished ...>
strcmp("h0no33", "kakaka")                                 = -1
printf("Enter the password> ")                             = 20
fgets(Enter the password> d
"d\n", 256, 0xf7fcac20)                              = 0xffffd58c
strcmp("d\n", "snlprintf\n")                               = -1
puts("bzzzzzzzzap. WRONG"bzzzzzzzzap. WRONG
)                                 = 19
+++ exited (status 0) +++
```

A little bit of obfuscation here, but to our keen eyes, we see where the test is happening `strcmp("d\n", "snlprintf\n")`. Our password is `snlprintf`.

```sourceCode
leviathan3@melinda:~$ ./level3
Enter the password> snlprintf
[You've got shell]!
$ whoami
leviathan4
$ cat /etc/leviathan_pass/leviathan4
vuH0coox6m
```

## Level 4 -&gt; 5

Inside the hidden directory (you always run `ls -la`, right?) we have a bin file. It's executable, so let's run it

```sourceCode
leviathan4@melinda:~$ ./.trash/bin
01010100 01101001 01110100 01101000 00110100 01100011 01101111 01101011 01100101 01101001 00001010
```

I'm guessing we need to decode that from the current binary encoded string to text. This gives us `Tith4cokei`. Testing it out takes us to

## Level 5 -&gt; 6

We have a binary called `leviathan5` that is suid `leviathan6`. Upon running it, we get a message that file `tmp/file.log` cannot be found. If you create one, it will open it, print its contents, close it, and then delete it. So let's give the old symlink method a try:

```sourceCode
leviathan5@melinda:~$ ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log
leviathan5@melinda:~$ ./leviathan5
UgaoFee4li
```

Result.

## Level 6 -&gt; 7

We need a 4 digit pass code to access this. I opted for brute force. For 4 digits that's by far the simplest and quickest way. After looking at the disassembly, we see that it will call `/bin/sh` and drop us to a shell, so we don't need an exit condition.

```sourceCode
leviathan6@melinda:~$ for i in $(seq -f "%04g" 0 9999); do echo $i && ./leviathan6 $i > /dev/null; done
0000
0001
.
.
7123
```

It stops there. Because we redirect to /dev/null, we need to ctrl+c and then enter the last printed number to get the password

```sourceCode
leviathan6@melinda:~$ ./leviathan6  7123
$ cat /etc/leviathan_pass/leviathan7
ahy7MaeBo9
```

The final flag for level 7 is

> Well Done, you seem to have used a \* nix system before, now try something more serious. (Please don't post writeups, solutions or spoilers about the games on the web. Thank you!)

I understand the reasons, but this is not the only write up out there, and it is a fairly old wargame too. IMO I feel that providing these walkthroughs will help those who are stuck. If you are however just following this so that you can complete Leviathan, then you should sit down and have a go at this, or other war games without a guide. Challenge yourself.
