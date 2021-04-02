---
extends: default.liquid
title: Binary Bomb with Radare2 - Secret Phase
date: 2016-06-06T10:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

Finally we come to that secret phase we saw at the very beginning. This is a two stage process. First we need to figure out how we get to it, and then we need to figure out how to solve it. So, let's hop to it

## Journey to the secret

After loading the binary up in r2, let's hop to the phase in question

```sourceCode
/ (fcn) sym.secret_phase 96
|           ; var int local_18h @ ebp-0x18
|           ; CALL XREF from 0x0804958d (sym.phase_defused)
|           0x08048ee8      55             push ebp
|           0x08048ee9      89e5           mov ebp, esp
```

One _XREF_ from _sym.phase_defused_, so let's head there and find out what needs to happen for it to call our secret phase. Clearly this is called on every phase, so there must be a specific condition that causes it to call _sym.secret_phase_

```sourceCode
/ (fcn) sym.phase_defused 122
|           ; var int local_68h @ ebp-0x68
|           ; var int local_54h @ ebp-0x54
|           ; var int local_50h @ ebp-0x50
|           ; XREFS: CALL 0x08048a60  CALL 0x08048a83  CALL 0x08048aa6  CALL 0x08048ac9  CALL 0x08048aec  CALL 0x08048b0f  CALL 0x08048f3c
|           0x0804952c      55             push ebp
|           0x0804952d      89e5           mov ebp, esp
|           0x0804952f      83ec64         sub esp, 0x64
|           0x08049532      53             push ebx
|           ; DATA XREF from 0x0804b480 (unk)
|           0x08049533      833d80b40408.  cmp dword [obj.num_input_strings], 6 ; [0x6:4]=1
|       ,=< 0x0804953a      7563           jne 0x804959f               ;[1]
|       |   0x0804953c      8d5db0         lea ebx, [ebp - local_50h]  ;[2]
|       |   0x0804953f      53             push ebx
|       |   0x08049540      8d45ac         lea eax, [ebp - local_54h]  ;[3]
|       |   0x08049543      50             push eax
|       |   0x08049544      68039d0408     push str._d__s ; str._d__s  ; "%d %s" @ 0x8049d03
|       |   0x08049549      6870b70408     push 0x804b770
|       |   0x0804954e      e80df3ffff     call sym.imp.sscanf         ;[4]
|       |   0x08049553      83c410         add esp, 0x10
|       |   0x08049556      83f802         cmp eax, 2                  ; "LF..." @ 0x2
|      ,==< 0x08049559      7537           jne 0x8049592               ;[5]
|      ||   0x0804955b      83c4f8         add esp, -8
|      ||   0x0804955e      68099d0408     push str.austinpowers ; str.austinpowers ; "austinpowers" @ 0x8049d09
|      ||   0x08049563      53             push ebx
|      ||   0x08049564      e8c7faffff     call sym.strings_not_equal  ;[6]
|      ||   0x08049569      83c410         add esp, 0x10
|      ||   0x0804956c      85c0           test eax, eax
|     ,===< 0x0804956e      7522           jne 0x8049592               ;[5]
|     |||   0x08049570      83c4f4         add esp, -0xc
|     |||   0x08049573      68209d0408     push str.Curses__you_ve_found_the_secret_phase__n ; str.Curses__you_ve_found_the_secret_phase__n ; "Curses, you've found the secret phas
|     |||   0x08049578      e893f2ffff     call sym.imp.printf         ;[7]
|     |||   0x0804957d      83c4f4         add esp, -0xc
|     |||   0x08049580      68609d0408     push str.But_finding_it_and_solving_it_are_quite_different..._n ; str.But_finding_it_and_solving_it_are_quite_different..._n ; "But find
|     |||   0x08049585      e886f2ffff     call sym.imp.printf         ;[7]
|     |||   0x0804958a      83c420         add esp, 0x20
|     |||   0x0804958d      e856f9ffff     call sym.secret_phase       ;[8]
|     ``--> 0x08049592      83c4f4         add esp, -0xc
|       |   0x08049595      68a09d0408     push str.Congratulations__You_ve_defused_the_bomb__n ; str.Congratulations__You_ve_defused_the_bomb__n ; "Congratulations! You've defuse
|       |   0x0804959a      e871f2ffff     call sym.imp.printf         ;[7]
|       `-> 0x0804959f      8b5d98         mov ebx, dword [ebp - local_68h]
|           0x080495a2      89ec           mov esp, ebp
|           0x080495a4      5d             pop ebp
\           0x080495a5      c3             ret
```

After the function prologe we see a reference to an object `obj.num_input_strings` which resides at `0x0804b480`. This seems to store a number, most likely the number of strings we've entered. Let's take a look at some of the XREFs for those object and see where the interesting bits are..... after some checking, this seems relevant

```sourceCode
/ (fcn) sym.read_line 193
|           ; var int local_18h @ ebp-0x18
|           ; XREFS: CALL 0x08048a52  CALL 0x08048a75  CALL 0x08048a98  CALL 0x08048abb  CALL 0x08048ade  CALL 0x08048b01
|           ; XREFS: CALL 0x08048eef
|           0x080491fc      55             push ebp
|           0x080491fd      89e5           mov ebp, esp
|           0x080491ff      83ec14         sub esp, 0x14
|           0x08049202      57             push edi
|           0x08049203      e8a8ffffff     call sym.skip               ;[1]
|           0x08049208      85c0           test eax, eax
|       ,=< 0x0804920a      7553           jne 0x804925f               ;[2]
|       |   ; DATA XREF from 0x0804b664 (unk)
|       |   0x0804920c      a164b60408     mov eax, dword [obj.infile] ; [0x804b664:4]=43 LEA obj.infile ; "+" @ 0x804b664
|       |   0x08049211      3b0548b60408   cmp eax, dword [obj.stdin__GLIBC_2.0] ; [0x804b648:4]=0x134f section_end..stabstr LEA ob
|      ,==< 0x08049217      7431           je 0x804924a                ;[3]
|      ||   0x08049219      83c4f4         add esp, -0xc
|      ||   0x0804921c      687f9b0408     push str.GRADE_BOMB ; str.GRADE_BOMB ; "GRADE_BOMB" @ 0x8049b7f
|      ||   0x08049221      e83af5ffff     call sym.imp.getenv         ;[4]
|      ||   0x08049226      83c410         add esp, 0x10
|      ||   0x08049229      85c0           test eax, eax
|     ,===< 0x0804922b      740a           je 0x8049237                ;[5]
|     |||   0x0804922d      83c4f4         add esp, -0xc
|     |||   0x08049230      6a00           push 0
|     |||   0x08049232      e819f6ffff     call sym.imp.exit           ;[6]
|     `---> 0x08049237      a148b60408     mov eax, dword [obj.stdin__GLIBC_2.0] ; [0x804b648:4]=0x134f section_end..stabstr LEA ob
|      ||   ; DATA XREF from 0x0804b664 (unk)
|      ||   0x0804923c      a364b60408     mov dword [obj.infile], eax ; [0x804b664:4]=43 LEA obj.infile ; "+" @ 0x804b664
|      ||   0x08049241      e86affffff     call sym.skip               ;[1]
|      ||   0x08049246      85c0           test eax, eax
|     ,===< 0x08049248      7515           jne 0x804925f               ;[2]
|     |`--> 0x0804924a      83c4f4         add esp, -0xc
|     | |   0x0804924d      68609b0408     push str.Error:_Premature_EOF_on_stdin_n ; str.Error:_Premature_EOF_on_stdin_n ; "Error:
|     | |   0x08049252      e8b9f5ffff     call sym.imp.printf         ;[7]
|     | |   0x08049257      e8a0020000     call sym.explode_bomb       ;[8]
|     | |   0x0804925c      83c410         add esp, 0x10
|     | |   ; DATA XREF from 0x0804b480 (unk)
|     `-`-> 0x0804925f      a180b40408     mov eax, dword [obj.num_input_strings] ; [0x804b480:4]=0 LEA obj.num_input_strings ; obj
|           0x08049264      8d0480         lea eax, [eax + eax*4]      ;[9]
|           0x08049267      c1e004         shl eax, 4
|           0x0804926a      8db880b60408   lea edi, [eax + obj.input_strings] ;[?] ; 0x804b680 ; obj.input_strings ; obj.input_stri
|           0x08049270      b000           mov al, 0
|           0x08049272      fc             cld
|           0x08049273      b9ffffffff     mov ecx, loc.imp.__gmon_start__ ; loc.imp.__gmon_start__
|           0x08049278      f2ae           repne scasb al, byte es:[edi]
|           0x0804927a      89c8           mov eax, ecx
|           0x0804927c      f7d0           not eax
|           0x0804927e      8d78ff         lea edi, [eax - 1]          ;[?]
|           0x08049281      83ff4f         cmp edi, 0x4f               ; 'O'
|       ,=< 0x08049284      7512           jne 0x8049298               ;[?]
|       |   0x08049286      83c4f4         add esp, -0xc
|       |   0x08049289      688a9b0408     push str.Error:_Input_line_too_long_n ; str.Error:_Input_line_too_long_n ; "Error: Input
|       |   0x0804928e      e87df5ffff     call sym.imp.printf         ;[?]
|       |   0x08049293      e864020000     call sym.explode_bomb       ;[?]
|       |   ; DATA XREF from 0x0804b480 (unk)
|       `-> 0x08049298      a180b40408     mov eax, dword [obj.num_input_strings] ; [0x804b480:4]=0 LEA obj.num_input_strings ; obj
|           0x0804929d      8d0480         lea eax, [eax + eax*4]      ;[?]
|           0x080492a0      c1e004         shl eax, 4
|           0x080492a3      c684077fb604.  mov byte [edi + eax + 0x804b67f], 0 ; [0x804b67f:1]=0
|           0x080492ab      0580b60408     add eax, obj.input_strings
|           ; DATA XREF from 0x0804b480 (unk)
|           0x080492b0      ff0580b40408   inc dword [obj.num_input_strings]
|           0x080492b6      8b7de8         mov edi, dword [ebp - local_18h]
|           0x080492b9      89ec           mov esp, ebp
|           0x080492bb      5d             pop ebp
\           0x080492bc      c3             ret
```

It seems that this function does a little more than just read a line from stdin or a file. The interesting parts are between lines 33 and 55. First the current value of the number of input strings is moved into `eax` which is then multiplied by 4 and added to itself. Then `shl eax, 4` shifts `eax` left by 4 bits. This is effectively multiplying the value by 2^4. The current input string at `obj.input_strings` is then loaded into `edi`. The `al` register is then set to _0_. The `cld` instruction then clears the direction flag, which controls the direction of string processing. The purpose of these instructions becomes apparent two lines down when we see `repne scasb al, byte es:[edi]`. This will find the value at `al` starting at `edi`. Or, in simpler terms, find the NULL terminator of the input string, and even simpler, how long is the input string.

Once we have that value we check to see if `eax -1` is equal to 79 (`0x4f`). This lines up with the offset calculation above. It seems the strings are 80 characters long and stored consecutivly in memory. The first part of this block gets the memory offset to the current string. All being well, (i.e. our string isn't too long), we do the same multiplication and shift again and now store the _input_string_ at `eax` after placing a _0_ at `edi + eax + 0x804b67f`. Make a note of that memory location, we'll need it later I'm sure. Finally the number of input strings is increased and we return out of the function.

Ok, so now back in _sym.phase_defused_

```sourceCode
|           0x08049533      833d80b40408.  cmp dword [obj.num_input_strings], 6 ; [0x6:4]=1
|       ,=< 0x0804953a      7563           jne 0x804959f               ;[1]
|       |   0x0804953c      8d5db0         lea ebx, [ebp - local_50h]  ;[2]
|       |   0x0804953f      53             push ebx
|       |   0x08049540      8d45ac         lea eax, [ebp - local_54h]  ;[3]
|       |   0x08049543      50             push eax
|       |   0x08049544      68039d0408     push str._d__s ; str._d__s  ; "%d %s" @ 0x8049d03
|       |   0x08049549      6870b70408     push 0x804b770
|       |   0x0804954e      e80df3ffff     call sym.imp.sscanf         ;[4]
|       |   0x08049553      83c410         add esp, 0x10
|       |   0x08049556      83f802         cmp eax, 2                  ; "LF..." @ 0x2
|      ,==< 0x08049559      7537           jne 0x8049592               ;[5]
|      ||   0x0804955b      83c4f8         add esp, -8
|      ||   0x0804955e      68099d0408     push str.austinpowers ; str.austinpowers ; "austinpowers" @ 0x8049d09
|      ||   0x08049563      53             push ebx
|      ||   0x08049564      e8c7faffff     call sym.strings_not_equal  ;[6]
|      ||   0x08049569      83c410         add esp, 0x10
|      ||   0x0804956c      85c0           test eax, eax
|     ,===< 0x0804956e      7522           jne 0x8049592               ;[5]
|     |||   0x08049570      83c4f4         add esp, -0xc
|     |||   0x08049573      68209d0408     push str.Curses__you_ve_found_the_secret_phase__n ; str.Curses__you_ve_found_the_secret_phase__n ; "Curses, you've found the secret phas
|     |||   0x08049578      e893f2ffff     call sym.imp.printf         ;[7]
|     |||   0x0804957d      83c4f4         add esp, -0xc
|     |||   0x08049580      68609d0408     push str.But_finding_it_and_solving_it_are_quite_different..._n ; str.But_finding_it_and_solving_it_are_quite_different..._n ; "But find
|     |||   0x08049585      e886f2ffff     call sym.imp.printf         ;[7]
|     |||   0x0804958a      83c420         add esp, 0x20
|     |||   0x0804958d      e856f9ffff     call sym.secret_phase       ;[8]
|     ``--> 0x08049592      83c4f4         add esp, -0xc
|       |   0x08049595      68a09d0408     push str.Congratulations__You_ve_defused_the_bomb__n ; str.Congratulations__You_ve_defused_the_bomb__n ; "Congratulations! You've defuse
|       |   0x0804959a      e871f2ffff     call sym.imp.printf         ;[7]
|       `-> 0x0804959f      8b5d98         mov ebx, dword [ebp - local_68h]
|           0x080495a2      89ec           mov esp, ebp
|           0x080495a4      5d             pop ebp
\           0x080495a5      c3             ret
```

So first of all we see that the total number of input strings has to be 6. So this check will only run on the 6th stage. At lines 7 - 11 a call to _sym.imp.sscanf_ reads a number and a string from `0x804b770`. What is at that address? Well our inputs are at `0x804b67f` and beyond, each being 80 characters long. The difference between the two addresses is 241 (dec), which will be 240 bytes between the addresses, dividing that by 80 gives us: 3, that magic number. This is where the input from phase 4 will reside (0 indexed). Remember that in [phase 4](https://unlogic.co.uk/2016/05/05/binary-bomb-with-radare2-phase-4/) the answer was a single digit? Well that would satisfy the first part of the _sscanf_ format string. Let's keep going and see what else we need.

Ah, a call to our friend _sym.strings_not_equal_ comparing it with `str.austinpowers`. Looking at the comparisions and conditional jumps, this seems to unlock the secret phase. So to get to the secret phase we need to supply `9 austinpowers` as our answer to phase 4.

## Solving the secret phase

Here's the phase itself

```sourceCode
/ (fcn) sym.secret_phase 96
|           ; var int local_18h @ ebp-0x18
|           ; CALL XREF from 0x0804958d (sym.phase_defused)
|           0x08048ee8      55             push ebp
|           0x08048ee9      89e5           mov ebp, esp
|           0x08048eeb      83ec14         sub esp, 0x14
|           0x08048eee      53             push ebx
|           0x08048eef      e808030000     call sym.read_line          ;[1]
|           0x08048ef4      6a00           push 0
|           0x08048ef6      6a0a           push 0xa
|           0x08048ef8      6a00           push 0
|           0x08048efa      50             push eax
|           0x08048efb      e8f0f8ffff     call sym.imp.__strtol_internal ;[2]
|           0x08048f00      83c410         add esp, 0x10
|           0x08048f03      89c3           mov ebx, eax
|           0x08048f05      8d43ff         lea eax, [ebx - 1]          ;[0]
|           0x08048f08      3de8030000     cmp eax, 0x3e8
|       ,=< 0x08048f0d      7605           jbe 0x8048f14               ;[3]
|       |   0x08048f0f      e8e8050000     call sym.explode_bomb       ;[4]
|       `-> 0x08048f14      83c4f8         add esp, -8
|           0x08048f17      53             push ebx
|           0x08048f18      6820b30408     push obj.n1 ; obj.n1        ; "$" @ 0x804b320
|           0x08048f1d      e872ffffff     call sym.fun7               ;[5]
|           0x08048f22      83c410         add esp, 0x10
|           0x08048f25      83f807         cmp eax, 7
|       ,=< 0x08048f28      7405           je 0x8048f2f                ;[6]
|       |   0x08048f2a      e8cd050000     call sym.explode_bomb       ;[4]
|       `-> 0x08048f2f      83c4f4         add esp, -0xc
|           0x08048f32      6820980408     push str.Wow__You_ve_defused_the_secret_stage__n ; str.Wow__You_ve_defused_the_secret_st
|           0x08048f37      e8d4f8ffff     call sym.imp.printf         ;[7]
|           0x08048f3c      e8eb050000     call sym.phase_defused      ;[8]
|           0x08048f41      8b5de8         mov ebx, dword [ebp - local_18h]
|           0x08048f44      89ec           mov esp, ebp
|           0x08048f46      5d             pop ebp
\           0x08048f47      c3             ret
```

Ok, so looks like it needs a number that is less or equal to 1000 after having 1 subtracted from it. That covers lines 1 -15. Nothing much new here for us. Then we have a new obj `obj.n1` which, along with our input value is passed into _sym.fun7_. The expected return value from _sym.fun7_ is 7, so let's peek at this _fun_ function and see what it does.

````sourceCode
/ (fcn) sym.fun7 82
|           ; arg int arg_8h @ ebp+0x8
|           ; arg int arg_ch @ ebp+0xc
|           ; CALL XREF from 0x08048ed1 (sym.fun7)
|           ; CALL XREF from 0x08048ebc (sym.fun7)
|           ; CALL XREF from 0x08048f1d (sym.secret_phase)
|           0x08048e94      55             push ebp
|           0x08048e95      89e5           mov ebp, esp
|           0x08048e97      83ec08         sub esp, 8
|           0x08048e9a      8b5508         mov edx, dword [ebp + arg_8h] ; [0x8:4]=0
|           0x08048e9d      8b450c         mov eax, dword [ebp + arg_ch] ; [0xc:4]=0
|           0x08048ea0      85d2           test edx, edx
|       ,=< 0x08048ea2      750c           jne 0x8048eb0               ;[1]
|       |   0x08048ea4      b8ffffffff     mov eax, 0xffffffff         ; ebp
|      ,==< 0x08048ea9      eb37           jmp 0x8048ee2               ;[2]
       ||   0x08048eab      90             nop
       ||   0x08048eac      8d742600       lea esi, [esi]              ;[3]
|      |`-> 0x08048eb0      3b02           cmp eax, dword [edx]
|      |,=< 0x08048eb2      7d11           jge 0x8048ec5               ;[4]
|      ||   0x08048eb4      83c4f8         add esp, -8
|      ||   0x08048eb7      50             push eax
|      ||   0x08048eb8      8b4204         mov eax, dword [edx + 4]    ; [0x4:4]=0x10101
|      ||   0x08048ebb      50             push eax
|      ||   0x08048ebc      e8d3ffffff     call sym.fun7               ;[5]
|      ||   0x08048ec1      01c0           add eax, eax
|     ,===< 0x08048ec3      eb1d           jmp 0x8048ee2               ;[2]
|     ||`-> 0x08048ec5      3b02           cmp eax, dword [edx]
|     ||,=< 0x08048ec7      7417           je 0x8048ee0                ;[6]
|     |||   0x08048ec9      83c4f8         add esp, -8
|     |||   0x08048ecc      50             push eax
|     |||   0x08048ecd      8b4208         mov eax, dword [edx + 8]    ; [0x8:4]=0
|     |||   0x08048ed0      50             push eax
|     |||   0x08048ed1      e8beffffff     call sym.fun7               ;[5]
|     |||   0x08048ed6      01c0           add eax, eax
|     |||   0x08048ed8      40             inc eax
|    ,====< 0x08048ed9      eb07           jmp 0x8048ee2               ;[2]
     ||||   0x08048edb      90             nop
     ||||   0x08048edc      8d742600       lea esi, [esi]              ;[3]
|    |||`-> 0x08048ee0      31c0           xor eax, eax
|    |||    ; JMP XREF from 0x08048ed9 (sym.fun7)
|    |||    ; JMP XREF from 0x08048ec3 (sym.fun7)
|    |||    ; JMP XREF from 0x08048ea9 (sym.fun7)
|    ```--> 0x08048ee2      89ec           mov esp, ebp
|           0x08048ee4      5d             pop ebp
\           0x08048ee5      c3             ret
````

Right, so our arguments to this function are our input value in `ebp + arg_ch` and the object `obj.n1`. Let's see if we can't figure out what this object is:

```sourceCode
[0x08048e94]> s obj.n1
[0x0804b320]> pd
            ;-- n1:
            ; DATA XREF from 0x08048f18 (sym.secret_phase)
            0x0804b320      2400           and al, 0
            0x0804b322      0000           add byte [eax], al
            0x0804b324 .dword 0x0804b314 ; obj.n21
            0x0804b328 .dword 0x0804b308 ; obj.n22
```

Looks a little like this:

```sourceCode
struct n {
    int value;
    n *a;
    n *b;
};
```

Something a little like a binary tree perhaps?

```sourceCode
struct node {
    int value;
    n *left;
    n *right;
};
```

Let's use `pf` again like we did in [phase 6](https://unlogic.co.uk/2016/05/27/binary-bomb-with-radare2-phase-6)

```sourceCode
:> pf.node i*?*? value (node)l (node)r
:> pf.node @ obj.n1
  value : 0x0804b320 = 36
      l : (*0x804b314)
    struct<node>
  value : 0x0804b314 = 8
      l : (*0x804b2e4)
    struct<node>
  value : 0x0804b2e4 = 6
      l : (*0x804b2c0)
    struct<node>
  value : 0x0804b2c0 = 1
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b29c)
    struct<node>
  value : 0x0804b29c = 7
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b2fc)
    struct<node>
  value : 0x0804b2fc = 22
      l : (*0x804b290)
    struct<node>
  value : 0x0804b290 = 20
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b2a8)
    struct<node>
  value : 0x0804b2a8 = 35
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b308)
    struct<node>
  value : 0x0804b308 = 50
      l : (*0x804b2f0)
    struct<node>
  value : 0x0804b2f0 = 45
      l : (*0x804b2cc)
    struct<node>
  value : 0x0804b2cc = 40
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b284)
    struct<node>
  value : 0x0804b284 = 47
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b2d8)
    struct<node>
  value : 0x0804b2d8 = 107
      l : (*0x804b2b4)
    struct<node>
  value : 0x0804b2b4 = 99
      l : (*0x0) NULL
      r : (*0x0) NULL
      r : (*0x804b278)
    struct<node>
  value : 0x0804b278 = 1001
      l : (*0x0) NULL
      r : (*0x0) NULL
```

This is the binary tree in a flat list... with the pointers in a bit of a mess. Seems like it follows one side, then when it reaches the end follows the next.... regardless, here's the tree:

![image](/images/bin_tree.png)

There's also that special number _1001_, so let's try and figure out how we can get _sym.fun7_ to return _7_. It's a recursive function call that checks our value from `eax` against the current node's value. Following through the conditional jumps we can see that on line 12 it compares the value to the current node, if it is not greater or equal, go doen the left node. Once returned from there double `eax` and return. This doesn't seem to get what we want, as 7 is clearly not going to happen here. Essentially it will traverse down until the node matches, and return a the number of left nodes visited times two.

What about if we take the other branch at line 13? Here we'll do the same, except in this case we also call `inc eax` which means, that we visit 3 nodes, times 2, plus 1 **if** we go to 1001. It's all making sense now.

Let's try using that :)

```sourceCode
Curses, you've found the secret phase!
But finding it and solving it are quite different...
1001
Wow! You've defused the secret stage!
Congratulations! You've defused the bomb!
```

[_BOOM_](https://i.imgur.com/T0e5SaK.gif)

## THE END

Thanks for reading this rather long series of posts. I hope you learned something from it, because I certainly did. I've not even scratched the surface of r2, but I'm getting more comfortable with it for sure. Hoping to bring some more RE posts in the future, perhaps with some crackmes or other interesting stuff, and perhaps more walkthroughs for Vulnhub VMs

Cheers and have a nice day :)
