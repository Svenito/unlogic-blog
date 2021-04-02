---
title: Examining malware with r2
date: 2017-06-28T14:16:50Z
tags:
  - malware
  - r2
  - reversing
authors: Sven Steinbauer
summary: So @malwareunicorn wrote a great "Malware RE101" article and I thought I would go over the static analysis and debugging chapter with our good chum radare2
---

So [@malwareunicorn](https://twitter.com/malwareunicorn) wrote a great "Malware RE101" article and I thought I would go over the static analysis chapter
with our good chum radare2. I like to learn, and I like to share, so here it is.

It's a good idea to go and read the [original article](https://securedorg.github.io/RE101/section5) and then come back here if you
are very new to malware analysis. If you are just here to learn more about radare2, then grab a beer and take a seat.

The chapter I am going to go over cover static analysis of an _unknown_ malware using windows tools.
These tools are great, but I'm not a big user of Windows. I have some Windows VMs that I use to run samples in, but
that's about it. For the most part I am a Linux person. So I'm going to go over this chapter using Linux tools,
mostly from within [Remnux](https://remnux.org) as there's a lot of PE tools packaged in with the repo.

But before we start you **must** update/install r2 from git in Remnux, otherwise you are working with a dinosaur version and
you won't enjoy it. All you have to do is clone the repo and run `sys/install.sh` from the directory.

All set? Let's get on with it then

## Static Analysis

First we need to examine the PE file for any suspicious imports, headers and the like. For this use `PEScanner` on the unzipped
file:

EDIT: You can actually get all this info with r2 too using the `i?` set of tools, or with rabin2.

```shell
$ pescanner Unknown
##########################################################################################
[0] File: Unknown
##########################################################################################

Meta-data
==========================================================================================
Size		: 376832 bytes
Type		: PE32 executable (console) Intel 80386, for MS Windows, UPX compressed
Architecture	: 32 Bits binary
MD5		: 25d562f46c14c5267d56722f6a43b8ed
SHA1		: 7cd4d6f44bdb71d24574d0b4bc326abd006eb510
ssdeep		: 6144:XR/o988gWaraDqRrgwKzBe31i4SgfHysWTx92ltFc87t1VEaj:B+lmRrgwK0lRysiG+Utt
imphash		: 461c5bf3a8c6e0e3c9aec95ff91c5e17
Date		: 0x58D41A71 [Thu Mar 23 18:56:49 2017 UTC]
Language	: ENGLISH
CRC:	(Claimed) : 0x0, (Actual): 0x6155a [SUSPICIOUS]
Entry Point	: 0x480c00 UPX1 1/3 [SUSPICIOUS]
================
Offset | Instructions
----------------------------------------
0	pusha
1	mov esi,0x42f000
6	lea edi,[esi-0x2e000]
12	push edi
13	jmp 0x480c1a
15	nop
16	mov al,[esi]
18	inc esi
19	mov [edi],al
21	inc edi
22	add ebx,ebx
24	jnz 0x480c21
26	mov ebx,[esi]
28	sub esi,0xfffffffc
31	adc ebx,ebx
33	jc 0x480c10
35	mov eax,0x1
40	add ebx,ebx
42	jnz 0x480c33
44	mov ebx,[esi]
46	sub esi,0xfffffffc
49	adc ebx,ebx
51	adc eax,eax
53	add ebx,ebx
55	jnc 0x480c44
57	jnz 0x480c63
59	mov ebx,[esi]
61	sub esi,0xfffffffc
64	adc ebx,ebx
66	jc 0x480c63
68	dec eax
69	add ebx,ebx
71	jnz 0x480c50
73	mov ebx,[esi]
75	sub esi,0xfffffffc
78	adc ebx,ebx
80	adc eax,eax
82	jmp 0x480c28
84	add ebx,ebx
86	jnz 0x480c5f
88	mov ebx,[esi]
90	sub esi,0xfffffffc
93	adc ebx,ebx
95	adc ecx,ecx
97	jmp 0x480cb5
99	xor [eax],eax

Version info
==========================================================================================
LegalCopyright	: Copyright (C) 2017
InternalName	: rainbowmagic.exe
FileVersion	: 1.0.0.1
CompanyName	: Rainbow Magic, Inc.
ProductName	: Rainbow Magic Game.
ProductVersion	: 1.0.0.1
FileDescription	: Ranbow Magic Game.
OriginalFilename: rainbowmagic.exe
Translation	: 0x0409 0x04b0

Sections
==========================================================================================
Name       VirtAddr     VirtSize     RawSize    MD5                              Entropy
------------------------------------------------------------------------------------------
UPX0       0x1000       0x2e000      0x0        d41d8cd98f00b204e9800998ecf8427e 0.000000    [SUSPICIOUS]
UPX1       0x2f000      0x52000      0x52000    16257d5acfdbed7dca880991bf0418ff 7.707761    [SUSPICIOUS]
.rsrc      0x81000      0xa000       0x9c00     567c038240474732cc9dbc92402dd111 4.102480

Resource entries
==========================================================================================
Resource type      Total
-------------------------
BIN                : 1
RT_ICON            : 1
RT_VERSION         : 1
RT_GROUP_ICON      : 1

Imports
==========================================================================================
[1] ADVAPI32.dll
[2] CRYPT32.dll
[3] KERNEL32.DLL
[4] ole32.dll
[5] SHELL32.dll
[6] USER32.dll
[7] WININET.dll

Suspicious IAT alerts
==========================================================================================
[1] GetProcAddress
[2] InternetOpenA
[3] LoadLibraryA
[4] RegCloseKey
[5] ShellExecuteA
[6] VirtualProtect

```

The most interesting things here are the entrypoint `Entry Point : 0x480c00 UPX1 1/3 [SUSPICIOUS]`, and the
Sections, specifically the sizes. The raw size is `0x0` and the VirtSize `0x2e800` is a big difference. This
binary is packed with UPX (the type field also says `UPX compressed`), so let's unpack it

```shell
$ upx -d Unknown -o unknown-decomp
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2013
UPX 3.91        Markus Oberhumer, Laszlo Molnar & John Reiser   Sep 30th 2013

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    433152 <-    376832   87.00%    win32/pe     unknown-decomp

Unpacked 1 file.
```

Now we get into the meaty parts: open the file in r2

```shell
$ r2 unknown-decomp -A
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[ ] [*] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan))
 -- You have been designated for disassembly
[0x00401e9d]>
```

The original article dumps the strings first, so if we use `izz` to dump the strings we see **a lot** of
strings. Of course just scroll through and find the interesting bits, but from the article we
know the string, so we can search for it

```shell
[0x00401e9d]> izz ~Soft
vaddr=0x00412b08 paddr=0x00011108 ordinal=511 sz=46 len=45 section=.rdata type=ascii string=Software\Microsoft\Windows\CurrentVersion\Run
```

> `izz` = search strings in whole binary. `~Soft` search for lines with match _Soft_

Now we have the address of the string, let's find where it's used:

```shell
[0x00401e9d]> axt 0x00412b08
data 0x401399 push str.Software_Microsoft_Windows_CurrentVersion_Run in sub.ADVAPI32.dll_RegOpenKeyExA_340
data 0x4014aa push str.Software_Microsoft_Windows_CurrentVersion_Run in sub.ADVAPI32.dll_RegCreateKeyExA_430
```

So two places, matching with what the original article finds with IDA: `sub.ADVAPI32.dll_RegOpenKeyExA_340`. If we seek
to the location and enter visual mode

```shell
[0x00401e9d]> s 0x401399
[0x04013993]> V
```

We see where the string is being pushed onto the stack. Press `0` to go to the start of the current function and lo
and behold we're at the same address as in the original article:

```shell
/ (fcn) sub.ADVAPI32.dll_RegOpenKeyExA_340 228
|   sub.ADVAPI32.dll_RegOpenKeyExA_340 ();
|           ; var int local_114h @ ebp-0x114
|           ; var int local_110h @ ebp-0x110
|           ; var int local_10ch @ ebp-0x10c
|           ; var int local_108h @ ebp-0x108
|           ; var int local_4h @ ebp-0x4
|              ; CALL XREF from 0x00401618 (main)
|           0x00401340      55             push ebp
```

Right, so the next step is to go to where the function is called, press `x` to bring up the caller selection. There's
only one entry, so press `Enter` and we head straight to `main`. Shortly after the call you can see some more
interesting API calls such as

```
|       |   0x00401652      ff1578e14000   call dword [sym.imp.WININET.dll_InternetOpenA] ;[6] ; 0x40e178 ; ",8\x01"
```

So Malware Unicorn says that we need to look for the value in `esi` because that's the argument that contains the URL it
will connect to. As she says, the value from `eax` is moved into `esi` a few lines before, and `eax` gets populated
as the return value from `call fcn.00401260`. So let's look at what this does. In my session there's a `;[5]` at the end
of the line. This means that if I press `5` it will take me directly to that function. So looking at the preceeding lines
we can see the string being pushed into the stack and possibly its length

```shell
|       |   0x00401635      ba17000000     mov edx, 0x17
|       |   0x0040163a      b9402b4100     mov ecx, str.___343._6_w45.w__36t957 ; 0x412b40 ; ">?<343.?6#w45.w?,36t957"
|       |   0x0040163f      e81cfcffff     call fcn.00401260           ;[5]
```

So we got this function and it's a big one. The original article points out to look for the `xor al, 5Ah` instruction, which
is at `0x00401316`. Scroll there and you can see the xor loop. Or not? Press `V` to see the ASCII graph, which might make the
loop a lot clearer.

So we need to label this function for later use. So let's do that. In visual mode hit `:` to enter command mode and enter

```shell
: afn XorDecode 0x00401260
```

> **a**(nalysis) **f**(unction) **n**(ame) newname address

Where `XorDecode` is the new name and `0x00401260` is the address of the function.

The next part suggests to use `xorsearch` on our binary. This tool also exists on linux and is installed
on Remnux. It has the same options:

```shell
$ xorsearch unknown-decomp ".com"
Found XOR 5A position 11153: .comZdope.exeZZZZdopeZZZZYo this is dope!ZZZZ.**.;
```

Snap.

## Looking deeper

We need to go to the program's main. Easy

```shell
: s main
```

And done. Now, let's follow the rest of the article talking about some of the conditional jumps etc. and
how we are to proceed using the information we got to create a general program flow. From `V`isual mode use `V`
to enter graph mode to help with this and follow Malware Unicorn's original article for that. The workflow
is pretty much the same as in the article. Use the number keys to follow jumps to their location and use `u` to
return to the previous location.

As always, use `?` if you get stuck :)

Thus ends the static analysis.

Thanks [@malwareunicorn](https://twitter.com/malwareunicorn) for the original article!
