---
extends: default.liquid
title: crackserial_linux with radare2
date: 2016-06-13T19:03:50Z
tags:
  - reverse engineering
  - radare
  - crackme
category: reverse engineering
summary: It's time for another crackme from crackmes.de called crackserial_linux
---

It's time for another crackme from [crackmes.de](https://crackmes.de) called _crackserial_linux_.

[crack serial linux](https://crackmes.de/users/josamont/crack_serial_in_linux/)

## Preliminaries

As usual we'll gather some info before we start looking at the disassembly, and running it is always a good start:

```sourceCode
$ ./crackserial_linux
User: user
Serial: 1234
Try again
```

It requires a username and serial, so chances are it will use the username to generate a valid serial. Might even be worth creating a keygen for this in the end. Right, so infos

```sourceCode
$ r2 crackserial_linux
 -- I'm in your source securing your bits.
[0x080488d0]> iI
havecode true
pic      false
canary   false
nx       true
crypto   false
va       true
intrp    /lib/ld-linux.so.2
bintype  elf
class    ELF32
lang     cxx
arch     x86
bits     32
machine  Intel 80386
os       linux
minopsz  1
maxopsz  16
pcalign  0
subsys   linux
endian   little
stripped true
static   false
linenum  false
lsyms    false
relocs   false
rpath    NONE
binsz    8584
```

We can see it's a 32bit binary written in C++. Imports next

A lot of c++ string imports, no big surprises here. Ok, let's take a peek at the code.

## Looking into main

Let's see what `main` has to offer us

```sourceCode
[0x080488d0]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[ ] [*] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan))
[0x080488d0]> pdf @ main
/ (fcn) main 356
|           ; var int local_4h @ ebp-0x4
|           ; var int arg_4h @ esp+0x4
|           ; var int arg_10h @ esp+0x10
|           ; var int arg_14h @ esp+0x14
|           ; var int arg_18h @ esp+0x18
|           ; var int arg_1ch @ esp+0x1c
|           ; DATA XREF from 0x080488e7 (entry0)
|           0x08048a41      55             push ebp
|           0x08048a42      89e5           mov ebp, esp
|           0x08048a44      53             push ebx
|           0x08048a45      83e4f0         and esp, 0xfffffff0
|           0x08048a48      83ec20         sub esp, 0x20
|           0x08048a4b      8d442410       lea eax, [esp + arg_10h]    ; 0x10
|           0x08048a4f      890424         mov dword [esp], eax
|           0x08048a52      e859fdffff     call sub._ZNSsC1Ev_12_7b0
|           0x08048a57      c744241c0000.  mov dword [esp + arg_1ch], 0
|       ,=< 0x08048a5f      eb38           jmp 0x8048a99
|       |   ; JMP XREF from 0x08048a9e (main)
|       |   ; JMP XREF from 0x08048aa5 (main)
|     ..--> 0x08048a61      c7442404808d.  mov dword [esp + arg_4h], str.User: ; [0x8048d80:4]=0x72657355 LEA str.User: ; "User: " @ 0x8048d80
|     |||   0x08048a69      c7042400b104.  mov dword [esp], obj.std::cout ; [0x804b100:4]=0x742e0074 LEA obj.std::cout ; "t" @ 0x804b100
|     |||   0x08048a70      e8dbfdffff     call sym.std::operator___std::char_traits_char__
|     |||   0x08048a75      8d442410       lea eax, [esp + arg_10h]    ; 0x10
|     |||   0x08048a79      89442404       mov dword [esp + arg_4h], eax
|     |||   0x08048a7d      c7042460b004.  mov dword [esp], obj.std::cin ; [0x804b060:4]=0x62552820 LEA obj.std::cin ; " (Ubuntu 4.8.2-19ubuntu1) 4.8.2" @ 0x804b060
|     |||   0x08048a84      e847fdffff     call sym.std::getline_char_std::char_traits_char__std::allocator_char__
|     |||   0x08048a89      8d442410       lea eax, [esp + arg_10h]    ; 0x10
|     |||   0x08048a8d      890424         mov dword [esp], eax
|     |||   0x08048a90      e89bfdffff     call sym.std::string::length
|     |||   0x08048a95      8944241c       mov dword [esp + arg_1ch], eax
|     |||   ; JMP XREF from 0x08048a5f (main)
|     ||`-> 0x08048a99      837c241c02     cmp dword [esp + arg_1ch], 2 ; [0x2:4]=0x101464c
|     `===< 0x08048a9e      7ec1           jle 0x8048a61
|      |    0x08048aa0      837c241c1e     cmp dword [esp + arg_1ch], 0x1e ; [0x1e:4]=0x21880000
|      `==< 0x08048aa5      7fba           jg 0x8048a61
|           0x08048aa7      c7442404878d.  mov dword [esp + arg_4h], str.Serial: ; [0x8048d87:4]=0x69726553 LEA str.Serial: ; "Serial: " @ 0x8048d87
|           0x08048aaf      c7042400b104.  mov dword [esp], obj.std::cout ; [0x804b100:4]=0x742e0074 LEA obj.std::cout ; "t" @ 0x804b100
|           0x08048ab6      e895fdffff     call sym.std::operator___std::char_traits_char__
|           0x08048abb      c7442404c18c.  mov dword [esp + arg_4h], 0x8048cc1 ; [0x8048cc1:4]=0x83e58955
|           0x08048ac3      c7042460b004.  mov dword [esp], obj.std::cin ; [0x804b060:4]=0x62552820 LEA obj.std::cin ; " (Ubuntu 4.8.2-19ubuntu1) 4.8.2" @ 0x804b060
|           0x08048aca      e811fdffff     call sub._ZNSirsEPFRSt8ios_baseS0_E_24_7e0
|           0x08048acf      8d542414       lea edx, [esp + arg_14h]    ; 0x14
|           0x08048ad3      89542404       mov dword [esp + arg_4h], edx
|           0x08048ad7      890424         mov dword [esp], eax
|           0x08048ada      e8e1fdffff     call sym.std::istream::operator__
|           0x08048adf      8b5c2414       mov ebx, dword [esp + arg_14h] ; [0x14:4]=1
|           0x08048ae3      8d442410       lea eax, [esp + arg_10h]    ; 0x10
|           0x08048ae7      89442404       mov dword [esp + arg_4h], eax
|           0x08048aeb      8d442418       lea eax, [esp + arg_18h]    ; 0x18
|           0x08048aef      890424         mov dword [esp], eax
|           0x08048af2      e829fdffff     call sym.std::basic_string_char_std::char_traits_char__std::allocator_char__::basic_string
|           0x08048af7      895c2404       mov dword [esp + arg_4h], ebx
|           0x08048afb      8d442418       lea eax, [esp + arg_18h]    ; 0x18
|           0x08048aff      890424         mov dword [esp], eax
|           0x08048b07      89c3           mov ebx, eax
|           0x08048b09      8d442418       lea eax, [esp + arg_18h]    ; 0x18
|           0x08048b0d      890424         mov dword [esp], eax
|           0x08048b10      e84bfdffff     call sym.std::basic_string_char_std::char_traits_char__std::allocator_char__::_basic_string
|           0x08048b15      84db           test bl, bl
|       ,=< 0x08048b17      7426           je 0x8048b3f
|       |   0x08048b19      c7442404908d.  mov dword [esp + arg_4h], str.Good_job_ ; [0x8048d90:4]=0x646f6f47 LEA str.Good_job_ ; "Good job!" @ 0x8048d90
|       |   0x08048b21      c7042400b104.  mov dword [esp], obj.std::cout ; [0x804b100:4]=0x742e0074 LEA obj.std::cout ; "t" @ 0x804b100
|       |   0x08048b28      e823fdffff     call sym.std::operator___std::char_traits_char__
|       |   0x08048b2d      c74424048088.  mov dword [esp + arg_4h], sym.std::endl_char_std::char_traits_char__ ; [0x8048880:4]=0xb04025ff LEA sym.std::endl_char_std::char_traits_char__ ; sym.std::endl_char_std::char_traits_char__
|       |   0x08048b35      890424         mov dword [esp], eax
|       |   0x08048b38      e833fdffff     call sym.std::ostream::operator__
|      ,==< 0x08048b3d      eb24           jmp 0x8048b63
|      ||   ; JMP XREF from 0x08048b17 (main)
|      |`-> 0x08048b3f      c74424049a8d.  mov dword [esp + arg_4h], str.Try_again ; [0x8048d9a:4]=0x20797254 LEA str.Try_again ; "Try again" @ 0x8048d9a
|      |    0x08048b47      c7042400b104.  mov dword [esp], obj.std::cout ; [0x804b100:4]=0x742e0074 LEA obj.std::cout ; "t" @ 0x804b100
|      |    0x08048b4e      e8fdfcffff     call sym.std::operator___std::char_traits_char__
|      |    0x08048b53      c74424048088.  mov dword [esp + arg_4h], sym.std::endl_char_std::char_traits_char__ ; [0x8048880:4]=0xb04025ff LEA sym.std::endl_char_std::char_traits_char__ ; sym.std::endl_char_std::char_traits_char__
|      |    0x08048b5b      890424         mov dword [esp], eax
|      |    0x08048b5e      e80dfdffff     call sym.std::ostream::operator__
|      |    ; JMP XREF from 0x08048b3d (main)
|      `--> 0x08048b63      bb00000000     mov ebx, 0
|           0x08048b68      8d442410       lea eax, [esp + arg_10h]    ; 0x10
|           0x08048b6c      890424         mov dword [esp], eax
|           0x08048b6f      e8ecfcffff     call sym.std::basic_string_char_std::char_traits_char__std::allocator_char__::_basic_string
|           0x08048b74      89d8           mov eax, ebx
|       ,=< 0x08048b76      eb28           jmp 0x8048ba0
        |   0x08048b78      89c3           mov ebx, eax
        |   0x08048b7a      8d442418       lea eax, [esp + 0x18]       ; 0x18
        |   0x08048b7e      890424         mov dword [esp], eax
        |   0x08048b81      e8dafcffff     call sym.std::basic_string_char_std::char_traits_char__std::allocator_char__::_basic_string
       ,==< 0x08048b86      eb02           jmp 0x8048b8a
       ||   0x08048b88      89c3           mov ebx, eax
       ||   ; JMP XREF from 0x08048b86 (unk)
       `--> 0x08048b8a      8d442410       lea eax, [esp + 0x10]       ; 0x10
        |   0x08048b8e      890424         mov dword [esp], eax
        |   0x08048b91      e8cafcffff     call sym.std::basic_string_char_std::char_traits_char__std::allocator_char__::_basic_string
        |   0x08048b96      89d8           mov eax, ebx
        |   0x08048b98      890424         mov dword [esp], eax
        |   0x08048b9b      e800fdffff     call sym.imp._Unwind_Resume
|       |   ; JMP XREF from 0x08048b76 (main)
|       `-> 0x08048ba0      8b5dfc         mov ebx, dword [ebp - local_4h]
|           0x08048ba3      c9             leave
\           0x08048ba4      c3             ret
```

What I'm looking for here is some more information what happens with our input, and how the decision between _good_ and _bad_ is made.

Lines 28 to 43 is where the input routine for the _user_ starts. Printing out the **User:** prompt, and reading the input for it. Each time it checks that the input is longer than two but shorter than 30 characters (lines 40 and 42). Following this is the prompt and input for the serial number. There's no checks at this stage, and the first interesting bit happens on line 63 with `call fcn.080489cd`, a function we might want to look into, not only because it has no name, but also because the _good_ and _bad_ exits happen just after this function call. I would hazard a guess that the serial is calculated in this function.

So let's take a look...

## Looking into fcn.080489cd

First I'm going to rename our function to something more memorable, and then print its disassembly

```sourceCode
[0x08048a41]> afn serial_gen  fcn.080489cd
[0x08048a41]> s serial_gen
[0x080489cd]> pdf
/ (fcn) serial_gen 116
|           ; arg int arg_8h @ ebp+0x8
|           ; arg int arg_ch @ ebp+0xc
|           ; var int local_ch @ ebp-0xc
|           ; var int local_10h @ ebp-0x10
|           ; var int local_14h @ ebp-0x14
|           ; var int local_18h @ ebp-0x18
|           ; CALL XREF from 0x08048b02 (main)
|           0x080489cd      55             push ebp
|           0x080489ce      89e5           mov ebp, esp
|           0x080489d0      83ec28         sub esp, 0x28
|           0x080489d3      c745ec000000.  mov dword [ebp - local_14h], 0
|           0x080489da      c745f0004750.  mov dword [ebp - local_10h], 0x4e504700
|           0x080489e1      c745e8000000.  mov dword [ebp - local_18h], 0
|       ,=< 0x080489e8      eb37           jmp 0x8048a21
|       |   ; JMP XREF from 0x08048a34 (serial_gen)
|      .--> 0x080489ea      8b45e8         mov eax, dword [ebp - local_18h]
|      ||   0x080489ed      89442404       mov dword [esp + 4], eax
|      ||   0x080489f1      8b4508         mov eax, dword [ebp + arg_8h] ; [0x8:4]=0
|      ||   0x080489f4      890424         mov dword [esp], eax
|      ||   0x080489f7      e8b4feffff     call sym.std::string::at
|      ||   0x080489fc      0fb600         movzx eax, byte [eax]
|      ||   0x080489ff      0fbec0         movsx eax, al
|      ||   0x08048a02      8945f4         mov dword [ebp - local_ch], eax
|      ||   0x08048a05      8b45f4         mov eax, dword [ebp - local_ch]
|      ||   0x08048a08      8b55f0         mov edx, dword [ebp - local_10h]
|      ||   0x08048a0b      31c2           xor edx, eax
|      ||   0x08048a0d      89d0           mov eax, edx
|      ||   0x08048a0f      c1e003         shl eax, 3
|      ||   0x08048a12      29d0           sub eax, edx
|      ||   0x08048a14      8945f0         mov dword [ebp - local_10h], eax
|      ||   0x08048a17      8b45f0         mov eax, dword [ebp - local_10h]
|      ||   0x08048a1a      0145ec         add dword [ebp - local_14h], eax
|      ||   0x08048a1d      8345e801       add dword [ebp - local_18h], 1
|      ||   ; JMP XREF from 0x080489e8 (serial_gen)
|      |`-> 0x08048a21      8b4508         mov eax, dword [ebp + arg_8h] ; [0x8:4]=0
|      |    0x08048a24      890424         mov dword [esp], eax
|      |    0x08048a27      e804feffff     call sym.std::string::length
|      |    0x08048a2c      3b45e8         cmp eax, dword [ebp - local_18h]
|      |    0x08048a2f      0f97c0         seta al
|      |    0x08048a32      84c0           test al, al
|      `==< 0x08048a34      75b4           jne 0x80489ea
|           0x08048a36      8b450c         mov eax, dword [ebp + arg_ch] ; [0xc:4]=0
|           0x08048a39      3b45ec         cmp eax, dword [ebp - local_14h]
|           0x08048a3c      0f94c0         sete al
|           0x08048a3f      c9             leave
\           0x08048a40      c3             ret
```

So we've got 2 arguments and 4 local variables. I'm going to look over the function and see how much information I can get about these variables in order to determine their purpose.

## local_18h

The biggest clue for this variable is on line 31 where it's compared to `eax` after a call to `sym.std::string::length`. We also see it being increased by _1_ on line 26, which is the end of the loop which starts on line 9. Thus `local_18h` is likely to be a counter for the number of characters we've processed. In that case we'll give it a better name, so we can keep track of it.

```sourceCode
[0x080489cd]> afan local_18h counter
[0x080489cd]> pdf
╒ (fcn) serial_gen 116
│           ; var int counter @ ebp-0x18
│           ; var int local_14h @ ebp-0x14
│           ; var int local_10h @ ebp-0x10
│           ; var int local_ch @ ebp-0xc
│           ; arg int arg_8h @ ebp+0x8
│           ; arg int arg_ch @ ebp+0xc
│           ; CALL XREF from 0x08048b02 (main)
│           0x080489cd      55             push ebp
│           0x080489ce      89e5           mov ebp, esp
│           0x080489d0      83ec28         sub esp, 0x28
│           0x080489d3      c745ec000000.  mov dword [ebp - local_14h], 0
│           0x080489da      c745f0004750.  mov dword [ebp - local_10h], 0x4e504700
│           0x080489e1      c745e8000000.  mov dword [ebp - counter], 0
```

next....

## local_14h

This variable is initialised to _0_ at the start of the loop and then `eax` gets added into it at the end of each loop. Finally, at the end of the function, it is compared to `eax`, which at this point is `[ebp + arg_ch]`. Most likely this will store the generated serial that we should match.

    > \[0x080489cd\]&gt; afan local_14h trgt_serial

## local_10h

On line 5 the immediate value of `0x4e504700` is stored in this variable. Inside the loop it is part of various operations on `eax`, the result of which is ultimately written back to our `trgt_serial`. It's very likely to be some sort of variable that is used in the serial generation. We'll call this `serial_mask`

    > \[0x080489cd\]&gt; afan local_10h serial_mask

## local_ch

This is only used inside the loop and seems to store a single character during the serial generation. Possibly just used to temporarily store a value during each iteration. I dub thee `tmp_char`

    > \[0x080489cd\]&gt; afan local_ch tmp_char

## arg_8h

One of the passed in arguments which is very involved inside the loop. I'm betting it'll be our username entry

    > \[0x080489cd\]&gt; afan arg_8h username

## arg_ch

And finally, with all that's left, this is probably going to be our entered serial. The instruction on line 35 backs this up. It's moved into `eax` and then compared to the generated serial. So...

    > \[0x080489cd\]&gt; afan arg_ch user_serial

Now we're working with this:

```sourceCode
[0x080489cd]> pdf
╒ (fcn) serial_gen 116
│           ; var int counter @ ebp-0x18
│           ; var int trgt_serial @ ebp-0x14
│           ; var int serial_mask @ ebp-0x10
│           ; var int tmp_char @ ebp-0xc
│           ; arg int username @ ebp+0x8
│           ; arg int user_serial @ ebp+0xc
│           ; CALL XREF from 0x08048b02 (main)
│           0x080489cd      55             push ebp
│           0x080489ce      89e5           mov ebp, esp
│           0x080489d0      83ec28         sub esp, 0x28
│           0x080489d3      c745ec000000.  mov dword [ebp - trgt_serial], 0
│           0x080489da      c745f0004750.  mov dword [ebp - serial_mask], 0x4e504700
│           0x080489e1      c745e8000000.  mov dword [ebp - counter], 0
│       ┌─< 0x080489e8      eb37           jmp 0x8048a21
│      ┌──> 0x080489ea      8b45e8         mov eax, dword [ebp - counter]
│      ││   0x080489ed      89442404       mov dword [esp + 4], eax
│      ││   0x080489f1      8b4508         mov eax, dword [ebp + username] ; [0x8:4]=0
│      ││   0x080489f4      890424         mov dword [esp], eax
│      ││   0x080489f7      e8b4feffff     call sym.std::string::at
│      ││   0x080489fc      0fb600         movzx eax, byte [eax]
│      ││   0x080489ff      0fbec0         movsx eax, al
│      ││   0x08048a02      8945f4         mov dword [ebp - tmp_char], eax
│      ││   0x08048a05      8b45f4         mov eax, dword [ebp - tmp_char]
│      ││   0x08048a08      8b55f0         mov edx, dword [ebp - serial_mask]
│      ││   0x08048a0b      31c2           xor edx, eax
│      ││   0x08048a0d      89d0           mov eax, edx
│      ││   0x08048a0f      c1e003         shl eax, 3
│      ││   0x08048a12      29d0           sub eax, edx
│      ││   0x08048a14      8945f0         mov dword [ebp - serial_mask], eax
│      ││   0x08048a17      8b45f0         mov eax, dword [ebp - serial_mask]
│      ││   0x08048a1a      0145ec         add dword [ebp - trgt_serial], eax
│      ││   0x08048a1d      8345e801       add dword [ebp - counter], 1
│      ││   ; JMP XREF from 0x080489e8 (serial_gen)
│      │└─> 0x08048a21      8b4508         mov eax, dword [ebp + username] ; [0x8:4]=0
│      │    0x08048a24      890424         mov dword [esp], eax
│      │    0x08048a27      e804feffff     call sym.std::string::length
│      │    0x08048a2c      3b45e8         cmp eax, dword [ebp - counter]
│      │    0x08048a2f      0f97c0         seta al
│      │    0x08048a32      84c0           test al, al
│      └──< 0x08048a34      75b4           jne 0x80489ea
│           0x08048a36      8b450c         mov eax, dword [ebp + user_serial] ; [0xc:4]=0
│           0x08048a39      3b45ec         cmp eax, dword [ebp - trgt_serial]
│           0x08048a3c      0f94c0         sete al
│           0x08048a3f      c9             leave
╘           0x08048a40      c3             ret
```

## Generating the serial

All the work is done between lines 19 and 25 and is fairly easy to pick apart, now that we've renamed all our variables. Basically the operations in between those lines are performed on each character of our entered username in turn. I'll go through it line by line so we can create our keygen at the end

- line 19 : the current char is xored with the serial mask and stored in `edx`
- line 20 : the result of the xor is put mack into `eax`
- line 21 : `eax` is the shifted left by 3
- line 22 : the result of the xor operation is subtracted from the result of the left shift
- line 23 : the result from the previous op replaces the `serial_mask`
- line 24 : `eax` (new serial_mask) is add to the `trgt_serial`

Therefore the target serial is an accumulator of the previous operations. Now that we know what this routine does, we can write a keygen for it:

```python
#!/usr/bin/env python

user_name = ''
while len(user_name) < 3 or len(user_name) > 30:
    user_name = raw_input("User:")

mask = 0x4e504700
tmp = 0
serial = 0

for n in user_name.strip():
    t = mask ^ ord(n)
    x = 0xFFFFFFFF & (t << 3)
    t = x - t
    mask = t
    serial += t

serial = 0xFFFFFFFF & serial
print hex(serial)[2:]
```

Apart from the two **and** ops, this mirrors what the serial_gen function does. The two **and** ops are to keep the results limited to 8bits, otherwise we would get incorrect values. All that's left to do now is to run the Python script and verify it's correct. I'll leave that as an exercise for the reader.

## Summary

This is another **level 1** crackme, but a little bit more involved than the last one we did. This time, instead of just finding the flag we examined the serial generation routine and reversed it in order to produce a keygen. Using Radare's ability to rename functions and variables made it easier to follow along see what the purpose of each of the **local\_** variables was. No dynamic analysis this time as it was possible to do the whole thing statically. Of course, had it turned out that the keygen was wrong, I'd probably have debugged it to see which one of my assumptions was incorrect.

Thanks for reading
