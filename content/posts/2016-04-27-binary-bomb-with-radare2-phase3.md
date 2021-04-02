---
title: Binary Bomb with Radare2 - Phase 3
date: 2016-04-27T19:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

Wasn't [phase 2](https://unlogic.co.uk/2016/04/20/binary-bomb-with-radare2-phase-2/) fun? Well brace yourself for phase 3. This time we're going to see what a switch statement looks like in (dis)assembly. _squeeee_

So let's update our radare from git, install it, and get started.

## Phase 3

Here's what we're going to be analysing.

````````sourceCode
/ (fcn) sym.phase_3 264
|           ; arg int arg_7h @ ebp+0x7
|           ; arg int arg_8h @ ebp+0x8
|           ; var int local_4h @ ebp-0x4
|           ; var int local_5h @ ebp-0x5
|           ; var int local_ch @ ebp-0xc
|           ; var int local_18h @ ebp-0x18
|           ; CALL XREF from 0x08048aa1 (sym.phase_3)
|           0x08048b98      55             push ebp
|           0x08048b99      89e5           mov ebp, esp
|           0x08048b9b      83ec14         sub esp, 0x14
|           0x08048b9e      53             push ebx
|           0x08048b9f      8b5508         mov edx, dword [ebp+arg_8h] ; [0x8:4]=0
|           0x08048ba2      83c4f4         add esp, -0xc
|           0x08048ba5      8d45fc         lea eax, [ebp - local_4h]
|           0x08048ba8      50             push eax
|           0x08048ba9      8d45fb         lea eax, [ebp - local_5h]
|           0x08048bac      50             push eax
|           0x08048bad      8d45f4         lea eax, [ebp - local_ch]
|           0x08048bb0      50             push eax
|           0x08048bb1      68de970408     push str._d__c__d ; str._d__c__d ; "%d %c %d" @ 0x80497de
|           0x08048bb6      52             push edx
|           0x08048bb7      e8a4fcffff     call sym.imp.sscanf         ;[1]
|           0x08048bbc      83c420         add esp, 0x20
|           0x08048bbf      83f802         cmp eax, 2                  ; "LF..." @ 0x2
|       ,=< 0x08048bc2      7f05           jg 0x8048bc9                ;[2]
|       |   0x08048bc4      e833090000     call sym.explode_bomb       ;[3]
|       `-> 0x08048bc9      837df407       cmp dword [ebp - local_ch], 7 ; [0x7:4]=0
|       ,=< 0x08048bcd      0f87b5000000   ja 0x8048c88                ;[4]
|       |   0x08048bd3      8b45f4         mov eax, dword [ebp - local_ch]
|       |   0x08048bd6      ff2485e89704.  jmp dword [eax*4 + 0x80497e8]
|       |   0x08048bdd      8d7600         lea esi, [esi]
|       |   0x08048be0      b371           mov bl, 0x71                ; 'q'
|       |   0x08048be2      817dfc090300.  cmp dword [ebp - local_4h], 0x309 ; [0x309:4]=0x21000000
|      ,==< 0x08048be9      0f84a0000000   je 0x8048c8f                ;[5]
|      ||   0x08048bef      e808090000     call sym.explode_bomb       ;[3]
|     ,===< 0x08048bf4      e996000000     jmp 0x8048c8f               ;[5]
|       |   0x08048bf9      8db426000000.  lea esi, [esi]
|       |   0x08048c00      b362           mov bl, 0x62                ; 'b'
|       |   0x08048c02      817dfcd60000.  cmp dword [ebp - local_4h], 0xd6 ; [0xd6:4]=0x1080000
|      ,==< 0x08048c09      0f8480000000   je 0x8048c8f                ;[2]
|      ||   0x08048c0f      e8e8080000     call sym.explode_bomb       ;[1]
|     ,===< 0x08048c14      eb79           jmp 0x8048c8f               ;[2]
|     |||   0x08048c16      b362           mov bl, 0x62                ; 'b'
|     |||   0x08048c18      817dfcf30200.  cmp dword [ebp - local_4h], 0x2f3 ; [0x2f3:4]=0x45208
|    ,====< 0x08048c1f      746e           je 0x8048c8f                ;[2]
|    ||||   0x08048c21      e8d6080000     call sym.explode_bomb       ;[1]
|   ,=====< 0x08048c26      eb67           jmp 0x8048c8f               ;[2]
|   |||||   0x08048c28      b36b           mov bl, 0x6b                ; 'k'
|   |||||   0x08048c2a      817dfcfb0000.  cmp dword [ebp - local_4h], 0xfb ; [0xfb:4]=0x6e696c2d ; "-li
|  ,======< 0x08048c31      745c           je 0x8048c8f                ;[2]
|  ||||||   0x08048c33      e8c4080000     call sym.explode_bomb       ;[1]
| ,=======< 0x08048c38      eb55           jmp 0x8048c8f               ;[2]
| |||||||   0x08048c3a      8db600000000   lea esi, [esi]
| |||||||   0x08048c40      b36f           mov bl, 0x6f                ; 'o'
| |||||||   0x08048c42      817dfca00000.  cmp dword [ebp - local_4h], 0xa0 ; [0xa0:4]=0x804ade0 loc.__d
| ========< 0x08048c49      7444           je 0x8048c8f                ;[2]
| |||||||   0x08048c4b      e8ac080000     call sym.explode_bomb       ;[1]
| ========< 0x08048c50      eb3d           jmp 0x8048c8f               ;[2]
| |||||||   0x08048c52      b374           mov bl, 0x74                ; 't'
| |||||||   0x08048c54      817dfcca0100.  cmp dword [ebp - local_4h], 0x1ca ; [0x1ca:4]=0x100000
| ========< 0x08048c5b      7432           je 0x8048c8f                ;[2]
| |||||||   0x08048c5d      e89a080000     call sym.explode_bomb       ;[1]
| ========< 0x08048c62      eb2b           jmp 0x8048c8f               ;[2]
| |||||||   0x08048c64      b376           mov bl, 0x76                ; 'v'
| |||||||   0x08048c66      817dfc0c0300.  cmp dword [ebp - local_4h], 0x30c ; [0x30c:4]=33 ; "!" @ 0x30c
| ========< 0x08048c6d      7420           je 0x8048c8f                ;[2]
| |||||||   0x08048c6f      e888080000     call sym.explode_bomb       ;[1]
| ========< 0x08048c74      eb19           jmp 0x8048c8f               ;[2]
| |||||||   0x08048c76      b362           mov bl, 0x62                ; 'b'
| |||||||   0x08048c78      817dfc0c0200.  cmp dword [ebp - local_4h], 0x20c ; [0x20c:4]=287
| ========< 0x08048c7f      740e           je 0x8048c8f                ;[2]
| |||||||   0x08048c81      e876080000     call sym.explode_bomb       ;[1]
| ========< 0x08048c86      eb07           jmp 0x8048c8f               ;[2]
| |||||||   0x08048c88      b378           mov bl, 0x78                ; 'x'
| |||||||   0x08048c8a      e86d080000     call sym.explode_bomb       ;[2]
| ```````-> 0x08048c8f      3a5dfb         cmp bl, byte [ebp - local_5h]
|       ,=< 0x08048c92      7405           je 0x8048c99                ;[3]
|       |   0x08048c94      e863080000     call sym.explode_bomb       ;[2]
|       `-> 0x08048c99      8b5de8         mov ebx, dword [ebp - local_18h]
|           0x08048c9c      89ec           mov esp, ebp
|           0x08048c9e      5d             pop ebp
\           0x08048c9f      c3             ret
````````

Quite the doozy of a function with some serious branching. A bit worried about how bonkers this looks? Take a good look at it, because that's the switch statement I mentioned earlier. It's actually a lot less daunting than it looks, so take a sip of your favourite drink and let's dive in.

### Parsing the input

After the function prologue the address of our input string gets moved into `edx` (line 5). Then three local variables get pushed onto the stack (lines 7-12). Following this the string `%d %c %d` gets pushed onto the stack, and then our input string, (as its address stored in `edx`) gets pushed onto the stack.

These lines set up the function arguments for the `sscanf` call. Our string being the source, the format string (`%d %c %d`) and the three variables that will contain the scanned out values. It's now apparent that the required input is in the form of a number, a character, and another number. We can extract that information from the format string passed into `sscanf`. Good to know what we need to pas in, because if we don't get that right we'll explode the bomb.

Once we return from `sscanf`, `eax` gets compared to the immediate value _2_. After `sscanf` returns, `eax` contains its return value, which is, according to its manpage

> the number of conversions which were successfully completed is returned.

If the value in `eax` is greater than _2_, then we jump over _sym.explode_bomb_. Therefore this test is to ensure that three values have been scanned out successfully.

### Switching things

Now the switch statement begins (line 20). Because `ebp - local_ch` was the last variable to be pushed onto the stack before the call to _sscanf_, it will contain the first number. As arguments are pushed onto the stack in left to right order. The value stored there is compared to the immediate value of _7_. On the following line (line 21) is a conditional jump we haven't seen before. To the docs!

> JA : Jump if above (if leftOp &gt; rightOp)

To expand on this, `ja` is part of the jumps based on unsigned comparisons. Ok, so we take the jump if our first value is greater (or equal) _7_. What with the jump being a check for _unsigned_ integers, we also know it can't be less than _0_. Let's peek at `0x8048c88`, which is where the jump would take us. Use `:s 0x8048c88` to seek there.

```sourceCode
|           0x08048c88      b378           mov bl, 0x78                ; 'x'
|           0x08048c8a      e86d080000     call sym.explode_bomb       ;[1]
```

No, that doesn't like somewhere we want to end up. Take a note: the first number has to be less than or equal to seven. It would seem that this is the default case statement.

Let's go back and see what happens if we don't take the jump. The value is moved into `eax` and then we have a jump based on that value. This is the start of the switch statement. At line 23 the code `jmp dword [eax*4 + 0x80497e8]` will jump to the relevant case statement, based on the address stored at `[eax*4 + 0x80497e8]`. Let's take a look at what's stored at `0x80497e8`

```sourceCode
:> px 35 @ 0x80497e8
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x080497e8  e08b 0408 008c 0408 168c 0408 288c 0408  ............(...
0x080497f8  408c 0408 528c 0408 648c 0408 768c 0408  @...R...d...v...
0x08049808  2564 00
```

We know that there's a check to make sure our input not greater than seven, so I figured I might just check 35 bytes (4 \* 7 plus 4 extra to see what's there). What we have here is a lookup table (or more correctly named, a _jump table_) which contains the addresses of the _case_ statements, in little endian format of course.

> **note**
>
> You will only see this kind of jump table if the case statements are based on sequential values. That's because the jump values needed to get into the jump table will only work in such a situation. Other switch statements you encouter will work differently. Here it's that the compiler can optimises sequential values to produce what we have here.

So the first entry is `0x08048be0`, which is the address of the first case statement

```sourceCode
|       |   0x08048be0      b371           mov bl, 0x71                ; 'q'
|       |   0x08048be2      817dfc090300.  cmp dword [ebp - local_4h], 0x309 ; [0x309:4]=0x21000000
|      ,==< 0x08048be9      0f84a0000000   je 0x8048c8f                ;[5]
|      ||   0x08048bef      e808090000     call sym.explode_bomb       ;[3]
|     ,===< 0x08048bf4      e996000000     jmp 0x8048c8f
```

By the time we get to the end of the lookup table you will notice that there's no valid address, hence the condition that our value is less than or equal to 7.

Now we know that our first input will direct us to a specific case statement, but how does that affect our other entries? Let's just look at the example above, where the first number we entered is _0_.

On line 1 the character _q_ is moved into `bl`. This is an 8 bit register, which forms the low byte of the `bpx` register. After that our second digit is compared with `0x309` which is 777 in decimal. If the numbers are equal the conditional jump takes us to `0x8049c8f`. And because we don't call _sym.explode_bomb_, we're doing good. Let's take a look at what happens at the address of the jump

```sourceCode
|           0x08048c8f      3a5dfb         cmp bl, byte [ebp - local_5h]
|       ,=< 0x08048c92      7405           je 0x8048c99                ;[1]
|       |   0x08048c94      e863080000     call sym.explode_bomb       ;[2]
|       `-> 0x08048c99      8b5de8         mov ebx, dword [ebp - local_18h]
|           0x08048c9c      89ec           mov esp, ebp
|           0x08048c9e      5d             pop ebp
\           0x08048c9f      c3             ret
```

Another `cmp`, this time comparing our input character to what's in `bl`. If they are equal, we jump over _sym.explode_bomb_ once again and start to exit the function to return back to _sym.main_.

## What do we know?

So we know we need to enter a number, followed by a character, followed by another number. The first number needs to be less than or equal to seven. The choice of number will determine what letter and number follows. Looking at the switch statement, we can work out the following options

```sourceCode
0 q 777
1 b 214
2 b 755
3 k 251
4 o 160
5 t 458
6 v 780
7 b 524
```

Pick one and try it out. If it works (and it should), add it to the _defuse_ file, and get ready for phase 4.

Because we've pretty much figured out the whole phase from static analysis, I don't think there's a need to cover it with dynamic analysis here.

If you spot any errors, have some feedback, or questions about any of the phases, let me know, either via the Disqus comments below, or via [Twitter](https://twitter.com/binaryheadache).

[Read Phase 4](https://unlogic.co.uk/2016/05/05/binary-bomb-with-radare2-phase-4)
