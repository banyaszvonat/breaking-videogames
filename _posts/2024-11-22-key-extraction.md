---
layout: default
title: "key-extraction"
date: 2024-11-22 21:30:00 -0000
tags: psp savegame emulation mips disgaea
---

I've been procrastinating on getting back to Vampire, so this is post is going to be about PSP savegames. Looking to reverse engineer a save game, I quickly found out that PSP savegames are encrypted.

Researching how to bypass this, I found a PSP-based save-editing tool:
[bucanero/apollo-psp on GitHub](https://github.com/bucanero/apollo-psp)

The README has the information needed:
- Save games are encrypted by a game-specific key
- This encryption is provided by the PSP kernel

Dumping this key is reasonably easy on real hardware using custom firmware, but since I'm using PPSSPP to run games, I wanted to see if I can use an emulator to extract the encryption key, since it's static to a particular game.

The README also helpfully linked to a key dumper plugin, whose behavior I decided to try to replicate by hand:
[psptools/SGKeyDumper on GitHub](https://github.com/bucanero/psptools/tree/ee050436680455812726c651709a194acf441040/SGKeyDumper)

This plugin searches for `sceUtilitySavedataInitStart`, hooks it and grabs the key from the save data structure it gets passed. 

At first I was confused by how `FindExport` works [here](https://github.com/bucanero/psptools/blob/ee050436680455812726c651709a194acf441040/SGKeyDumper/src/main.c#L99), but as I later learned, the third parameter is the so-called "NID", the first 32 bits of the SHA-1 hash of [an export's name](https://wololo.net/2012/05/30/syscalls-nids-imports/), used as the identifier. Either way, the next line names the function we're looking for.

----

Obviously, I don't have a `FindExport` function in the emulator, but after starting up the game, I exported a symbol file:

![](/breaking-videogames/assets/sym.jpg)

I used this to locate the address of `sceUtilitySavedataInitStart`. As far as I understand, the load addresses of the exports are randomized between boots, so this would need to be re-exported every time a game is loaded. However, it turned out to be superfluous, since the disassembler has a list of symbols. We can just set a breakpoint here:

![](/breaking-videogames/assets/breakpoint.jpg)

The subroutine the plugin intercepts is 0x18 from the function's start, and `a1` should hold the pointer to the save data structure to extract the key from.

![](/breaking-videogames/assets/subroutine.jpg)

However, the value of a `a1` decidedly does not look like the pointer. (Stepping to the syscall instruction makes no difference.)

I stepped into `__UtilityWorkUs`. Coincidentally, there also happens to be an unnamed subroutine 0x18 from the start, so it occurred to me, maybe this one is being hooked, but the code seems to say otherwise.

![](/breaking-videogames/assets/subroutine2.jpg)

The debugging would have hit a wall here, but I decided to inspect the other registers, and as it happens `s0` pointed somewhere that looks very much like the structure in question. I don't know enough about MIPS register semantics on the PSP enough to tell if this is just a coincidence.

----

At addresses slightly below the one in `s0`, there is something that looks like save metadata:

![](/breaking-videogames/assets/memory.jpg)

At this point I decided to see if I can find something about the structure passed to `sceUtilitySavedataInitStart`, to see if the definition fits the data. First, there was a false start, when I found [old documentation](https://www.freeshell.de/~sven/pspsdk/structSceUtilitySavedataParam.html) from the unofficial PSP SDK called psp-sdk. However, this seemed to be incomplete and/or wrong. I saw similar claims in a thread [about someone trying to do the same thing](https://wololo.net/talk/viewtopic.php?t=43988).

The SDK [on GitHub](https://github.com/pspdev/pspsdk/blob/master/src/utility/psputility_savedata.h#L192) seems more accurate.

Anyway, parts of the `SceUtilitySavedataParam` structure seem to line up with the memory contents.

![](/breaking-videogames/assets/memory2.jpg)

After skipping past the irrelevant fields in the structure...

![](/breaking-videogames/assets/key.jpg)

Using [another tool](https://github.com/38-vita-38/psp-save) I found on GitHub, decrypting with mode 5, the result was something that doesn't look encrypted:

![](/breaking-videogames/assets/result.jpg)

The resulting file doesn't contain ASCII strings, so I'm not entirely confident it worked, but figuring out the format will come later.

----

[Back to index](/breaking-videogames/)