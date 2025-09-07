---
layout: default
title: "hidden-functionality-in-spyro-the-dragon"
date: 2025-09-07 02:45:00 -0000
tags: spyro psx ps1 undocumented levelselect
---

So I was procrastinating on the Disgaea reverse engineering^[1] with reverse engineering Spyro the Dragon, and I think I found something wild. Short version:

At least in the US version of Spyro the Dragon (SCUS-94228), set the 32-bit value at `0x80075880` to `1`. Open the inventory screen, and tilt the analog stick or press any of the face buttons or D-Pad inputs. You'll trigger a level transition without a skybox, and you'll arrive at one of the levels within the current hub world.

![](/breaking-videogames/assets/purple.png)

As far as I was able to see so far, nothing sets `0x80075880`, so this might be a remnant of a cheat used during development. The flag gets reset after the level select, so you'll have to set it again.^[2]

Each level within a hub world corresponds to an index, and I believe they go in the order that they appear on the inventory screen. Here is the mapping between button and level indices:
```
Circle:		0 (The hub world itself)
X:		1
Square:		2
Triangle:	3
Right:		4
Down:		5
Left:		6 (Explicitly ignored -- worlds have at most 6 levels)
Up:		? (See below)
```

The last one is a bit more complex: it grabs the value from `0x8007596c`, increments it by one, adjusts it to be a valid level ID^[3], and loads the corresponding level. I'm guessing this address stores the last completed or visited level.

-----

How I found this:

Previously I identified global variables that seem responsible for storing the current level's ID, and looking through references to it, I stumbled upon a function at `0x8002d580` that checked for flags in a bitfield at `0x80077378`, set the current level ID based on the results, then called a bunch of functions. Using this bitfield as a pivot to move around in the code, I found a function at `0x80053c68` that calls `PadGetState()`. After a detour into reversing/looking up how gamepad inputs are processed, I ended up back at this function, and realized it held which inputs were pressed since the last time the gamepad input handler ran. Going back to `0x8002d580`, and looking at its callers, I quickly hit the game's main state machine, and found that it was called from game state 3 (the inventory screen), but only if `0x80075880` is set. At this point I loaded up the emulator, set the flag in memory, and was able to jump between levels on the inventory screen without any further modifications.

I couldn't find anything on the Internet about this. If it's really a new discovery, I'm surprised it went unnoticed for this long.

-----

^[1] Which is in turn procrastinating on the Vampire reverse engineering, but that's beside the point for now.

^[2] Addresses are as reported by Ghidra. In RetroArch+Beetle HW the flag in question is `0x00075880`.

^[3] The game actually stores and uses 2-3 IDs: the level ID, the world ID, and the index of the level within the world. This branch also sets the world ID if it's needed. An ID can be derived from the others using the following relations: `world_id * 10 + level_index == level_id`, `level_id / 10 == world_id`, `level_id % 10 == level_index`

[Back to index](/breaking-videogames/)