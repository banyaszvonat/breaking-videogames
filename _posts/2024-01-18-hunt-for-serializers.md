---
layout: default
title: "hunt-for-serializers"
date: 2024-01-18 00:00:00 -0000
tags: vtmb radare2 strings parsing save serialization
---

Happy new year. First, I would like to give a reminder to back up your project files in git. The project encountered a minor setback when rizin decided to throw out previously-saved names for functions when saving a new version. Luckily, I do use git, but I had to review about 600 changes to hopefully produce a projectfile that incorporates both changesets.

![](/breaking-videogames/assets/yodawg.jpg)

I'm not sure if I mentioned, but the long term goal of this exercise is to gain insight into a bug plaguing my Vampire - The Masquerade: Bloodlines save file. In short: after some time, discipline timers are highly likely to break, never showing the bar that represents remaining time. It affects a particular map the bug is triggered on, and once it happens, the bug persists through leaving the map and reentering, as well as saves and loads.

According to the author of the Unofficial Patch, this is a bug that cannot be fixed in script, because the discipline timer is hard-coded into the executable. But I don't think that means all is lost. Since it only affects particular maps on particular saves, I suspect it has to do with improperly serialized/deserialized game state. So I decided to try looking into that.

When you open up a save file, you might notice that each block begins with `0x78 0xDA`, which a quick search reveals to be [a zlib magic number](https://stackoverflow.com/questions/9050260/what-does-a-zlib-header-look-like).

![](/breaking-videogames/assets/savefile_block.png)

The censored parts are fragments of PATH. No idea why it does that.

Decompressing it via a quick python script yields something that looks like another serialized format. `VALV` or `VALVu` is likely another magic header, but information about this is scarce. Here's an example. Note that it's not the same block as depicted of the uncompressed block.

![](/breaking-videogames/assets/uncompressed_block.png)

This is actually the block I think is the likely offender. So far, it's also the only one where the presumed size field for the compressed data does not match the compressed data's length.

It may take a while until the next post, because the lazy approach to finding the attribute writer functions didn't work out, and I'll have to grind away at it for a bit.

[Back to index](/breaking-videogames/)
