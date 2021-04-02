---
title: UnChapelierFou's var crackme [easy]
date: 2016-06-09T21:03:50Z
tags:
  - reverse engineering
  - radare
  - crackme
category: reverse engineering
---

Today we'll be going at an easy crackme from [crackmes.de](https://crackmes.de) with our good buddy **radare2**. So fire up your shells, run **sys/install.sh** and tag along.

Download: <https://crackmes.de/users/unchapelierfou/var/download> (account required)

## Recon

Let's just check out what we're dealing with and then run it to see what it wants from us

```sourceCode
$ rabin2 -I var
Warning: Cannot initialize dynamic strings
havecode true
pic      false
canary   false
nx       true
crypto   false
va       true
bintype  elf
class    ELF64
lang     c
arch     x86
bits     64
machine  AMD x86-64 architecture
os       linux
minopsz  1
maxopsz  16
pcalign  0
subsys   linux
endian   little
stripped true
static   true
linenum  false
lsyms    false
relocs   false
rpath    NONE
binsz    794042
$ rabin2 -l var
Warning: Cannot initialize dynamic strings
[Linked libraries]

0 libraries
$ rabin2 -i var
Warning: Cannot initialize dynamic strings
[Imports]

0 imports
$ ./var
Flag : isthisit?
No dude =/
```

This is a 64bit stripped binary written in C. Running it shows that it needs a flag, nothing else.

## Easy way?

If it's just a chunk of text it's after, perhaps we can find something in the strings of the file. If you run `strings` on the file, you'll see that there's a lot of strings in there. Grepping for **flag** however

```sourceCode
$ strings ./var | grep -i flag
Flag_OveH
Flag :
WARNING: Unsupported flag value(s) of 0x%x in DT_FLAGS_1.
s->_flags2 & 4
version == ((void *)0) || (flags & ~(DL_LOOKUP_ADD_DEPENDENCY | DL_LOOKUP_GSCOPE_LOCK)) == 0
imap->l_type == lt_loaded && (imap->l_flags_1 & 0x00000008) == 0
```

The first result looks interesting.... but no dice. Radare time

## Doing it dynamically

Going to load it up with `r2 -d var` and debug it.

```sourceCode
$ r2 -d var
Process with PID 23451 started...
attach 23451 23451
bin.baddr 0x00400000
Assuming filepath ./var
Warning: Cannot initialize dynamic strings
asm.bits 64
 -- Your project name should contain an uppercase letter, 8 vowels, some numbers, and the first 5 numbers of your private bitcoin key.
[0x00400f4e]> s main
[0x0040106e]> Vp

 ;-- main:
        0x0040106e      55             push rbp
        0x0040106f      4889e5         mov rbp, rsp
        0x00401072      4883ec50       sub rsp, 0x50
        0x00401076      897dbc         mov dword [rbp - 0x44], edi
        0x00401079      488975b0       mov qword [rbp - 0x50], rsi
        0x0040107d      64488b042528.  mov rax, qword fs:[0x28]    ; [0x28:8]=-1 ; '(' ; 40
        0x00401086      488945f8       mov qword [rbp - 8], rax
        0x0040108a      31c0           xor eax, eax
        0x0040108c      48b8466c6167.  movabs rax, 0x65764f5f67616c46
        0x00401096      488945c0       mov qword [rbp - 0x40], rax
        0x0040109a      48b8725f4865.  movabs rax, 0x445f657265485f72
        0x004010a4      488945c8       mov qword [rbp - 0x38], rax
        0x004010a8      c745d0756465.  mov dword [rbp - 0x30], 0x656475
        0x004010af      bf84394900     mov edi, str.Flag_:         ; "Flag : " @ 0x493984
        0x004010b4      b800000000     mov eax, 0
        0x004010b9      e8626e0000     call 0x407f20               ;[1]
        0x004010be      488d45e0       lea rax, [rbp - 0x20]       ;[2]
        0x004010c2      4889c6         mov rsi, rax
        0x004010c5      bf8c394900     mov edi, 0x49398c
        0x004010ca      b800000000     mov eax, 0
        0x004010cf      e87c6f0000     call 0x408050               ;[3]
        0x004010d4      488d55e0       lea rdx, [rbp - 0x20]       ;[2]
        0x004010d8      488d45c0       lea rax, [rbp - 0x40]       ;[4]
        0x004010dc      4889d6         mov rsi, rdx
        0x004010df      4889c7         mov rdi, rax
        0x004010e2      e849f2ffff     call 0x400330               ;[5]
        0x004010e7      85c0           test eax, eax
    ┌─< 0x004010e9      750c           jne 0x4010f7                ;[6]
    │   0x004010eb      bf8f394900     mov edi, str.Congratz_..._easy_wasn_t_it__ ; "Congratz ... easy wasn't it ?" @ 0x49398f
    │   0x004010f0      e87b780000     call 0x408970               ;[7]
   ┌──< 0x004010f5      eb0a           jmp 0x401101                ;[8]
   │└─> 0x004010f7      bfad394900     mov edi, str.No_dude___     ; "No dude =/" @ 0x4939ad
   │    0x004010fc      e86f780000     call 0x408970               ;[7]
   └──> 0x00401101      b800000000     mov eax, 0
        0x00401106      488b4df8       mov rcx, qword [rbp - 8]
        0x0040110a      6448330c2528.  xor rcx, qword fs:[0x28]
    ┌─< 0x00401113      7405           je 0x40111a                 ;[9]
    │   0x00401115      e846640300     call 0x437560               ;[?]
    └─> 0x0040111a      c9             leave
        0x0040111b      c3             ret
```

Taking a quick look over this we can see the good and bad exists after a test of `eax`. At `0x004010af` the prompt string is moved into `edi`. The call to `0x407f20` will likely be a print function. At `0x004010c5` there's an address which we'll look at

```sourceCode
:> ps @0x49398c
%s
```

Interesting, so it would seem that the call to `0x408050` at `0x004010cf` reads our input. The interesting bit (to us) is then this chunk:

```sourceCode
0x004010d4      488d55e0       lea rdx, [rbp - 0x20]       ;[2]
0x004010d8      488d45c0       lea rax, [rbp - 0x40]       ;[4]
```

Most likely where our string and the flag will get moved to `rdx` and `rax` respectively. Thus `call 0x400330` is probably a function to compare the two strings. So let's set a breakpoint at `0x004010dc` and then take a look at what's at `rdx` and `rax`

```sourceCode
[0x0040106e]> db 0x004010dc
[0x0040106e]> dc
Flag : idunno
hit breakpoint at: 4010dc
attach 23451 1
[0x004010dc]> ps @ rdx
idunno
[0x004010dc]> ps @ rax
Flag_Over_Here_Dude
[0x004010dc]>
```

Now just to be sure

```sourceCode
$ ./var
Flag : Flag_Over_Here_Dude
Congratz ... easy wasn't it ?
```

Yeah, that was actually quite easy, thanks :)
