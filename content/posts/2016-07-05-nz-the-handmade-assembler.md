---
title: nz the handmade assembler
date: 2016-07-04T19:03:50Z
tags:
  - radare
  - programming
category: programming
---

It's been a little quiet lately in this corner of the intertubes because I've been working on refactoring the nz (handmade) assembler for radare2. The original implementation, while effective, was a long list of **if** statements and quite a bit of duplicated code. In addition to this there's also quite a few assemblers that come with radare2, and another intention of the refactor was to reduce this number a little bit. One idea was to merge the tab assembler with nz, creating a hackable tab assembler.

My idea was to create a lookup table that will index assembly functions for each instruction. I've merged in the instruction parsing functions from the tab assembler, as they were nice and clean, and tweaked the structures a little, to allow me to pass all the information I need to assemble an instruction to each function.

Instructions that have no arguments, and thus fixed opcodes, would be defined in the lookup table directly. In addition I was also able to abstract out the _group 1_ and _group 2_ instructions into generic functions, greatly reducing the number of required functions.

So far I've managed to get over 100 instructions supported, some of which weren't in the original nz assembler or tab assembler. I'm quite happy with how it works, and I feel that it is a lot more approachable and readable than what nz was before. It is also more flexible and easier to extend than the tab assembler.

Hopefully it will make it into the next release (coming in the next few days) and you can take a look and perhaps add some instructions yourself.

Otherwise, you can find the code on my [Github](https://github.com/Svenito)
