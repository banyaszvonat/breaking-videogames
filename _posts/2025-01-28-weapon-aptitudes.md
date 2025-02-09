---
layout: default
title: "weapon-aptitudes"
date: 2025-01-28 05:30:00 -0000
tags: psp disgaea savedata
---

I think I found a hidden mechanic in Disgaea: Afternoon of Darkness. I'm not sure if this has been documented elsewhere, but apparently [not entirely unknown.](https://github.com/Xkeeper0/disgaea-pc-tools/blob/master/disgaea/data/character.php#L57)

In the game, characters possess aptitudes or proficiencies with weapons, which affects the rate at which they level up the weapon skill. The game displays this as a letter grade, ranging from E to A, and S. Today's discovery is that the game represents this as a 1-byte numeric value, meaning there are finer distinctions within the letter grades. I don't have all the possible values and their letter grades, but if I had to guess:

- S: `0x19`- (?)
- A: `0x14`-`0x18`
- B: `0x0F`-`0x13`(?)
- C: `0x0A`-`0x0E`
- D: `0x05`-`0x09`
- E: `0x01`(?) -`0x04`

Highest value I've seen so far for S rank is `0x22`.

I'm planning to do a longer post on how this was discovered, but tl;dr I set a read breakpoint on fields whose meaning isn't yet known, and when opening the status screen, the function at `0x08880180` accesses three sets of 8 fields. Noticed one set of values exactly corresponded to the displayed levels, another roughly corresponded to the aptitude letter grades on the status screen and that these fields were accessed together allowed an educated guess. The third set also roughly corresponds to weapon skill levels, but since the game doesn't display the exact XP value, and I don't know the XP curve, the guess that this is the XP was based on other cues.

What I'm not yet sure about is whether monster weapons have a corresponding XP/Level/Aptitude field, because if they do, the status screen does not display it, and it seems to go unused.

Also, a correction to the previous post: the field at offset `0x689` holds value `0x7` if a staff is equipped. Consequently, in skill definitions the value `0x0` probably means no associated weapon. Monster weapons are also represented as `0x0`.

The parser has been updated with the new field definitions, although the enums for weapon types and aptitudes aren't in yet.

----

Update 2/9/2025: Apparently it's [documented](https://disgaea.fandom.com/wiki/Weapon_Mastery) after all. The wiki page also identifies the official names as "Weapon Mastery", and gives the formula for converting weapon XP to weapon level.

----

[Back to index](/breaking-videogames/)