---
layout: default
title: "off-by-one-or-is-it-two"
date: 2025-03-30 12:00:00 -0000
tags: disgaea savegame senators darkassembly bug
---

So, next post was going to be about probabilities, but I really wanted to share something I just found.

Disgaea: Afternoon of Darkness (and later titles) have a feature where you can submit various propositions to what is known as the Dark Assembly, that alter the game or unlock features. The game generates "senator" NPCs that vote, can be bribed, or fought to pass a bill.

The data related to senators in the Dark Assembly mechanic is stored in an array of 20-byte structures after the player's inventory in the save data. This includes the desired rarity value of items to be given as bribes. (The closer the given item's rarity to the number, the higher the disposition increase). At the moment, the known fields look like this:

```
struct SenatorData {
	u8;
	u8;
	u8;
	u8;
	u32;
	u8;
	u8;
	u8;
	u8 FavoriteRarity; // offset 0xb FUN_08871d14
	u8;
	Text<9> Name;
	u8;
};
```

The name string is stored in the senator data directly, on bytes `0xd` to `0x1e`. However, `0x08871d14` references offset `0x1e` separately. It would seem that was intended to be another field, but gets overwritten by the last byte of the name, if it's 9 characters long. This is probably an oversight. However, said function sanity checks the value to be within expected bounds, and clamps it if it isn't. It writes back the value after doing something with it, so in theory bribing a senator with a 9-letter name will alter the last letter in their name on their subsequent appearances.

Before bribing:

![](/breaking-videogames/assets/bribing_senator.jpg)

After bribing:

![](/breaking-videogames/assets/post_bribe.jpg)

So what our struct really looks like is:

```
struct SenatorData {
	u8;
	u8;
	u8;
	u8;
	u32;
	u8;
	u8;
	u8;
	u8 FavoriteRarity; // offset 0xb FUN_08871d14
	u8;
	Text<8> Name;
	u8;
	s8 SenatorDisposition;
	u8;
};
```

I kind of wonder if this issue is exclusive to the English versions of the game. In principle it isn't, but whether it happens is dependent on the actual name strings in the game data. Luckily, the effects here are relatively contained^[1], which might be why it slipped past QA. Also, 🫴🦋 is this a zero day?

Edit: if you're wondering about the title, or why one letter difference affects a field two bytes after the name, this is due to the two-byte encoding of text mentioned in a previous post. I called it a custom encoding in error, but it is actually Shift-JIS. The reason I didn't realize was because the game uses the fullwidth versions of Latin characters and punctuation, which I didn't know were part of the character set until digging deeper into it. This is also why I wonder if maybe this was an oversight on part of the translation team, and exclusive to the English version that way. Anyway, I'm going to eventually swap out the ad-hoc decoder in the savegame parser.

----

^[1] "Foreshadowing is a narrative device in which a storyteller gives an advance hint of what is to come later in the story." - Wikipedia

[Back to index](/breaking-videogames/)
