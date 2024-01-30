---
layout: default
title: "mildly-cursed"
date: 2024-01-30 04:00:00 -0000
tags: vtmb console cursed python
---

Intermission time: cursed things in VtmB. One is about Python. Future ones will also feature the game's embedded Python engine in some form or another. Did you know that the console is rigged to also be a Python REPL?

![](/breaking-videogames/assets/cursedparser.jpg)

Invalid commands get interpreted as Python code and executed. Today's first mildly cursed thing: a console that behaves as one of two different consoles, depending on your input.

After I captured this, I went on to load a level, and encountered an unexpected crash.

```
* * * Vampire Crash Data Log, Generated From Exe Built On: Oct  6 2004 * * *

vampire caused an Access Violation (0xc0000005) 
in module engine.dll at 0023:20095ef1.

Read from location 00000000 caused an access violation.

Context:
EDI:    0x0522d33d  ESI: 0x05227fd8  EAX:   0x0018e3f0
EBX:    0x00000000  ECX: 0x00000000  EDX:   0x00c79d88
[...]
```

Initially I thought it was a simple matter of running out of memory to allocate, since I had a couple other memory-hungry things running, but closing them didn't alleviate it. Taking a quick look around turned out to be more informative than I expected. The crash location:

![](/breaking-videogames/assets/crashloc.jpg)

`ecx` is supposed to hold a pointer, but it's zero in our crash log. `eip` is consistent between runs. Luckily, this pointer's location (`0x21300c88`) is also static, and it's assigned in two places in the entirety of `engine.dll`. One of them essentially just does `xor eax, eax; mov [0x21300c88], eax`, so only the other one is of interest:

![](/breaking-videogames/assets/ServerGameDLL_addr_assignment.jpg)

`var_8h` is a pointer to `CreateInterface`. I think. Seems like the game fails to make the `ServerGameDLL002` interface from an unknown library. I would know more if I found out how to enable the logs it's writing, but for now, it just looks like it one day decided it can no longer load an interface from a `.dll` it was bundled with. I put investigating on hold at this point.

So I decree this today's second mildly cursed thing in Vampire: The Masquerade - Bloodlines. I haven't touched the game itself in a while, so I don't have a good guess what outside factor could have changed to produce this.

[Back to index](/breaking-videogames/)