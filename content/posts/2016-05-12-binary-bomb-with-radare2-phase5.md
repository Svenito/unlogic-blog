---
extends: default.liquid
title: Binary Bomb with Radare2 - Phase 5
date: 2016-05-12T19:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

Recursion eh? Aren't you glad we've got [phase 4](/2016/05/05/binary-bomb-with-radare2-phase4) behind us? We're going to get a little rest with phase 5, as there are no new constructs here, just some stringy and loopy stuff. Plus, there'll be a debugging session at the end. Oh the glamour, the joy.

## Static peeking

Let's load her up, analyse, and get some disassembly going...

```sourceCode
$ r2 bomb
-- I am Pentium of Borg. Division is futile. You will be approximated.
[0x080488e0]> aa
[x] Analyze all flags starting with sym. and entry0 (aa)
[0x080488e0]> pdf @ sym.phase_5
/ (fcn) sym.phase_5 105
|           ; arg int arg_8h @ ebp+0x8
|           ; var int local_2h @ ebp-0x2
|           ; var int local_8h @ ebp-0x8
|           ; var int local_18h @ ebp-0x18
|           ; CALL XREF from 0x08048ae7 (sym.main)
|           0x08048d2c      55             push ebp
|           0x08048d2d      89e5           mov ebp, esp
|           0x08048d2f      83ec10         sub esp, 0x10
|           0x08048d32      56             push esi
|           0x08048d33      53             push ebx
|           0x08048d34      8b5d08         mov ebx, dword [ebp + arg_8h] ; [0x8:4]=0
|           0x08048d37      83c4f4         add esp, -0xc
|           0x08048d3a      53             push ebx
|           0x08048d3b      e8d8020000     call sym.string_length
|           0x08048d40      83c410         add esp, 0x10
|           0x08048d43      83f806         cmp eax, 6
|       ,=< 0x08048d46      7405           je 0x8048d4d
|       |   0x08048d48      e8af070000     call sym.explode_bomb
|       `-> 0x08048d4d      31d2           xor edx, edx
|           0x08048d4f      8d4df8         lea ecx, [ebp - local_8h]
|           0x08048d52      be20b20408     mov esi, str.isrveawhobpnutfg ; "isrveawhobpnutfg.." @ 0x804b220
|       .-> 0x08048d57      8a041a         mov al, byte [edx + ebx]
|       |   0x08048d5a      240f           and al, 0xf
|       |   0x08048d5c      0fbec0         movsx eax, al
|       |   0x08048d5f      8a0430         mov al, byte [eax + esi]
|       |   0x08048d62      88040a         mov byte [edx + ecx], al
|       |   0x08048d65      42             inc edx
|       |   0x08048d66      83fa05         cmp edx, 5
|       `=< 0x08048d69      7eec           jle 0x8048d57
|           0x08048d6b      c645fe00       mov byte [ebp - local_2h], 0
|           0x08048d6f      83c4f8         add esp, -8
|           0x08048d72      680b980408     push str.giants ; str.giants ; "giants" @ 0x804980b
|           0x08048d77      8d45f8         lea eax, [ebp - local_8h]
|           0x08048d7a      50             push eax
|           0x08048d7b      e8b0020000     call sym.strings_not_equal
|           0x08048d80      83c410         add esp, 0x10
|           0x08048d83      85c0           test eax, eax
|       ,=< 0x08048d85      7405           je 0x8048d8c
|       |   0x08048d87      e870070000     call sym.explode_bomb
|       `-> 0x08048d8c      8d65e8         lea esp, [ebp - local_18h]
|           0x08048d8f      5b             pop ebx
|           0x08048d90      5e             pop esi
|           0x08048d91      89ec           mov esp, ebp
|           0x08048d93      5d             pop ebp
\           0x08048d94      c3             ret
```

You see lines 17 and 24? Remember that? Our good buddy the conditional loop. Radare was also kind enough to provide us with some info on some of the hard coded strings. We'll need to take a closer look at the one on line 16, hard to tell what's going on here from the first glance. The second one however, at line 27 is an argument to _sym.strings_not_equal_. I'll wager one internet that this is somehow related to our input, especially as the resulting test jumps over the _sym.explode_bomb_.

Ok, so we've had our sneak peek, let's do some proper analysis, starting with line 6, which is just after the function prologue, and just before our input gets stored in `ebx`. This is then pushed onto the stack and forms the argument to the call to _sym.string_length_. We're lucky that we can see all these functions, and that they have easy identifiable names, otherwise we'd have to poke around in there too. For now, we'll assume that the nice people who wrote this used appropriate function names. The return value (now in `eax`) is compared to the immediate value of _6_. The following jcc (conditional jump) is `je`, which means we're ok as long as our string is 6 characters long. No more, no less.

We'll jump over _sym.explode_bomb_ and we zero out `edx` by _xoring_ it with itself. The address of a local variable is then stored in `ecx`, so `ecx` is a pointer to that variable. Likely going to be used to store something, but we'll see. Next up the string `isrveawhobpnutfg` is stored in `esi`. Then a single byte is stored in `al`.

I haven't really covered registers properly, so for those who don't know what `al` is, it's the lower byte of the `eax` register. The `eax` register is composed of two smaller registers: _ah_ and _al_. For of the general purpose registers are set up like this.

![](/images/regs.png)

As I was saying, a single byte is stored in `al`. The byte comes from `[edx + ebx]`. Our string is at `ebx`, and `edx` was initialised to zero earlier. Could it be the loop counter? Look down to the jump arrow that goes back up. `edx` in incremented and compared. Yup, that's our loop counter.

```sourceCode
|       .-> 0x08048d57      8a041a         mov al, byte [edx + ebx]
|       |   0x08048d5a      240f           and al, 0xf
|       |   0x08048d5c      0fbec0         movsx eax, al
|       |   0x08048d5f      8a0430         mov al, byte [eax + esi]
|       |   0x08048d62      88040a         mov byte [edx + ecx], al
|       |   0x08048d65      42             inc edx
|       |   0x08048d66      83fa05         cmp edx, 5
|       `=< 0x08048d69      7eec           jle 0x8048d57
```

That's the whole loop for easier reference. Now where were we? Right, single byte, loop counter, our string. Look familiar? We're copying the letter at offset `loop_counter` from our string into `al`. This letter is then _anded_ with `0xf`. What does that do? Well, let's take a look with _rax2_. ASCII characters start at _0_ and end at _127_ decimal. So let's pick the letter _a_ which is `0x61` and _f_ which is `0x66`

```sourceCode
$ rax2 "0x61&0xf"
1
$ rax2 "0x66&0xf"
6
```

So it looks like we get the last digit, looking at the binary digits will confirm this:

```sourceCode
0x66      01100110
0xf       00001111
```

So indeed we just get the lower four bits of the character. This is then stored in `eax` which is used in the next op. Line 4 is another index operation, specifically into the weird string at `esi` (_isrveawhobpnutfg_). Now we see why a) this string is longer than 6 characters, and b) the purpose of the _and_ operation. The _and_ operation ensures that the value is between 0 and 16, giving a nice index into that string. The letter at position `eax` is then stored back in `al` and then put into `local_8h[loop]`. We can now see that the loop will take the right most nybble, use that as an index into the _key_ string, and replace the letter in our string with the letter from the _key string_.

To make it a bit clearer, here's some C code

```c
char *key = "isrveawhobpnutfg";
char *src = "ourstr";

if (string_length(src) != 6) {
    explode_bomb();
}

char output[8];
unsigned int i = 0;
while (i <= 5) {
    int x = src[i] & 0xf;
    char a = key[x];
    output[i] = a;
    i ++;
}
```

Once we've processed all the characters the loop exits. Now is this a _while_ loop or a _for_ loop? Hard to say from what we can see. It could be either. Can it really? Well, allow me to demonstrate :) Let's write a _while_ loop and a _for_ loop, assemble it, and see what we get. I'm using `gcc (Debian 4.9.2-10) 4.9.2` to assemble the source with `gcc -S`

```c
int main() {
    int i = 0;
    int x = 0;
    for (; i <= 5; i++) {
        x++;
    }
    return x;
}
```

and its output

```sourceCode
.file   "for.c"
.text
.globl  main
.type   main, @function
    .text
    .globl  main
    .type   main, @function
main:
.LFB0:
    .cfi_startproc
    pushq   %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp
    .cfi_def_cfa_register 6
    movl    $0, -4(%rbp)
    movl    $0, -8(%rbp)
    jmp     .L2
.L3:
    addl    $1, -8(%rbp)
    addl    $1, -4(%rbp)
.L2:
    cmpl    $5, -4(%rbp)
    jle     .L3
    movl    -8(%rbp), %eax
    popq    %rbp
    .cfi_def_cfa 7, 8
    ret
    .cfi_endproc
.LFE0:
    .size   main, .-main
    .ident  "GCC: (Debian 4.9.2-10) 4.9.2"
    .section        .note.GNU-stack,"",@progbits
```

Now the while loop

```c
int main() {
    int i = 0;
    int x = 0;
    while (i <=5 ) {
        x++;
        i++;
    }
    return x;
}
```

and its output

```sourceCode
    .file   "while.c"
    .text
    .globl  main
    .type   main, @function
main:
.LFB0:
    .cfi_startproc
    pushq   %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp
    .cfi_def_cfa_register 6
    movl    $0, -4(%rbp)
    movl    $0, -8(%rbp)
    jmp     .L2
.L3:
    addl    $1, -8(%rbp)
    addl    $1, -4(%rbp)
.L2:
    cmpl    $5, -4(%rbp)
    jle     .L3
    movl    -8(%rbp), %eax
    popq    %rbp
    .cfi_def_cfa 7, 8
    ret
    .cfi_endproc
.LFE0:
    .size   main, .-main
    .ident  "GCC: (Debian 4.9.2-10) 4.9.2"
    .section        .note.GNU-stack,"",@progbits
```

So you see, the same output. If you can't be bothered to check line by line, here's the diff

```sourceCode
$ diff while.s for.s
1c1
<   .file   "while.c"
---
>   .file   "for.c"
```

Moving on we're now at the loop's exit condition and begin executing this code

```sourceCode
|           0x08048d6b      c645fe00       mov byte [ebp - local_2h], 0
|           0x08048d6f      83c4f8         add esp, -8
|           0x08048d72      680b980408     push str.giants ; str.giants ; "giants" @ 0x804980b
|           0x08048d77      8d45f8         lea eax, [ebp - local_8h]
|           0x08048d7a      50             push eax
|           0x08048d7b      e8b0020000     call sym.strings_not_equal
|           0x08048d80      83c410         add esp, 0x10
|           0x08048d83      85c0           test eax, eax
|       ,=< 0x08048d85      7405           je 0x8048d8c
|       |   0x08048d87      e870070000     call sym.explode_bomb
|       `-> 0x08048d8c      8d65e8         lea esp, [ebp - local_18h]
|           0x08048d8f      5b             pop ebx
|           0x08048d90      5e             pop esi
|           0x08048d91      89ec           mov esp, ebp
|           0x08048d93      5d             pop ebp
\           0x08048d94      c3             ret
```

Line 1 stores _0_ at a local variable. This never gets used, so I'm not sure what the point is, but it's likely that `ebp - local_2h` points to the end of the new string, and thus terminates it with a NULL. On line 3 another string is pushed onto the stack: _giants_. Then our new string is stored in `eax`, pushed onto the stack, and together with _giants_, forms the two arguments to _sym.strings_not_equal_. Once the call returns `eax` is tested and if it's equal, we jump over _sym.explode_bomb_ to the function epilogue.

## Mathing

So now we know enough to work out what input will work for us here.

```
i   |   s   |   r   |   v   |   e   |   a   |   w   |   h   |   o   |   b   |   p   |   n   |   u   |   t   |   f   |   g
0   |   1   |   2   |   3   |   4   |   5   |   6   |   7   |   8   |   9   |   10  |   11  |   12  |   13  |   14  |   15
```

The word we need is _giants_, therefore the letters we need are at positions _15, 0, 5, 11, 13, 1_, or _0xf, 0x0, 0x5, 0xb, 0xd, 0x1_. Head over to [asciitable](https://www.asciitable.com/) and see what characters will give us the values we need by checking the hex values of the letters.

For instance, this will do fine: `oPe[]a`

```sourceCode
$ ./bomb defuse
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
oPe[]a
Good work!  On to the next...
```

So add whatever string you settled on to the _defuse_ file, and wait for phase 6. But before we do, let's look at what happens when it runs.

## Seeing it happen

Keep and eye on eax as the loop runs. I've added some comments in the command prompt to help explain some things.

Roll tape.... (pop it up to 1080p if it defaults to 360potato)

<iframe width="853" height="480" src="https://www.youtube-nocookie.com/embed/5KwddWml2tM?vq=hd1080" frameborder="0" allowfullscreen></iframe>
So hopefully that helped explain a little about string handling and how the compiler can reduce different bits of code to the same assembly. Basically they do the same thing in this case, so the choice between a *while* and *for* loop is purely a matter of preference.

Thanks for reading and let me know if you have any comments, suggestions, or corrections.

Read [Phase 6](/2016/05/27/binary-bomb-with-radare2-phase-6)
