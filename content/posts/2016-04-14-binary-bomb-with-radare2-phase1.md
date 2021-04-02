---
extends: default.liquid
title: Binary Bomb with Radare2 - Phase 1
date: 2016-04-14T19:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

Welcome back fellow reverser, I hope you enjoyed the [prelude](https://unlogic.co.uk/2016/04/12/binary-bomb-with-radare2-prelude/) and are ready to move onto _phase 1_. But just before we do that let's take a look at that last chunk of code in main: the bit after `call sym.initialize_bomb`

## Last part of sym.main

We saw that if everything went ok in the first part of main, we jump to _sym.initialize_bomb_. Now we'll take a look what happens after that call. I should mention that unless absolutely neccessary I won't be diving into auxillary functions like _initialize_bomb_ at this stage. I will just outline their purpose and then cover them in more detail at the end of the series.

Here's the truncated disassembly

```sourceCode
0x08048a30      e82b070000     call sym.initialize_bomb    ; bomb.c:66
0x08048a35      83c4f4         add esp, -0xc               ; bomb.c:68
0x08048a38      6860960408     push str.Welcome_to_my_fiendish_little_bomb._You_have_6_phases_with_n ; str.Welcome_to_my_fiendish_little_bomb._You_have_6_phases_with_n ; "Welcome to my fiendish little bomb. You have 6 phases with." @ 0x8049660
0x08048a3d      e8cefdffff     call sym.imp.printf
0x08048a42      83c4f4         add esp, -0xc               ; bomb.c:69
0x08048a45      68a0960408     push str.which_to_blow_yourself_up._Have_a_nice_day__n ; str.which_to_blow_yourself_up._Have_a_nice_day__n ; "which to blow yourself up. Have a nice day!." @ 0x80496a0
0x08048a4a      e8c1fdffff     call sym.imp.printf
0x08048a4f      83c420         add esp, 0x20               ; bomb.c:72
0x08048a52      e8a5070000     call sym.read_line
0x08048a57      83c4f4         add esp, -0xc               ; bomb.c:73
0x08048a5a      50             push eax
0x08048a5b      e8c0000000     call sym.phase_1
0x08048a60      e8c70a0000     call sym.phase_defused      ; bomb.c:74
0x08048a65      83c4f4         add esp, -0xc               ; bomb.c:76
0x08048a68      68e0960408     push str.Phase_1_defused._How_about_the_next_one__n ; str.Phase_1_defused._How_about_the_next_one__n ; "Phase 1 defused. How about the next one?." @ 0x80496e0
0x08048a6d      e89efdffff     call sym.imp.printf
0x08048a72      83c420         add esp, 0x20               ; bomb.c:80
0x08048a75      e882070000     call sym.read_line
0x08048a7a      83c4f4         add esp, -0xc               ; bomb.c:81
0x08048a7d      50             push eax
0x08048a7e      e8c5000000     call sym.phase_2
0x08048a83      e8a40a0000     call sym.phase_defused      ; bomb.c:82v
.. snip ..
```

Lines 2-7 print out a welcome message to the terminal. One of which says there are 6 phases. Remember in the prelude I said there are seven phases? If you looked at the list of functions you might have noticed the secret phase. I'll save that until the end though. Anyways, back to analysing the code. At line 9 we have a call to _read_line_ which is an auxillary function of the bomb. What this function does is read a line from _stdin_ or the file handle from the file we supplied earlier. That's enough to know about that function for now, we'll cover this function towards the end of the series. It's at this point where we supply what we think will defuse the bomb.

The line is then pushed onto the stack with `push eax` (line 11) followed by calling _sym.phase_1_. If everything goes well inside _phase_1_ we'll return back to here and then call _sym.phase_defused_. Otherwise we'll get a **kaboom**. This pattern is then repeated for each of the subsequent phases.

Ok phase 1, whatchya got for us?

## Phase 1

In order to examine the code we can either seek to it with `s sym.phase_1` and view it in visual mode with `V`, or dump it with `pdf @ sym.phase_1`. Because the function is relatively short, I'll use `pdf` this time.

```sourceCode
            ;-- gcc2_compiled._5:
/ (fcn) sym.phase_1 39
|           ; arg int arg_8h @ ebp+0x8
|           ; CALL XREF from 0x08048a5b (sym.phase_1)
|           0x08048b20      55             push ebp
|           0x08048b21      89e5           mov ebp, esp
|           0x08048b23      83ec08         sub esp, 8
|           0x08048b26      8b4508         mov eax, dword [ebp+arg_8h] ; [0x8:4]=0
|           0x08048b29      83c4f8         add esp, -8
|           0x08048b2c      68c0970408     push str.Public_speaking_is_very_easy. ; str.Public_speaking_is_very_easy. ; "Public speaking is very easy." @ 0x80497c0
|           0x08048b31      50             push eax
|           0x08048b32      e8f9040000     call sym.strings_not_equal  ;[1]
|           0x08048b37      83c410         add esp, 0x10
|           0x08048b3a      85c0           test eax, eax
|       ,=< 0x08048b3c      7405           je 0x8048b43                ;[2]
|       |   0x08048b3e      e8b9090000     call sym.explode_bomb       ;[3]
|       `-> 0x08048b43      89ec           mov esp, ebp
|           0x08048b45      5d             pop ebp
\           0x08048b46      c3             ret
            0x08048b47      90             nop
```

Isn't that a cute little function? Lines 1 to 3 are the function prologue and on line 4 is where the function argument is moved into `eax`. What's in `dword [ebp+arg_8h]` though? Remember how the result of _sym.read_line_ is pushed onto the stack? Well, that's what the function argument is, and that's now being put back into `eax`. The really interesting bit now happens between lines 6 and 10. Here the string _Public speaking is very easy._ is pushed onto to the stack. Want to make sure that it's the right string? Use `ps @ 0x80497c0` to print the string at that address.

```sourceCode
[0x080488e0]> ps @ 0x80497c0
Public speaking is very easy.
[0x080488e0]>
```

Then `eax` is pushed onto the stack and _sym.strings_not_equal_ is called. Luckily this is named well enough that we can assume it tests if two strings are equal or not. Returning _true_ if they are **not** the same would be the logical conclusion. Being an auxillary function, it will be covered later on, but for now we just take it for what it is. We'll soon see if we are correct or not. If you are feeling adventurous, by all means, go and take a look at it.

Once we return we test `eax`. Pardon? Test `eax` with `eax`? Yeah well, that's assembly language for you. Stuff happens behind the scenes. Get out the reference manual and take a look at what `test` actually does:

> In the x86 assembly language, the TEST instruction performs a bitwise AND on two operands. The flags SF, ZF, PF are modified while the result of the AND is discarded.

Ok, got it? Good, we're done then. Just kidding, here's what those flags represent and how they are used:

- _SF_ : Sign flag - is set if the result of the previous op is negative
- _ZF_ : Zero flag - is set if the previous op is zero
- _PF_ : Parity flag - is set if the number of set bits of the results of previous op is even

And by set I mean set to 1. Now the _ZF_ is often used for conditional jumps following ops that modify it. And looking at the reference for `je` we see that

> je: Jump short if zero/equal

I'll repeat that bit of code here for context

```sourceCode
|           0x08048b3a      85c0           test eax, eax
|       ,=< 0x08048b3c      7405           je 0x8048b43                ;[2]
|       |   0x08048b3e      e8b9090000     call sym.explode_bomb       ;[3]
|       `-> 0x08048b43      89ec           mov esp, ebp
|           0x08048b45      5d             pop ebp
\           0x08048b46      c3             ret
```

If it turns out we don't take the jump, we end up in _sym.explode_bomb_, which is somewhere we do not want to be. Therefore we want to take the jump, and thus `test eax,eax` has to set the zero flag. That will happen if our input string (read via _sym.read_line_) matches the one at `0x80497c0` causing _sym.strings_not_equal_ to return 0 (or false).

The logic here is a bit awkward, so I'll reiterate

- call _sym.strings_not_equal_
- if strings are not equal returns 1 (true), otherwise 0 (false), in `eax`
- _test_ value in `eax`. If `eax` is 0, _ZF_ will be set, else it will be unset
- jump over _sym.explode_bomb_ if _ZF_ is set

If the strings are not not equal, everything's ok. Pretty sure they did that to mess with our heads and make me write more than I needed to, just to explain that double negative.

Finally the stack is restored, the instruction pointer reset, and the function returns back to _sym.main_ (lines 4-6 above)

Now we know what to enter to defuse phase 1: _Public speaking is easy._

Don't forget that full stop!

Did we figure this out correctly? The proof is in the pudding. So let's run a debug session and confirm our findings.

<iframe width="720" height="415" src="https://www.youtube.com/embed/KuqjYNzKajg" frameborder="0" allowfullscreen></iframe>
Embedding youtube vids with text is a bit potato, so you're better off going direct: [<https://youtu.be/KuqjYNzKajg>](https://youtu.be/KuqjYNzKajg) and even that is a little fuzzy. Need to figure out the best way to record terminals in HD. It's a shame that the embedded player of asciinema doesn't do monospace fonts.

PNow add that string to the file of defuse commands which is were we'll store our progress. Now get ready for phase 2...

[Read Phase 2](https://unlogic.co.uk/2016/04/20/binary-bomb-with-radare2-phase-2/)
