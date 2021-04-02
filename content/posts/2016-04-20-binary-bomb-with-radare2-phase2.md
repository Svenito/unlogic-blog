---
extends: default.liquid
title: Binary Bomb with Radare2 - Phase 2
date: 2016-04-20T19:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

Welcome once again to _defusing a binary bomb with radare_. Now we've got [phase 1](https://unlogic.co.uk/2016/04/15/binary-bomb-with-radare-phase-1) behind us, let's progress to phase 2. We're going to see loop constructs and array indexing in this phase. Such excite.

This post took a little longer to put together than the last two for a few reasons. One I'm still trying to work out the best way to present the info and record screen sessions. This time I tried to record the screen on OSX while running the debug session on a VM. So that all took a little time to get running well.

And as always, use radare from git ;)

## Phase 2

First we'll take a look at the disassembly

```
[0x080488e0]> pdf @ sym.phase_2
/ (fcn) sym.phase_2 79
|           ; arg int arg_1h @ ebp+0x1
|           ; arg int arg_8h @ ebp+0x8
|           ; var int local_18h @ ebp-0x18
|           ; var int local_28h @ ebp-0x28
|           ; CALL XREF from 0x08048a7e (sym.phase_2)
|           0x08048b48      55             push ebp
|           0x08048b49      89e5           mov ebp, esp
|           0x08048b4b      83ec20         sub esp, 0x20
|           0x08048b4e      56             push esi
|           0x08048b4f      53             push ebx
|           0x08048b50      8b5508         mov edx, dword [ebp+arg_8h] ; [0x8:4]=0
|           0x08048b53      83c4f8         add esp, -8
|           0x08048b56      8d45e8         lea eax, [ebp - local_18h]
|           0x08048b59      50             push eax
|           0x08048b5a      52             push edx
|           0x08048b5b      e878040000     call sym.read_six_numbers
|           0x08048b60      83c410         add esp, 0x10
|           0x08048b63      837de801       cmp dword [ebp - local_18h], 1 ; [0x1:4]=0x1464c45
|       ,=< 0x08048b67      7405           je 0x8048b6e
|       |   0x08048b69      e88e090000     call sym.explode_bomb
|       `-> 0x08048b6e      bb01000000     mov ebx, 1
|           0x08048b73      8d75e8         lea esi, [ebp - local_18h]
|       .-> 0x08048b76      8d4301         lea eax, [ebx + 1]          ; 0x1
|       |   0x08048b79      0faf449efc     imul eax, dword [esi + ebx*4 - 4]
|       |   0x08048b7e      39049e         cmp dword [esi + ebx*4], eax ; [0x13:4]=256
|      ,==< 0x08048b81      7405           je 0x8048b88
|      ||   0x08048b83      e874090000     call sym.explode_bomb
|      `--> 0x08048b88      43             inc ebx
|       |   0x08048b89      83fb05         cmp ebx, 5
|       `=< 0x08048b8c      7ee8           jle 0x8048b76
|           0x08048b8e      8d65d8         lea esp, [ebp - local_28h]
|           0x08048b91      5b             pop ebx
|           0x08048b92      5e             pop esi
|           0x08048b93      89ec           mov esp, ebp
|           0x08048b95      5d             pop ebp
\           0x08048b96      c3             ret
```

A bit longer than _phase_1_, but hopefully nothing too complex. I've numbered the lines so that function start is at line 1, in case you are wondering about the negative line numbers.

Before we look at the instructions, take a look at the jump arrows in the left column. There's two conditional jumps over the _sym.explode_bomb_, which we need to make sure are taken. Then there's one that goes from a high address to lower address. This usually indicates a loop of some sort, so note that down.

The loop can clearly be seen in the graph view

<img src="/images/r2-phase2-tree.png" alt="phase2 graph" width="400" />

_hint_: it's the green loopy looking thing

On line 11 there's a function call to _sym.read_six_numbers_, so there's a good chance that the loop will take our input string and parse the numbers out, most likely into six integers variables. So let's crack on and figure out what six numbers we need to enter.

### Doing it statically

The interesting parts start at line 6 where our input is moved into `edx`. That is, the address of where our six numbers are stored is moved into `edx`. We know that it's our argument because it sits at a _higher_ address than `ebp`, (`[ebp+arg_8]`) meaning it was pushed into the stack before the function call. Were it a local variable, it would read more like line 8, where `0x18` is _subtracted_ from `ebp`.

Speaking of which, that is where the address of the local variable is moved into `eax`. The values in these registers are now pushed onto the stack, and _read_six_numbers_ is called. Thus our input, and the local variable are the arguments passed to _sym.read_six_numbers_. So `eax` will most likely by used as a pointer to the numbers that are read by that function, thus containing the return values. This function uses `sscanf` to extract the six numbers from the input string as _ints_. It also checks to make sure there are six numbers, else **boom**. I'll cover this function along with the other helper functions later in the series.

So on line 13 our first number (now in _\[ebp - local_18h\]_) is compared to the immediate value _1_. If that checks out we jump over _sym.explode_bomb_ to safety.

Now _1_ is moved into `ebx`. This marks the start of the loop and `ebx` is our loop counter. Following this, the first number is loaded into `esi` and the loop counter plus 1 is loaded into `eax`. Now comes a tricky little line. Here it calculates what the next expected value should be.

```sourceCode
0x08048b79      0faf449efc     imul eax, dword [esi + ebx*4 - 4]
```

This might seem a little daunting at first, but all this is is an index into an array. `esi` is pointing to the address at the start of our number array. `ebx` is the loop counter. So `[esi + ebx*4]` is like saying `numbers[loop]`. So why the _\*4_? It's pointer arithmetic so to speak. Because we are dealing with an array of ints, we need to add 4 bytes to the address to get the next int, because ints are 4 bytes. Now, the _- 4_ just gives us `numbers[loop - 1]`. Because we need to go 4 bytes back from the loop to get to _loop - 1_. So we multiply this by _loop + 1_, which is the the value in `eax`.

This value is then compared to `[esi + ebx*4]`, which is, as we now know, an index into our number array, in this case `numbers[loop]`. So in plain english, each number needs to be equal to its position + 1 multiplied by its preceeding number.

If that checks out, the loop counter is incremented and if it's less than 5, we return to the start of the loop. Otherwise the loop terminates and we return out of the function.

Let's repeat that bit of disassembly here and translate it to the equivalent in C, which should be easier to understand

```sourceCode
|           0x08048b6e      bb01000000     mov ebx, 1
|           0x08048b73      8d75e8         lea esi, [ebp - local_18h]
|       .-> 0x08048b76      8d4301         lea eax, [ebx + 1]          ; 0x1
|       |   0x08048b79      0faf449efc     imul eax, dword [esi + ebx*4 - 4]
|       |   0x08048b7e      39049e         cmp dword [esi + ebx*4], eax ; [0x13:4]=256
|      ,==< 0x08048b81      7405           je 0x8048b88
|      ||   0x08048b83      e874090000     call sym.explode_bomb
|      `--> 0x08048b88      43             inc ebx
|       |   0x08048b89      83fb05         cmp ebx, 5
|       `=< 0x08048b8c      7ee8           jle 0x8048b76
```

Which is (roughly) equivalent to:

```c
numbers = []
for i=1; i <= 5; i++ {
    int t = (i + 1) * nums[i-1];
    if (t != nums[i]) {
        explode_bomb();
    }
}
```

If all numbers are ok, the exit condtion of the loop is met and the loop ends. It then cleans up the stack and returns to _sym.main_

So doing the math, we can deduce that the defuse code should be `1 2 6 24 120 720`. Well, let's try that out while running it through Radare's debugger.

### Doing it dynamically

In this debugging session we'll make a mistake on purpose to make use of some of Radare's features we haven't used yet. We'll make a typo on our input and then fix it during the debug session. There's a number of ways we could cheat to defuse the bomb, but for this example I'm going to opt for editing the values in memory.

#### Fire her up

To start an interactive debugging session with Radare, simply add the `-d` flag. We'll pass the exisiting list of solutions (which only contains phase 1 so far) as the argument. I've named my file _defuse_. This means we don't have to re-enter the known solutions for previous phases. Once launched we need to analyse everything again, and then set a breakpoint on _sym.phase_2_. After that we'll check the breakpoint is set and begin execution.

Here's how that will look:

```sourceCode
[unlogic@debbie bin_bomb]$ r2 -d bomb defuse
Process with PID 17116 started...
attach 17116 17116
bin.baddr 0x08048000
Assuming filepath ./bomb
asm.bits 32
 -- radare2 contributes to the One Byte Per Child fundation.
[0xf7784000]> aa
[x] Analyze all flags starting with sym. and entry0 (aa)
[0xf7784000]> db sym.phase_2
[0xf7784000]> db
0x08048b48 - 0x08048b49 1 --x sw break enabled cmd="" name="sym.phase_2"
[0xf7784000]> dc
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
```

I'll enter `1 2 7 24 120 720` here, purposefully getting the third number wrong.

```sourceCode
[0xf7784000]> dc
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
1 2 7 24 120 720
hit breakpoint at: 8048b48
attach 17116 1
[0x08048b48]>
```

So enter visual mode by entering `V` and then press `p` twice to view the dissasembly. If you feel like it, press `!` to get to the pane view. Ooooh, lovely, isn't it?

Here's the code we're going to debug, so we have some context

```sourceCode
|           0x08048b48      55             push ebp
|           0x08048b49      89e5           mov ebp, esp
|           0x08048b4b      83ec20         sub esp, 0x20
|           0x08048b4e      56             push esi
|           0x08048b4f      53             push ebx
|           0x08048b50      8b5508         mov edx, dword [ebp+arg_8h] ; [0x8:4]=0
|           0x08048b53      83c4f8         add esp, -8
|           0x08048b56      8d45e8         lea eax, [ebp - local_18h]
|           0x08048b59      50             push eax
|           0x08048b5a      52             push edx
|           0x08048b5b      e878040000     call sym.read_six_numbers
|           0x08048b60      83c410         add esp, 0x10
|           0x08048b63      837de801       cmp dword [ebp - local_18h], 1 ; [0x1:4]=0x1464c45
|       ,=< 0x08048b67      7405           je 0x8048b6e
|       |   0x08048b69      e88e090000     call sym.explode_bomb
|       `-> 0x08048b6e      bb01000000     mov ebx, 1
|           0x08048b73      8d75e8         lea esi, [ebp - local_18h]
|       .-> 0x08048b76      8d4301         lea eax, [ebx + 1]          ; 0x1
|       |   0x08048b79      0faf449efc     imul eax, dword [esi + ebx*4 - 4]
|       |   0x08048b7e      39049e         cmp dword [esi + ebx*4], eax ; [0x13:4]=256
|      ,==< 0x08048b81      7405           je 0x8048b88
|      ||   0x08048b83      e874090000     call sym.explode_bomb
|      `--> 0x08048b88      43             inc ebx
|       |   0x08048b89      83fb05         cmp ebx, 5
|       `=< 0x08048b8c      7ee8           jle 0x8048b76
|           0x08048b8e      8d65d8         lea esp, [ebp - local_28h]
|           0x08048b91      5b             pop ebx
|           0x08048b92      5e             pop esi
|           0x08048b93      89ec           mov esp, ebp
|           0x08048b95      5d             pop ebp
```

Let's press `s` to start stepping through code. When you get to a function call, press `S` to step _over_ it, unless you are curious and want to take a peek inside. We're going to step to line 6 and take a look at `[ebp+arg_8h]`. We know the value of _arg_8h_ is `0x8`, from the information at the top of the disassembly. So let's print out what's at that memory location (press `:` to enter command mode)

```sourceCode
:> pr @ [ebp+0x8]
1 2 7 24 120 720
```

Sure enough, that's our input text. So moving onto line 8, let's check what's at `[ebp - local_18h]`

```sourceCode
:> pr @ [ebp-0x18]
GRADE_BOMBError: Input line too long
ERROR: dup(0) error
ERROR: close error
ERROR: tmpfile error
Subject: Bomb notification

nobodydefusedexplodedbomb-header:%s:%d:%s:%s:%d
bomb-string:%s:%d:%s:%d:%s
bom
:> ps @ [ebp-0x18]
GRADE_BOMB
```

Not sure what _GRADE_BOMB_ means, but if it's a local variable adress, pushed onto the stack, it's likely going to contain the result of the _sym.read_six_numbers_ call.

So our entry, and the local variable addresses are now on the stack and form the arguments for the _sym.read_six_numbers_ function. Stepping over this to line 12, gives us an opportunity to examine the state of play after this call.

If we look at the address stored in `ebp` and subtract 24 (0x18) we can see the numbers we entered.

```sourceCode
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffa683a0  d0b6 0408 c083 a6ff 209c 71f7 a484 a6ff  ........ .q.....
0xffa683b0  a484 a6ff 0000 0000 e883 a6ff 2692 0408  ............&...
0xffa683c0  0100 0000 0200 0000 0700 0000 1800 0000  ................
0xffa683d0  7800 0000 d002 0000 0884 a6ff 838a 0408  x...............
 eip 0x08048b60     oeax 0xffffffff      eax 0x00000006      ebx 0xffa684a4
 ecx 0x00000000      edx 0xffa683d4      esp 0xffa683a0      ebp 0xffa683d8
 esi 0x00000000      edi 0x00000000      eflags 1I
```

They start at `0xffa683c0` and are now ints instead of strings as they take up 4 bytes. We can check this by printing the values

```sourceCode
:> px 24 @ ebp - 0x18
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffa683c0  0100 0000 0200 0000 0700 0000 1800 0000  ................
0xffa683d0  7800 0000 d002 0000                      x.......
```

So far, so good. Our first number is _1_ and the initial check at line 13 will pass, and we enter the loop. Let's check that our assumptions about the `imul` operation were correct.

We said that it would effectively be _numbers\[loop - 1\]_, and right now our loop (ebx) is 1. So this should give us the first number

```sourceCode
:> dr ebx
0x00000001
:> px 4 @ esi + ebx*4 - 4
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffa683c0  0100 0000
```

The result should then be in `eax` and compared to _numbers\[loop\]_. Stepping past the `imul` operation, we'll examine what we've got.

```sourceCode
:> dr eax
0x00000002
:> px 4 @ esi + ebx*4
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffa683c4  0200 0000
```

Checks out. Now we'll jump over the _sym.explode_bomb_, increment our loop counter (`ebx`) and start over. This iteration however we need to stop at the _cmp_ call, because our value isn't correct.

```sourceCode
:> dr eax
0x00000006
:> px 4 @ esi + ebx*4
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffa683c8  0700 0000
```

We're going to change that value before stepping over. Using

> |Usage: w\[x\] \[str\] \[&lt;file\] \[&lt;&lt;EOF\] \[@addr\] w\[1248\]\[+-\]\[n\] increment/decrement byte,word.

We'll decrement the incorrect value to 6 using that command

```sourceCode
:> w4- 1 @ esi + ebx*4
:> px 4 @ esi + ebx*4
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0xffa683c8  0600 0000                                ....
```

Success. Stepping on we should jump over _sym.explode_bomb_ and go back to the top of the loop.

This will go on until the loop meets its exit condition and then we return from the function.

Now let's add those numbers to the end of our _defuse_ file, ready for the next phase.

For those less inclined to read (although you've read this far ;)), here's a video of the debugging session

<iframe width="853" height="480" src="https://www.youtube-nocookie.com/embed/DNor5zv3cNg?rel=0" frameborder="0" allowfullscreen></iframe>
Select *1080p* resolution to make it easier to view, or view it directly on [Youtube](https://www.youtube.com/watch?v=DNor5zv3cNg).

[Read Phase 3](https://unlogic.co.uk/2016/04/27/binary-bomb-with-radare2-phase-3)
