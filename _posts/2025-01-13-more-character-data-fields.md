---
layout: default
title: "more-character-data-fields"
date: 2025-01-13 18:50:00 -0000
tags: psp disgaea savedata
---

Happy new year. I've been making some incremental progress towards decoding fields in the save data. Here are some:

- `0x660`: character level
	- Value is identical to the character's level, 1 in unused slots, so this is an educated guess.
- `0x681`: number of spells/skills the character has learned.
	- This was determined from the function at `0x08824b64` ^[1], which loads this value, then iterates over SkillLevels until the loop index reaches it.
- `0x689`: Currently wielded weapon
	- Also an educated guess: the function checks this field against offset `0xA2` of the skill definition struct later mentioned in the post. Skills belonging to different weapon types (Fist, Sword, Spear, Bow, Gun, Axe) have values `0x1`-`0x6`, whereas spells and specials have `0x0` at this offset in the skill definition struct. Staves don't seem to have a corresponding value, because they are not needed to cast spells, they merely boost spells' range if the character has one equipped.
- `0x68D`: Dark Assembly rank
	- Found through a read breakpoint on the address when opening the assembly screen, confirmed with memory editing.

Moreover, I found some sort of skill definition structure in memory. I'm not sure if they always get loaded at the same address, but `0x89ca9ac` at runtime holds a pointer to it. The array of structs has `0x11c` (284) members, and each struct is `0xAC` bytes long. This revealed a few skillIDs not covered in the guide:

- `0x0354`: Slumber
- `0x0357`: Charm
- `0x0BCD`: Soaring Fire
- `0x0BCE`: Vulcan Blaze
- `0x0BD7`: Rose Thorns
- `0x0BD8`: Rose Liberation
- `0x0BE1`: Zetta Beam
- `0x0BE2`: BadAss Overdrive

With the exception of Slumber and Charm, these are the skills of the DLC guest characters from later games. Since the PS2 version didn't have them, they weren't included in the guide.

The structure also contains skill descriptions, which gives away the 2-byte encodings of some symbols and punctuation characters that weren't included in the last post about it. But that requires some more effort to fully document.

----

^[1] European version of Disgaea: Afternoon of Darkness, address in RAM at runtime.

----

Bonus:
![](/breaking-videogames/assets/esilref.png)
![](/breaking-videogames/assets/heh.jpg)

[Back to index](/breaking-videogames/)