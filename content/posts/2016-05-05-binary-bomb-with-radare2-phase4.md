---
extends: default.liquid
title: Binary Bomb with Radare2 - Phase 4
date: 2016-05-05T19:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

With the switch statement of [phase 3](https://unlogic.co.uk/2016/04/27/binary-bomb-with-radare2-phase-3/) behind us, shall we cary on with phase 4? This time we're going to get to see recursion. Nothing too complicated though, it's still early on in the game.

## What we doing?

Let's take a look at the disass of _sym.phase_4_. I'll skip the whole loading, analysing, etc parts, as we should be familiar with how we do that now. Here's _sym.phase_4_

```bash
/ (fcn) sym.phase_4 75
|           ; arg int arg_8h @ ebp+0x8
|           ; var int local_0h @ ebp-0x0
|           ; var int local_4h @ ebp-0x4
|           ; CALL XREF from 0x08048ac4 (sym.main)
|           0x08048ce0      55             push ebp
|           0x08048ce1      89e5           mov ebp, esp
|           0x08048ce3      83ec18         sub esp, 0x18
|           0x08048ce6      8b5508         mov edx, dword [ebp + arg_8h] ; [0x8:4]=0
|           0x08048ce9      83c4fc         add esp, -4
|           0x08048cec      8d45fc         lea eax, [ebp - local_4h]
|           0x08048cef      50             push eax
|           0x08048cf0      6808980408     push 0x8049808              ; "%d" @ 0x8049808
|           0x08048cf5      52             push edx
|           0x08048cf6      e865fbffff     call sym.imp.sscanf
|           0x08048cfb      83c410         add esp, 0x10
|           0x08048cfe      83f801         cmp eax, 1
|       ,=< 0x08048d01      7506           jne 0x8048d09
|       |   0x08048d03      837dfc00       cmp dword [ebp - local_4h], 0
|      ,==< 0x08048d07      7f05           jg 0x8048d0e
|      |`-> 0x08048d09      e8ee070000     call sym.explode_bomb
|      `--> 0x08048d0e      83c4f4         add esp, -0xc
|           0x08048d11      8b45fc         mov eax, dword [ebp - local_4h]
|           0x08048d14      50             push eax
|           0x08048d15      e886ffffff     call sym.func4
|           0x08048d1a      83c410         add esp, 0x10
|           0x08048d1d      83f837         cmp eax, 0x37               ; '7'
|       ,=< 0x08048d20      7405           je 0x8048d27
|       |   0x08048d22      e8d5070000     call sym.explode_bomb
|       `-> 0x08048d27      89ec           mov esp, ebp
|           0x08048d29      5d             pop ebp
\           0x08048d2a      c3             ret
```

"But where's the recursion?" you ask. Well it's not here. Line 20 has a call to _sym.func4_. This is where the recursion happens. But before we take a look at that, let's examine what pre-requisites we need to satisfy in order to get to that point. There is one call to _sym.explode_bomb_ after two conditionals, so we'll need to know what conditions those are.

First up is some code we're already familiar with. A setup and call to _sym.imp.sscanf_ using `"%d"` as the formatter. This tells us we need to enter a single number. Then there's the check that it's successfully read a single number (line 12). If not, we `jne` to _sym.explode_bomb_. Following that check we have `cmp dword [ebp - local_4h], 0`. This checks the value we entered against _0_. The following conditional test jumps past _sym.explode_bomb_ if the number we entered is greater than 0. So in order to get to the call for _sym.func4_ all we need to do is enter a single number that's greater than zero. I think I can manage that.

Our number is then moved into `eax`, forming the single argument to _sym.func4_. So let's take a look at this function

```sourceCode
/ (fcn) sym.func4 62
|           ; arg int arg_8h @ ebp+0x8
|           ; var int local_18h @ ebp-0x18
|           ; CALL XREF from 0x08048cb7 (sym.func4)
|           ; CALL XREF from 0x08048cc5 (sym.func4)
|           ; CALL XREF from 0x08048d15 (sym.phase_4)
|           0x08048ca0      55             push ebp
|           0x08048ca1      89e5           mov ebp, esp
|           0x08048ca3      83ec10         sub esp, 0x10
|           0x08048ca6      56             push esi
|           0x08048ca7      53             push ebx
|           0x08048ca8      8b5d08         mov ebx, dword [ebp + arg_8h] ; [0x8:4]=0
|           0x08048cab      83fb01         cmp ebx, 1                  ; "ELF..." @ 0x1
|       ,=< 0x08048cae      7e20           jle 0x8048cd0               ;[2]
|       |   0x08048cb0      83c4f4         add esp, -0xc
|       |   0x08048cb3      8d43ff         lea eax, [ebx - 1]
|       |   0x08048cb6      50             push eax
|       |   0x08048cb7      e8e4ffffff     call sym.func4              ;[3]
|       |   0x08048cbc      89c6           mov esi, eax
|       |   0x08048cbe      83c4f4         add esp, -0xc
|       |   0x08048cc1      8d43fe         lea eax, [ebx - 2]
|       |   0x08048cc4      50             push eax
|       |   0x08048cc5      e8d6ffffff     call sym.func4              ;[3]
|       |   0x08048cca      01f0           add eax, esi
|      ,==< 0x08048ccc      eb07           jmp 0x8048cd5               ;[4]
|      ||   0x08048cce      89f6           mov esi, esi
|      ||   ; JMP XREF from 0x08048cae (sym.func4)
|      |`-> 0x08048cd0      b801000000     mov eax, 1
|      |    ; JMP XREF from 0x08048ccc (sym.func4)
|      `--> 0x08048cd5      8d65e8         lea esp, [ebp - local_18h]
|           0x08048cd8      5b             pop ebx
|           0x08048cd9      5e             pop esi
|           0x08048cda      89ec           mov esp, ebp
|           0x08048cdc      5d             pop ebp
\           0x08048cdd      c3             ret
            0x08048cde      89f6           mov esi, esi
```

First thing to note is that our input is at `[ebp+0x8]`. Skipping past the function prologue we see that our input is moved into `ebx`, which is subsequently compared to the immediate _1_. If our value is less or equal to _1_ we jump to `0x08048cd0` (line 28). Here the value _1_ is copied to `eax`, the stack restored and the function returns. `eax` holds our return value.

So let's see what happens if we don't jump: Our value minus one is loaded into `eax` and _sym.func4_ is called. So our _value-1_ is the argument to the recursive call. Once we return from that call, the return value (in `eax`) is copied to `esi`, our original value is now decremented by _2_ (line 21), and passed to another call to _sym.func4_. The result of this is then added to `esi`, which contains the result of the previous call to _sym.func4_, and we jump to the exit portion of the function.

To help understand these recursive calls, I'm going to make use of [rcviz](https://github.com/carlsborg/rcviz) and replicate this function's behaviour in a python script

```sourceCode
from rcviz import callgraph, viz

@viz
def func(x):
    if (x <= 1):
        return 1

    t = x - 1
    z = func(t)
    t = x - 2
    z += func(t)
    return z

import sys
print func(int(sys.argv[1]))
callgraph.render("{}.png".format(sys.argv[1]))
```

Here's the graphs for the values 5, 6, and 7

![](/images/5.png)

![](/images/6.png)

![](/images/7.png)

If we count the number of leaves (nodes without children, or those that return _1_), we get the value that the function returns. What's also worth noting is that the call with _7_ contains the graphs for _5_ and _6_. _5_ results in a value of _8_ and an argument of _6_ gives us _13_. Adding the results together will give us the value for _7_, which is _21_.

Ok, so now we know what roughly happens in that function, let's head back to our caller and see what he wants from us.

```sourceCode
|           0x08048d15      e886ffffff     call sym.func4              ;[5]
|           0x08048d1a      83c410         add esp, 0x10
|           0x08048d1d      83f837         cmp eax, 0x37               ; '7'
|       ,=< 0x08048d20      7405           je 0x8048d27                ;[6]
|       |   0x08048d22      e8d5070000     call sym.explode_bomb       ;[4]
|       `-> 0x08048d27      89ec           mov esp, ebp
|           0x08048d29      5d             pop ebp
\           0x08048d2a      c3             ret
```

The return value will be in `eax` and we can see that it will get compared to `0x37` which is:

```sourceCode
$ rax2 0x37
55
```

So we need to make _sym.func4_ return _55_. Well the easiest way is just to simply call our python script with a few values. We can see that the result increases quite rapidly, so it shouldn't take long to get our value.

```sourceCode
$ for i in {3..10};do echo -n $i: ; python t.py $i; done
3:3
4:5
5:8
6:13
7:21
8:34
9:55
10:89
```

Well, there we have it, the value we want is _9_. Plug that into phase 4, add it to your _defuse_ file, and we can move onto phase 5.

[Read phase 5](/2016/05/12/binary-bomb-with-radare2-phase-5/)
