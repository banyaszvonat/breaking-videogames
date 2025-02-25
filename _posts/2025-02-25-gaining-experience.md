---
layout: default
title: "gaining-experience"
date: 2025-02-25 19:00:00 -0000
tags: disgaea savegame
---

Found the function responsible for calculating the XP required for the next level.

The function at `0x08880474` eventually accesses offsets `0x660` (the character's level) and `0x668`, an unknown field as of yet. It then calls `0x08880e80`, with the following parameters: `FUN_08880e80(unknown_field, char_level + 1)`

The unknown field gets used as an index into an array of structs. The structs look something like:

```
{
	short name[16];
	byte name_end;
	short title[13];
	byte title_end;
	byte unknown[8];
	byte weapon_masteries[8]; //possibly just 7 elements, depending on whether monster weapons have a dedicated mastery value
	byte unknown2[38];
	short XPCalculationValue; // accessed by 0x08880e80
	byte unknown3[102]; // total length of the structure is 0xd8
}
```

The function also uses the passed-in level parameter to index into a table of 64 bit values at `0x896c2d0`. By all indications, this is the table holding the "base" XP values required for the next level. To test this assumption, taking into account the minimum and maximum levels in the game (1 and 9999 respectively), and that there needs to be a value that's at least one higher than the maximum level, the last entry should be at `0x896c2d0 + (8*10000) ==  0x897fb50`. As indicated by the change in the pattern of bytes when eyeballing it, this seems to be the case.

![](/breaking-videogames/assets/xp_req_table.jpg)

Anyway, ultimately it ends up multiplying the base XP requirement value with XPCalculationValue. To accomplish this, it calls another function at `0x088b6d8c`, which seems to be a generic 64-bit multiplication using the FPU.

----

Changing values in memory and observing corresponding changes, the following one-byte fields were identified:

- `0x305`: Poison status flag
- `0x306`: Sleep status flag
- `0x307`: Paralyze status flag
- `0x308`: Forget/Amnesia status flag
- `0x309`: Deprave/Weaken status flag
- `0x68B`: Unit portrait ID
- `0x68C`: Unit portrait palette ID
- `0x691`: Precise meaning is unclear, but some sort of character index or ID: by default equals the position of the character in the character data array, and the status screen uses it to determine pupil list (characters with the same mentorID as this field will be shown)

----

After identifying the ailment flags, I noticed that the function that computes the cost of healing at the hospital also checks a few more addresses in memory after them. So I tried casting some buffs and debuffs with the memory viewer open, and the following fields seem to store them:

- `0x30C`: Braveheart/Enfeeble
- `0x30D`: Shield/Armor Break
- `0x30E`: Magic Wall
- `0x30F`: Magic Boost

The values these got were `0x14` (20) and `0x28` (40), possibly denoting the magnitude of the effect. Additionally, Braveheart/Enfeeble and Shield/Armor Break got the value `0xEC` (-20, presumably), when affected by the debuff versions.

Fun fact, if I'm reading the decompiled code right, if it were possible to bring these "conditions" outside battle, they would each add a HL cost equals to 3 times the character's level. (This also applies to status conditions like Poison, which I believe can persist after a battle. But presumably that's intended behavior.)

----

[Back to index](/breaking-videogames/)
