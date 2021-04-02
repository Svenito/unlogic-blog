---
extends: default.liquid
title: Binary Bomb with Radare2 - Phase 6
date: 2016-05-27T19:03:50Z
tags:
  - reverse engineering
  - radare
category: reverse engineering
---

Here we are the _last_ phase of our binary bomb. [Phase 5](/2016/05/12/binary-bomb-with-radare2-phase5) was quite interesting, so let's see what this one has in store for us. As per usual, we'll begin with some of the ol static analysis.

## Statanal

Off she goes into radare2, a bit of the old analysis, disassembly, and bask in her glory.

```sourceCode
[0x08048de0]> pdf @ sym.phase_6
/ (fcn) sym.phase_6 249
|           ; arg int arg_8h @ ebp+0x8
|           ; var int local_18h @ ebp-0x18
|           ; var int local_30h @ ebp-0x30
|           ; var int local_34h @ ebp-0x34
|           ; var int local_38h @ ebp-0x38
|           ; var int local_3ch @ ebp-0x3c
|           ; var int local_58h @ ebp-0x58
|           ; CALL XREF from 0x08048b0a (sym.main)
|           0x08048d98      55             push ebp
|           0x08048d99      89e5           mov ebp, esp
|           0x08048d9b      83ec4c         sub esp, 0x4c
|           0x08048d9e      57             push edi
|           0x08048d9f      56             push esi
|           0x08048da0      53             push ebx
|           0x08048da1      8b5508         mov edx, dword [ebp + arg_8h] ; [0x8:4]=0
|           0x08048da4      c745cc6cb204.  mov dword [ebp - local_34h], obj.node1
|           0x08048dab      83c4f8         add esp, -8
|           0x08048dae      8d45e8         lea eax, [ebp - local_18h]
|           0x08048db1      50             push eax
|           0x08048db2      52             push edx
|           0x08048db3      e820020000     call sym.read_six_numbers
|           0x08048db8      31ff           xor edi, edi
|           0x08048dba      83c410         add esp, 0x10
|           0x08048dbd      8d7600         lea esi, [esi]
|           ; JMP XREF from 0x08048e00 (sym.phase_6)
|       .-> 0x08048dc0      8d45e8         lea eax, [ebp - local_18h]
|       |   0x08048dc3      8b04b8         mov eax, dword [eax + edi*4]
|       |   0x08048dc6      48             dec eax
|       |   0x08048dc7      83f805         cmp eax, 5
|      ,==< 0x08048dca      7605           jbe 0x8048dd1
|      ||   0x08048dcc      e82b070000     call sym.explode_bomb
|      ||   ; JMP XREF from 0x08048dca (sym.phase_6)
|      `--> 0x08048dd1      8d5f01         lea ebx, [edi + 1]          ; 0x1 ; "ELF..." @ 0x1
|       |   0x08048dd4      83fb05         cmp ebx, 5
|      ,==< 0x08048dd7      7f23           jg 0x8048dfc
|      ||   0x08048dd9      8d04bd000000.  lea eax, [edi*4]
|      ||   0x08048de0      8945c8         mov dword [ebp - local_38h], eax
|      ||   0x08048de3      8d75e8         lea esi, [ebp - local_18h]
|      ||   ; JMP XREF from 0x08048dfa (sym.phase_6)
|     .---> 0x08048de6      8b55c8         mov edx, dword [ebp - local_38h]
|     |||   0x08048de9      8b0432         mov eax, dword [edx + esi]
|     |||   0x08048dec      3b049e         cmp eax, dword [esi + ebx*4]
|    ,====< 0x08048def      7505           jne 0x8048df6
|    ||||   0x08048df1      e806070000     call sym.explode_bomb
|    ||||   ; JMP XREF from 0x08048def (sym.phase_6)
|    `----> 0x08048df6      43             inc ebx
|     |||   0x08048df7      83fb05         cmp ebx, 5
|     `===< 0x08048dfa      7eea           jle 0x8048de6
|      ||   ; JMP XREF from 0x08048dd7 (sym.phase_6)
|      `--> 0x08048dfc      47             inc edi
|       |   0x08048dfd      83ff05         cmp edi, 5
|       `=< 0x08048e00      7ebe           jle 0x8048dc0
|           0x08048e02      31ff           xor edi, edi
|           0x08048e04      8d4de8         lea ecx, [ebp - local_18h]
|           0x08048e07      8d45d0         lea eax, [ebp - local_30h]
|           0x08048e0a      8945c4         mov dword [ebp - local_3ch], eax
|           0x08048e0d      8d7600         lea esi, [esi]
|           ; JMP XREF from 0x08048e42 (sym.phase_6)
|       .-> 0x08048e10      8b75cc         mov esi, dword [ebp - local_34h]
|       |   0x08048e13      bb01000000     mov ebx, 1
|       |   0x08048e18      8d04bd000000.  lea eax, [edi*4]
|       |   0x08048e1f      89c2           mov edx, eax
|       |   0x08048e21      3b1c08         cmp ebx, dword [eax + ecx]
|      ,==< 0x08048e24      7d12           jge 0x8048e38
|      ||   0x08048e26      8b040a         mov eax, dword [edx + ecx]
|      ||   0x08048e29      8db426000000.  lea esi, [esi]
|      ||   ; JMP XREF from 0x08048e36 (sym.phase_6)
|     .---> 0x08048e30      8b7608         mov esi, dword [esi + 8]    ; [0x8:4]=0
|     |||   0x08048e33      43             inc ebx
|     |||   0x08048e34      39c3           cmp ebx, eax
|     `===< 0x08048e36      7cf8           jl 0x8048e30
|      ||   ; JMP XREF from 0x08048e24 (sym.phase_6)
|      `--> 0x08048e38      8b55c4         mov edx, dword [ebp - local_3ch]
|       |   0x08048e3b      8934ba         mov dword [edx + edi*4], esi
|       |   0x08048e3e      47             inc edi
|       |   0x08048e3f      83ff05         cmp edi, 5
|       `=< 0x08048e42      7ecc           jle 0x8048e10
|           0x08048e44      8b75d0         mov esi, dword [ebp - local_30h]
|           0x08048e47      8975cc         mov dword [ebp - local_34h], esi
|           0x08048e4a      bf01000000     mov edi, 1
|           0x08048e4f      8d55d0         lea edx, [ebp - local_30h]
|           ; JMP XREF from 0x08048e5e (sym.phase_6)
|       .-> 0x08048e52      8b04ba         mov eax, dword [edx + edi*4]
|       |   0x08048e55      894608         mov dword [esi + 8], eax
|       |   0x08048e58      89c6           mov esi, eax
|       |   0x08048e5a      47             inc edi
|       |   0x08048e5b      83ff05         cmp edi, 5
|       `=< 0x08048e5e      7ef2           jle 0x8048e52
|           0x08048e60      c74608000000.  mov dword [esi + 8], 0
|           0x08048e67      8b75cc         mov esi, dword [ebp - local_34h]
|           0x08048e6a      31ff           xor edi, edi
|           0x08048e6c      8d742600       lea esi, [esi]
|           ; JMP XREF from 0x08048e85 (sym.phase_6)
|       .-> 0x08048e70      8b5608         mov edx, dword [esi + 8]    ; [0x8:4]=0
|       |   0x08048e73      8b06           mov eax, dword [esi]
|       |   0x08048e75      3b02           cmp eax, dword [edx]
|      ,==< 0x08048e77      7d05           jge 0x8048e7e
|      ||   0x08048e79      e87e060000     call sym.explode_bomb
|      ||   ; JMP XREF from 0x08048e77 (sym.phase_6)
|      `--> 0x08048e7e      8b7608         mov esi, dword [esi + 8]    ; [0x8:4]=0
|       |   0x08048e81      47             inc edi
|       |   0x08048e82      83ff04         cmp edi, 4
|       `=< 0x08048e85      7ee9           jle 0x8048e70
|           0x08048e87      8d65a8         lea esp, [ebp - local_58h]
|           0x08048e8a      5b             pop ebx
|           0x08048e8b      5e             pop esi
|           0x08048e8c      5f             pop edi
|           0x08048e8d      89ec           mov esp, ebp
|           0x08048e8f      5d             pop ebp
\           0x08048e90      c3             ret
```

Quite the doozy ain't she? We'll just do it bit by bit, and I am sure we'll be fine. Let's work up to the first _jcc_ and go from there. To make this a little easier to visualise, let's try something different: graphviz output. Use `ag sym.phase_6 > phase_6.dot` to generate it, then use `dot -Tpng -O phase_6.dot` to create the png.

[<img src="/images/phase_6.dot.png" alt="phase_6 graph" height="600" />](/images/phase_6.dot.png)

Neat eh? So let's break that up into smaller blocks and examine each one in turn.

```sourceCode
|           0x08048da1      8b5508         mov edx, dword [ebp + arg_8h] ; [0x8:4]=0
|           0x08048da4      c745cc6cb204.  mov dword [ebp - local_34h], obj.node1
|           0x08048dab      83c4f8         add esp, -8
|           0x08048dae      8d45e8         lea eax, [ebp - local_18h]
|           0x08048db1      50             push eax
|           0x08048db2      52             push edx
|           0x08048db3      e820020000     call sym.read_six_numbers
|           0x08048db8      31ff           xor edi, edi
|           0x08048dba      83c410         add esp, 0x10
|           0x08048dbd      8d7600         lea esi, [esi]
|           ; JMP XREF from 0x08048e00 (sym.phase_6)
|       .-> 0x08048dc0      8d45e8         lea eax, [ebp - local_18h]
|       |   0x08048dc3      8b04b8         mov eax, dword [eax + edi*4]
|       |   0x08048dc6      48             dec eax
|       |   0x08048dc7      83f805         cmp eax, 5
|      ,==< 0x08048dca      7605           jbe 0x8048dd1
```

The usual function prologue passes and we open on our input string's address being stored in `edx`.

Next up is something new: `obj1.node1`. What could this be? Let's take a look with `px`

```sourceCode
:> px @ obj.node1
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  0123456789ABCD
0x0804b26c  fd00 0000 0100 0000 60b2 0408 e903  ........`.....
0x0804b27a  0000 0000 0000 0000 0000 2f00 0000  ........../...
0x0804b288  0000 0000 0000 0000 1400 0000 0000  ..............
0x0804b296  0000 0000 0000 0700 0000 0000 0000  ..............
0x0804b2a4  0000 0000 2300 0000 0000 0000 0000  ....#.........
0x0804b2b2  0000 6300 0000 0000 0000 0000 0000  ..c...........
0x0804b2c0  0100 0000 0000 0000 0000 0000 2800  ............(.
0x0804b2ce  0000 0000 0000 0000 0000 6b00 0000  ..........k...
0x0804b2dc  b4b2 0408 78b2 0408 0600 0000 c0b2  ....x.........
0x0804b2ea  0408 9cb2 0408 2d00 0000 ccb2 0408  ......-.......
```

So we have two integers: _253_ (_0xfd_) and _1_, followed by what is likely an address. Let's check that too

```sourceCode
:> px @ 0x0804b260
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  0123456789ABCD
0x0804b260  d502 0000 0200 0000 54b2 0408 fd00  ........T.....
0x0804b26e  0000 0100 0000 60b2 0408 e903 0000  ......`......
```

Same thing again. I'd hazard a guess that this a node in a linked list of some sort and has the following structure:

```sourceCode
struct node {
    int value;
    int index;
    node *next;
};
```

Let's see how long this list is, and what's in it. For this we'll use the `pf` command.

```sourceCode
[0x0072f850]> pf.node ii*? value index (node)next
[0x0072f850]> pf.node @ obj.node1
     value : 0x0804b26c = 253
     index : 0x0804b270 = 1
      next : (*0x804b260)
    struct<node>
     value : 0x0804b260 = 725
     index : 0x0804b264 = 2
      next : (*0x804b254)
    struct<node>
     value : 0x0804b254 = 301
     index : 0x0804b258 = 3
      next : (*0x804b248)
    struct<node>
     value : 0x0804b248 = 997
     index : 0x0804b24c = 4
      next : (*0x804b23c)
    struct<node>
     value : 0x0804b23c = 212
     index : 0x0804b240 = 5
      next : (*0x804b230)
    struct<node>
     value : 0x0804b230 = 432
     index : 0x0804b234 = 6
      next : (*0x0) NULL
```

This is one of the reasons I did this writeup, to learn more about using Radare, and this one of those things. What's going on here? `pf` _prints formatted data_, use `pf?` to take a look at all the options. The `ii*?` defines the structure as being two ints, followed by a pointer to a structure. To see all the possible types use `pf??`. Therefore we define a structure called `node` that has two ints and a pointer to a struct. Then we give those things a name `value index (node)next`. The `(node)next` tells it that `next` is a pointer to another node type. This will now run until `next` is NULL, and thus, we have the complete list. I ♥ radare.

So take note: `[ebp - local_34h]` points to the first node of the list.

The address of the local variable `ebp - local_18h` is loaded into `eax`. This is then pushed onto the stack along with `edx` forming the arguments to the already familiar function _sym.read_six_numbers_. Once this call returns we'll have the six numbers in `eax`. Next up `edi` is zeroed out. Could this be a loop counter? Given the number of back jumps in this function, it's very likely we're in at least one loop. If you take a peek at `0x8048df6` (line 38 in the long listing above), you can see it gets incremented, tested and followed by a jcc. Yup, it's our loop counter in that case. The next interesting bit is at line 12 where the address of that local variable is loaded into `eax`. This points `eax` to the start of the six numbers. Then we see our old buddy the array index again. Line 13 translates to `six_numbers[loop]`, and that value gets stored in `eax`. This value is then decreased, compared to 5, and if it's below or equal, we jump to `0x8048dd1`. If we don't take it, the bomb assplodes, so let's see what happens at the jump destination.

## Loop 1

```sourceCode
|       ^   0x08048dd1      8d5f01         lea ebx, [edi + 1]          ; 0x1 ; "ELF..." @ 0x1
|       |   0x08048dd4      83fb05         cmp ebx, 5
|      ,==< 0x08048dd7      7f23           jg 0x8048dfc                ;[3]
|      ||   0x08048dd9      8d04bd000000.  lea eax, [edi*4]
|      ||   0x08048de0      8945c8         mov dword [ebp - local_38h], eax
|      ||   0x08048de3      8d75e8         lea esi, [ebp - local_18h]
|     .---> 0x08048de6      8b55c8         mov edx, dword [ebp - local_38h]
|     |||   0x08048de9      8b0432         mov eax, dword [edx + esi]
|     |||   0x08048dec      3b049e         cmp eax, dword [esi + ebx*4]
|    ,====< 0x08048def      7505           jne 0x8048df6               ;[4]
|    ||||   0x08048df1      e806070000     call sym.explode_bomb       ;[2]
|    `----> 0x08048df6      43             inc ebx
|     |||   0x08048df7      83fb05         cmp ebx, 5
|     `===< 0x08048dfa      7eea           jle 0x8048de6               ;[5]
|      `--> 0x08048dfc      47             inc edi
|       |   0x08048dfd      83ff05         cmp edi, 5
|       `=< 0x08048e00      7ebe           jle 0x8048dc0               ;[6]
```

This block is a loop in itself, with a few conditional jumps. After the last conditional jump another block starts, we'll look at that later. First we store `loop + 1` in `ebx` and compare that to _5_. I it's greater than _5_ we jump to line 15 where the loop counter itself is increased, compared to 5, and if less than _5_, jumps back to `0x08048dc0`.

Let's look at what happens if `loop + 1` is less or equal to 5 on line 3. We don't take the jump and we store `loop * 4` in `eax` This is then is moved into a local variable. The next instruction makes `esi` point to the start of the 6 numbers array.

Now a second, nested, loop begins at line 8. The loop variable for this loop is `ebx`, as can be determined by lines 15-17. This chunk of code is a little convoluted and will probably be more confusing if I write it out in words here, so let's reimplement it. At this stage you should be able to identify the instructions and follow along.

```sourceCode
for (i = 0; i <= 5; i++) {
    int curr_num = nums[i];
    curr_num--;
    if (curr_num > 5) {
        explode_bomb();
    }

    // Here is where the above block starts. The inner loop
    // counter is set up
    int ebx = i + 1;
    if (ebx > 5) {
        continue;
    }

    int tmp_eax = i * 4;
    int \*esi = nums;

    for (; ebx <= 5; ebx++) {
        edx = tmp_eax;
        eax = esi[edx];
        if (eax == esi[ebx]) {
            explode_bomb();
        }
     }
 }
```

So this is what we are dealing with. We loop over all the numbers, and for each number we check that none of the numbers that follow the current one are the same.

So our first requirements are:

- Must be six numbers
- None of the numbers must be greater than 6
- None of the numbers should be repeated in the sequence

## Loop 2

```sourceCode
|           0x08048e02      31ff           xor edi, edi
|           0x08048e04      8d4de8         lea ecx, [ebp - local_18h]
|           0x08048e07      8d45d0         lea eax, [ebp - local_30h]
|           0x08048e0a      8945c4         mov dword [ebp - local_3ch], eax
|           0x08048e0d      8d7600         lea esi, [esi]
|           ; JMP XREF from 0x08048e42 (sym.phase_6)
|       .-> 0x08048e10      8b75cc         mov esi, dword [ebp - local_34h]
|       |   0x08048e13      bb01000000     mov ebx, 1
|       |   0x08048e18      8d04bd000000.  lea eax, [edi*4]
|       |   0x08048e1f      89c2           mov edx, eax
|       |   0x08048e21      3b1c08         cmp ebx, dword [eax + ecx]
|      ,==< 0x08048e24      7d12           jge 0x8048e38               ;[6]
|      ||   0x08048e26      8b040a         mov eax, dword [edx + ecx]
|      ||   0x08048e29      8db426000000.  lea esi, [esi]
|      ||   ; JMP XREF from 0x08048e36 (sym.phase_6)
|     .---> 0x08048e30      8b7608         mov esi, dword [esi + 8]    ; [0x8:4]=0
|     |||   0x08048e33      43             inc ebx
|     |||   0x08048e34      39c3           cmp ebx, eax
|     `===< 0x08048e36      7cf8           jl 0x8048e30                ;[7]
|      ||   ; JMP XREF from 0x08048e24 (sym.phase_6)
|      `--> 0x08048e38      8b55c4         mov edx, dword [ebp - local_3ch]
|       |   0x08048e3b      8934ba         mov dword [edx + edi*4], esi
|       |   0x08048e3e      47             inc edi
|       |   0x08048e3f      83ff05         cmp edi, 5
|       `=< 0x08048e42      7ecc           jle 0x8048e10
```

Before we enter the second loop there's some activity we should look at. First up `edi` is zeroed out... so it's likely the loop counter again. A quick check at the end of the loop confirms this. So next up some local vars are stored in `ecx` and `eax`, the latter is then moved into a new local variable, `local_3ch`. The first local variable points to the start of the numbers (`local_18h`). After that it looks like `eax` becomes a pointer to `local_30h` and is then dereferenced into `local_3ch`.

Now the outerloop begins, and the first thing to happen is that `esi` now points to that list we looked at earlier. So this could get a little on the thinky side now. Nonetheless, we shall not let this thwart us, onwards to setting `ebx` to _1_, loop counter initialised. `eax` will take the loop counter multiplied by _4_, so will most likely be used as in index into the numbers array. The same value is also placed into `edx`. Then our loop counter is compared to _numbers\[loop\]_ and if it's greater or equal, it takes a jump. Let's start with not taking the jump and see what happens there.

First `eax` gets the current number. Then get a handle on our structureand then get `esi + 8`. That's the _next_ pointer because it's 8 bytes from the start of the structure. Then `ebx` is increased and compared to `eax` again. If it's less than `eax`, we jump up and run the inner loop again.

So what does this do? It will iterate over the list `eax` times, moving along the linked list each time. When this loop exits, `esi` will point to whatever node was set on line 16. Then the node is added to an array, and the outer loop restarts. This will keep getting the `next` object until we've looped _current number_ of times. In the end we'll end up with the objects arranged according to the values of the numbers in our input. Assume that our input was _3 1 6 2 5 4_, the result will be

> obj3, obj1, obj6, obj2, obj5, obj4

At the end of the inner loop `edx` will point to `[ebp - local_3ch]` which is an array storing `edi`, which is the pointer to the relevant object.

A good guess now would be that we need to arrange the nodes in a specific order.

## Loop 3

```sourceCode
|           0x08048e44      8b75d0         mov esi, dword [ebp - local_30h]
|           0x08048e47      8975cc         mov dword [ebp - local_34h], esi
|           0x08048e4a      bf01000000     mov edi, 1
|           0x08048e4f      8d55d0         lea edx, [ebp - local_30h]
|           ; JMP XREF from 0x08048e5e (sym.phase_6)
|       .-> 0x08048e52      8b04ba         mov eax, dword [edx + edi*4]
|       |   0x08048e55      894608         mov dword [esi + 8], eax
|       |   0x08048e58      89c6           mov esi, eax
|       |   0x08048e5a      47             inc edi
|       |   0x08048e5b      83ff05         cmp edi, 5
|       `=< 0x08048e5e      7ef2           jle 0x8048e52               ;[2]
```

Before the loop starts `esi` is set to the first element of the newly ordered list, and then its assigned as the first element of the origin obj list. `edi` is set to one, marking the starting index of the loop, followed by `edx` pointing to the first element of the ordered list. Now, enter the loop.

`eax` gets the value of _index_ of the first object, and then this is assigned to the _next_ pointer of the object. In short, this updates the next pointers in the list of objects.

## Recap

A lot has happened in these three loops, so let's go over it quickly

- the first loops checks our input values
- the second loops arranges the object by thei _index_ value according to the list of numbers we supplied
- the third loop updates the _next_ pointer for each object so that it's correct for the new order.

Ok, so we've boiled that complicated set of loops down to three atomic actions. We now know what our input is used for, what the values of the objects are. Now we need to figure out what the bomb wants from us. This will be the last bit

## Not exploding

```sourceCode
|           0x08048e60      c74608000000.  mov dword [esi + 8], 0
|           0x08048e67      8b75cc         mov esi, dword [ebp - local_34h]
|           0x08048e6a      31ff           xor edi, edi
|           0x08048e6c      8d742600       lea esi, [esi]
|           ; JMP XREF from 0x08048e85 (sym.phase_6)
|       .-> 0x08048e70      8b5608         mov edx, dword [esi + 8]    ; [0x8:4]=0
|       |   0x08048e73      8b06           mov eax, dword [esi]
|       |   0x08048e75      3b02           cmp eax, dword [edx]
|      ,==< 0x08048e77      7d05           jge 0x8048e7e               ;[4]
|      ||   0x08048e79      e87e060000     call sym.explode_bomb       ;[5]
|      ||   ; JMP XREF from 0x08048e77 (sym.phase_6)
|      `--> 0x08048e7e      8b7608         mov esi, dword [esi + 8]    ; [0x8:4]=0
|       |   0x08048e81      47             inc edi
|       |   0x08048e82      83ff04         cmp edi, 4
|       `=< 0x08048e85      7ee9           jle 0x8048e70               ;[6]
|           0x08048e87      8d65a8         lea esp, [ebp - local_58h]
```

The presence of the `call sym.explode_bomb` indicates that this is our _do or die_ bit, where our input choice is validated. The key instruction being on line 8, where the compare instruction happens. Working backwards from there, let's see what `eax` and `edx` are at this point. By looking at the preceeding lines, we can tell that `eax` holds a pointer to the current object (line 7) and `edx` a pointer to the next object (line 6). The compare operation compares the values of the current and _next_ object's value field to make sure that the list's `object.value` fields are in decending order.

Ok, so let's look at the original list again:

```sourceCode
[0x0072f850]> pf.node ii*? value index (node)next
[0x0072f850]> pf.node @ obj.node1
     value : 0x0804b26c = 253
     index : 0x0804b270 = 1
      next : (*0x804b260)
    struct<node>
     value : 0x0804b260 = 725
     index : 0x0804b264 = 2
      next : (*0x804b254)
    struct<node>
     value : 0x0804b254 = 301
     index : 0x0804b258 = 3
      next : (*0x804b248)
    struct<node>
     value : 0x0804b248 = 997
     index : 0x0804b24c = 4
      next : (*0x804b23c)
    struct<node>
     value : 0x0804b23c = 212
     index : 0x0804b240 = 5
      next : (*0x804b230)
    struct<node>
     value : 0x0804b230 = 432
     index : 0x0804b234 = 6
      next : (*0x0) NULL
```

So in order to get this list in the right order we need to pass:

> 4 2 6 3 1 5

Let's try:

```sourceCode
$ ./bomb defuse
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
Good work!  On to the next...
4 2 6 3 1 5
Congratulations! You've defused the bomb!
```

## End of the game?

That's the (official) end of the bomb. We know however that there's a secret level too. In fact, there's two: _secret_phase_ and _fun7_. We'll look at these two together in the next post.
