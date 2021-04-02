---
extends: default.liquid
title: Binary Bomb with Radare2 - Prelude
date: 2016-04-12T21:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

It's reverse engineering time here at Unlogic HQ. Time to dust off the cobwebs that have settled on those parts of my brain and reactivate those dormant synapses.

I've had an itch to write this series of posts for some time. It feels like now is the right time to do this. This will be a series which will walk through solving a binary bomb using R2 (Radare2). If you are not familiar with a binary bomb, it's a small (27K in this case) binary that contains a number of stages which prompt your for input to "defuse" it. If you get it wrong, the bomb "blows up" (exits). If you get it right, you pass onto the next stage. The idea is for you to reverse the binary in order to determine the right input for each stage. You can do this either statically or dynamically, and I think I'll cover both ways at some point in this series. The idea is to figure out the correct input from the disassembly, rather than patching the binary.

I obtained the binary some time ago, and cannot quite remember where I downloaded it from. It seems to originate from a lab exercise from Carnegie Mellon University, which has been modified to run outside the lab environment.

The plan is to solve one stage per post on a weekly/bi-weekly basis, depending on how much time I can dedicate to this. **SPOILER:** there are 7 phases. I'm no expert reverser, and this series will also serve as a refresher and learning exercise for me.

If you want to follow along at home, here are the links to the binary (Linux only) and Radare:

- [download binary bomb](https://unlogic.co.uk/bomb.tar.bz2)
- [download radare2](https://github.com/radare/radare2)

These posts don't assume you are overly familiar with Radare, but have at least used it before. The commands that are used will not always be explained in depth. It is advisable that you read the [R2 Book](https://radare.gitbooks.io/radare2book/content/), or other resources from the [Radare doc pages](https://radare.org/r/docs.html). Remember that you can always get help on commands with `?` from the radare prompt.

Right then, let's open her up...

## Recon

We can't just dive in before we know a little more about this binary. Let's get to know her first. Take her out to dinner and ask all those pertinant questions: _"What's your filetype?"_, _"How many stages have you got?"_, _"What's your favourite drink?"_, _"Are you stripped?"_.

We'll use _rabin2_ for this part of the exercise

### File infromation

```sourceCode
$> rabin2 -I bomb
havecode true
pic      false
canary   false
nx       false
crypto   false
va       true
intrp    /lib/ld-linux.so.2
bintype  elf
class    ELF32
lang     c
arch     x86
bits     32
machine  Intel 80386
os       linux
minopsz  1
maxopsz  16
pcalign  0
subsys   linux
endian   little
stripped false
static   false
linenum  true
lsyms    true
relocs   true
rpath    NONE
binsz    26943
```

So we've got ourselves a 32bit unstripped binary written in C. Cool, nice and straightforward. Let's look at what is linked in dynamically

### Dynamic libs

```sourceCode
$> rabin2 -l bomb
[Linked libraries]
libc.so.6

1 library
```

Nothing exotic, so less external stuff to worry about. But what are we going to import from these libs?

### Symbols

```sourceCode
$> rabin2 -i bomb
[Imports]
ordinal=001 plt=0x08048720 bind=UNKNOWN type=FUNC name=__register_frame_info
ordinal=002 plt=0x08048730 bind=GLOBAL type=FUNC name=close
ordinal=003 plt=0x08048740 bind=GLOBAL type=FUNC name=fprintf
ordinal=004 plt=0x08048750 bind=GLOBAL type=FUNC name=tmpfile
ordinal=005 plt=0x08048760 bind=GLOBAL type=FUNC name=getenv
ordinal=006 plt=0x08048770 bind=GLOBAL type=FUNC name=signal
ordinal=007 plt=0x08048780 bind=GLOBAL type=FUNC name=fflush
ordinal=008 plt=0x08048790 bind=GLOBAL type=FUNC name=bcopy
ordinal=009 plt=0x080487a0 bind=GLOBAL type=FUNC name=rewind
ordinal=010 plt=0x080487b0 bind=GLOBAL type=FUNC name=system
ordinal=011 plt=0x080487c0 bind=UNKNOWN type=FUNC name=__deregister_frame_info
ordinal=013 plt=0x080487d0 bind=GLOBAL type=FUNC name=fgets
ordinal=014 plt=0x080487e0 bind=GLOBAL type=FUNC name=sleep
ordinal=015 plt=0x080487f0 bind=GLOBAL type=FUNC name=__strtol_internal
ordinal=016 plt=0x08048800 bind=GLOBAL type=FUNC name=__libc_start_main
ordinal=017 plt=0x08048810 bind=GLOBAL type=FUNC name=printf
ordinal=018 plt=0x08048820 bind=GLOBAL type=FUNC name=fclose
ordinal=019 plt=0x08048830 bind=GLOBAL type=FUNC name=gethostbyname
ordinal=020 plt=0x08048840 bind=GLOBAL type=FUNC name=bzero
ordinal=021 plt=0x08048850 bind=GLOBAL type=FUNC name=exit
ordinal=022 plt=0x08048860 bind=GLOBAL type=FUNC name=sscanf
ordinal=024 plt=0x08048870 bind=GLOBAL type=FUNC name=connect
ordinal=026 plt=0x08048880 bind=GLOBAL type=FUNC name=fopen
ordinal=027 plt=0x08048890 bind=GLOBAL type=FUNC name=dup
ordinal=029 plt=0x080488a0 bind=GLOBAL type=FUNC name=sprintf
ordinal=030 plt=0x080488b0 bind=GLOBAL type=FUNC name=socket
ordinal=031 plt=0x080488c0 bind=GLOBAL type=FUNC name=cuserid
ordinal=032 plt=0xffffffff bind=UNKNOWN type=NOTYPE name=__gmon_start__
ordinal=033 plt=0x080488d0 bind=GLOBAL type=FUNC name=strcpy

29 imports
```

Also nothing crazy, mostly stuff for input and output. Right, enough foreplay, let's fire up radare.

> **note**
>
> We could also have used various linux tools to analyse the binary. So if you
> fancy an exercise, feel free to play around with `ldd`, `file`, `nm`, et al.

## Phase 0

_Getting to know sym.main_

So let's load her up into _r2_

```sourceCode
$ r2 bomb
 -- getdruid to get eclectic uid
[0x080488e0]>
```

Next we want to analyse the code, as we want to get all the xrefs, strings, and functions. Otherwise you will get _Cannot find function at 0x12345678_ type errors later on.

```sourceCode
[0x080488e0]> aa
[x] Analyze all flags starting with sym. and entry0 (aa)
[0x080488e0]>
```

Ok, now let's take a look at the functions or symbols in the binary. Using `afl` we get the following

```sourceCode
[0x080488e0]> afl
0x080486e0  47    3     sym._init
0x08048720  16    2     sym.imp.__register_frame_info
0x08048730  16    2     sym.imp.close
0x08048740  16    2     sym.imp.fprintf
0x08048750  16    2     sym.imp.tmpfile
0x08048760  16    2     sym.imp.getenv
0x08048770  16    2     sym.imp.signal
0x08048780  16    2     sym.imp.fflush
0x08048790  16    2     sym.imp.bcopy
0x080487a0  16    2     sym.imp.rewind
0x080487b0  16    2     sym.imp.system
0x080487c0  16    2     sym.imp.__deregister_frame_info
0x080487d0  16    2     sym.imp.fgets
0x080487e0  16    2     sym.imp.sleep
0x080487f0  16    2     sym.imp.__strtol_internal
0x08048800  16    2     sym.imp.__libc_start_main
0x08048810  16    2     sym.imp.printf
0x08048820  16    2     sym.imp.fclose
0x08048830  16    2     sym.imp.gethostbyname
0x08048840  16    2     sym.imp.bzero
0x08048850  16    2     sym.imp.exit
0x08048860  16    2     sym.imp.sscanf
0x08048870  16    2     sym.imp.connect
0x08048880  16    2     sym.imp.fopen
0x08048890  16    2     sym.imp.dup
0x080488a0  16    2     sym.imp.sprintf
0x080488b0  16    2     sym.imp.socket
0x080488c0  16    2     sym.imp.cuserid
0x080488d0  16    2     sym.imp.strcpy
0x080488e0  34    1     entry0
0x08048910  81    8     sym.__do_global_dtors_aux
0x08048964  10    1     sym.fini_dummy
0x08048970  37    3     sym.frame_dummy
0x08048998  10    1     sym.init_dummy
0x080489b0  365   7     sym.main
0x08048b20  39    3     sym.phase_1
0x08048b48  79    7     sym.phase_2
0x08048b98  264   7     sym.phase_3
0x08048ca0  62    4     sym.func4
0x08048ce0  75    6     sym.phase_4
0x08048d2c  105   7     sym.phase_5
0x08048d98  249   21    sym.phase_6
0x08048e94  82    8     sym.fun7
0x08048ee8  96    5     sym.secret_phase
0x08048f50  98    1     sym.sig_handler
0x08048fb4  33    1     sym.invalid_phase
0x08048fd8  61    3     sym.read_six_numbers
0x08049018  24    3     sym.string_length
0x08049030  89    8     sym.strings_not_equal
0x0804908c  212   7     sym.open_clientfd
0x08049160  25    1     sym.initialize_bomb
0x0804917c  50    7     sym.blank_line
0x080491b0  74    4     sym.skip
0x080491fc  193   9     sym.read_line
0x080492c0  570   22    sym.send_msg
0x080494fc  45    1     sym.explode_bomb
0x0804952c  122   6     sym.phase_defused
0x080495b0  38    3     sym.__do_global_ctors_aux
0x080495d8  10    1     sym.init_dummy_1
0x080495e4  26    1     obj._etext
```

Here, amongst other things, we can see the \_phase\__\* functions which are the various stages of the bomb. Take a look at the rest of the list and make a note of anything you might find interesting. We could start by looking at the disassembly of `phase_1` and figure out what it requires, but why not start right at the beginning: \_sym.main_

In order to do this, having analysed the symbols already, we can just print the disassembly with `pdf @ main`. Had we not run `aa` earlier, we'd be seeing the _not found_ error at this point. This command will print the disassembly of the _main_ function, and we can get a good idea of what happens at the start of the application, before we look at the phases.

```
[0x080488e0]> pdf @ main
            ;-- main:
            ;-- gcc2_compiled._4:
/ (fcn) sym.main 365
|           ; arg int arg_8h @ ebp+0x8
|           ; arg int arg_ch @ ebp+0xc
|           ; var int local_18h @ ebp-0x18
|           ; DATA XREF from 0x080488f7 (sym.main)
|           0x080489b0      55             push ebp
|           0x080489b1      89e5           mov ebp, esp
|           0x080489b3      83ec14         sub esp, 0x14
|           0x080489b6      53             push ebx
|           0x080489b7      8b4508         mov eax, dword [ebp+arg_8h] ; [0x8:4]=-1 ; 8 ; bomb.c:36
|           0x080489ba      8b5d0c         mov ebx, dword [ebp+arg_ch] ; [0xc:4]=-1 ; 12
|           0x080489bd      83f801         cmp eax, 1                  ; eax ; bomb.c:44
|       ,=< 0x080489c0      750e           jne 0x80489d0               ;[1]
|       |   0x080489c2      a148b60408     mov eax, dword [obj.stdin__GLIBC_2.0] ; [0x804b648:4]=0xf7766c20 LEA obj.stdin__GLIBC_2.0 ; " lv....." @ 0x804b648 ; bomb.c:45
|       |   0x080489c7      a364b60408     mov dword [obj.infile], eax ; [0x804b664:4]=0 LEA obj.infile ; obj.infile
|      ,==< 0x080489cc      eb62           jmp 0x8048a30               ;[2] ; bomb.c:46
|      ||   0x080489ce      89f6           mov esi, esi
|      |`-> 0x080489d0      83f802         cmp eax, 2                  ; 2 ; bomb.c:52
|      |,=< 0x080489d3      753b           jne 0x8048a10               ;[3]
|      ||   0x080489d5      83c4f8         add esp, -8                 ; bomb.c:53
|      ||   0x080489d8      6820960408     push 0x8049620
|      ||   0x080489dd      8b4304         mov eax, dword [ebx + 4]    ; [0x4:4]=-1 ; 4
|      ||   0x080489e0      50             push eax
|      ||   0x080489e1      e89afeffff     call sym.imp.fopen          ;[4]
|      ||   0x080489e6      a364b60408     mov dword [obj.infile], eax ; [0x804b664:4]=0 LEA obj.infile ; obj.infile
|      ||   0x080489eb      83c410         add esp, 0x10
|      ||   0x080489ee      85c0           test eax, eax
|     ,===< 0x080489f0      753e           jne 0x8048a30               ;[2]
|     |||   0x080489f2      83c4fc         add esp, -4                 ; bomb.c:54
|     |||   0x080489f5      8b4304         mov eax, dword [ebx + 4]    ; [0x4:4]=-1 ; 4
|     |||   0x080489f8      50             push eax
|     |||   0x080489f9      8b03           mov eax, dword [ebx]
|     |||   0x080489fb      50             push eax
|     |||   0x080489fc      6822960408     push str._s:_Error:_Couldn_t_open__s_n ; str._s:_Error:_Couldn_t_open__s_n ; "%s: Error: Couldn't open %s." @ 0x8049622
|     |||   0x08048a01      e80afeffff     call sym.imp.printf         ;[5]
|     |||   0x08048a06      83c4f4         add esp, -0xc               ; bomb.c:55
|     |||   0x08048a09      6a08           push 8                      ; 8
|     |||   0x08048a0b      e840feffff     call sym.imp.exit           ;[6]
|     ||`-> 0x08048a10      83c4f8         add esp, -8                 ; bomb.c:61
|     ||    0x08048a13      8b03           mov eax, dword [ebx]
|     ||    0x08048a15      50             push eax
|     ||    0x08048a16      683f960408     push str.Usage:__s___input_file___n ; str.Usage:__s___input_file___n ; "Usage: %s [<input_file>]." @ 0x804963f
|     ||    0x08048a1b      e8f0fdffff     call sym.imp.printf         ;[5]
|     ||    0x08048a20      83c4f4         add esp, -0xc               ; bomb.c:62
|     ||    0x08048a23      6a08           push 8                      ; 8
|     ||    0x08048a25      e826feffff     call sym.imp.exit           ;[6]
|     ||    0x08048a2a      8db600000000   lea esi, [esi]
|     ||    ; JMP XREF from 0x080489cc (sym.main)
|     ``--> 0x08048a30      e82b070000     call sym.initialize_bomb    ;[7] ; bomb.c:66
.. snip ..
```

This is the entry point of the binary, _sym.main_, truncated for clarity. Lines 9-12 are the function prologue. After this the function arguments are processed. Typically a main function will look like this: `int main(int argc, char* argv[])` and in lines 13 and 14 `argc` is moved into `eax` and `argv` into `ebx`. Line 15 compares the value in `eax` to 1. Line 16 is a conditional jump that jumps if `ebx` is not equal to 1. If `eax` is 1 (i.e. no command line arguments were supplied) it initialises _stdin_ and then jumps to `call sym.initialize_bomb`. This, without investigating further, is most likely where the game begins. We now know that we can pass our answers into the binary via _stdin_ as well as playing interactively.

So what happens if the number of arguments is not 1? In that case the branch is taken and we end up at `0x080489d0` (line 21) where we compare `eax` with 2. If not equal to 2 the conditional branch takes us to `0x08048a10` (line 42). Here it will print out a _usage_ message, clean up, and exit the program.

If the number of arguments is 2 then it will try to open a file with the name of the second argument. If it succeeds to open the file, it jumps to `0x8048a30` (line 52), which we know is _sym.initialize_bomb_. Otherwise, if it fails to open the file, an error message is printed and the program exits.

So it looks like we can also put our solutions for each level into a file and pass that into the binary. This will make it much easier to replay the binary for each level we attempt.

We now know enough about the launch of the binary to start looking at the first phase. We'll cover phase 1, and a little of the end of _sym.main_, in the next post.

## Graphs!!!

And because I love the feature so much, here's the ASCII callgraph of _sym.main_. Zoomed out, so a some of the code is truncated in the boxes

[<img src="https://i.imgur.com/CUh49dw.png" alt="graphs! Freaking graphs!" width="500" />](https://i.imgur.com/CUh49dw.png)

You can view this yourself with the following commands (comments after \#)

```sourceCode
[0x080489b0]> s main        # seek to main
[0x080489b0]> V             # enter visual mode
*press V*                   # from visual mode enter graph mode
```

You can clearly see the branch points and the _true/false_ branches from each of the boxes. So if you get stuck following the disassembly, the graph view is a big help.

Finally here's the [asciinema](https://asciinema.org/) recording of the above

<script type="text/javascript" src="https://asciinema.org/a/42214.js" id="asciicast-42214" async></script>

The embedded player doesn't have monospaced fonts, so if you want to see the graph properly, [download the recording](https://asciinema.org/a/42214.json) and play it locally.

There's a little extra in the recording, which is the bit where I scroll through the disassembly. To get there seek to _main_ with `s main` then enter _visual mode_ with `V` and when you see the hexdump view, press `p` until you get that view. Use _hjkl_ or the arrow keys to navigate, and as always, `?` for help.

> **note**
>
> On the original functionality of bomb
>
> If you've looked at the strings inside the binary you've probably noticed a list of hostnames. As far as I can tell the bomb communicated with these servers to register success/fails of the students and each user was only allowed to blow up so many times. I would assume that how often they "blew up" was taken into account when grading. Although the remenants of this functionality are still in the code, it is no longer operational, and we can use the bomb offline, without worrying about being locked out or anything.

[Read Phase 1](https://unlogic.co.uk/2016/04/14/binary-bomb-with-radare2-phase-1/)
