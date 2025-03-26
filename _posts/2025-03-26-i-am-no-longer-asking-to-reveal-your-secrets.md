---
layout: default
title: "i-am-no-longer-asking-to-reveal-your-secrets"
date: 2025-03-26 03:30:00 -0000
tags: disgaea classid
---

The blog has been quiet, and the at most 5 lost souls who are also looking at the GitHub may have noticed it didn't update either. The reason is that I found a lot of interesting things, got caught up documenting them as I went, and have been writing a cursed tome:

![](/breaking-videogames/assets/cursedtome.jpg)

That's more than 4x the size of the usual blog post. But in the interest of actually posting something, I'm going to split off a few parts and release it in smaller installments while I finish up the draft for another long post on the subject that may or may not end up elsewhere.

![](/breaking-videogames/assets/cursedtome2.jpg)

Now, let's get back to Disgaea.

----

After I've done all I think I could with read/write breakpoints, I decided to change approaches to the advanced technique "look up unknown offsets used as operands and also change some things in memory to see what happens". Don't remember how or when, but I also found a function at `0x08885b40` ^[1], which I knew was getting passed in a pointer to a character data structure. It does a couple of things that are interesting for our purposes.

- Whatever value it returns, it multiplies the value by 5 if the character field at `0x692` is set to anything other than zero. I wanted to know the meaning of this field, so I searched for all instructions that involve `0x692` as a literal operand. This turned up a bunch of other functions that all worked with character data, and eventually allowed provisionally determining that this is the "team ID" of a character. Sadly, I've forgotten how exactly. Probably just changing it in memory and noticing the game complains I'm trying to dispatch an enemy character. Regardless, this allows inferring what a function might be doing based on already-known meanings of fields, and knowing that in turn allows guesses at what unknown fields might be doing.

- The function turned out to be responsible for determining the cost using the healing service ingame. Beyond the inputs to the calculation being the HP and SP values of the passed-in character, the detail that allowed inferring what it does: it checks specifically whether a character is in a class known as Prinny, and returns 1 if it is. This is one of the longstanding conventions in the series, though later games make it a passive ability^[2] that can be applied to other characters.

- The specific check is for the value at offset `0x664`, which also happens to be the ID of the character's class in the [PC version](https://github.com/Xkeeper0/disgaea-pc-tools/blob/eaafcdf4cec6099fca331176bdd85b711794b7bf/disgaea/data/character.php#L68). It's (integer) divided by 10 and checked against the hardcoded value `0xdd` (221). The purpose of doing it this way seems to be to check for related classes at once. Classes can have upgraded ranks, and IDs are allocated such that similar classes share all but the least significant digit in base ten. For example, Warriors (female variant) have IDs 1040-1046, 1040 being the base class, 1041 the first upgrade, and so on. Then, the Majin line starts at ID 1050, 1051 being the first upgrade, etc. Prinnies have IDs ranging from 2210 to 2216, so dividing the ID by 10 and checking against that in principle^[3] checks for a character being a Prinny. The relationship between classes and ranks of classes will come up again, and I'll probably re-explain when it does, because it is a terminological clusterfuck.

- The function checks whether the character is affected by status conditions such as Paralysis, and bumps up the cost slightly if the field is nonzero. But it also checks whether the character is affected by other debuffs that (as far as I know) don't persist outside battle, such as Enfeeble. And adds to the cost if they are present. However, buffs simply use the same field as the corresponding debuffs, just with a positive value. So overall, if it were possible to carry buffs out of battle, they'd also increase the price. This seems to be an oversight that never becomes relevant.

It's unclear why the function checks for multiple impossible conditions. Checking for non-persisting debuffs could be explained by the function being written for a hypothetical earlier design iteration, where other debuffs did persist. However, I can't think of any circumstances where a character of a different "team" can be in the player's party. Maybe it's a multiplayer thing.

----

On that note I want to address Xkeeper's [comment at line 73.](https://github.com/Xkeeper0/disgaea-pc-tools/blob/eaafcdf4cec6099fca331176bdd85b711794b7bf/disgaea/data/character.php#L73) While writing this post, I think I realized why might this field have been added: the templates for skill advancement are in a contiguous array, whereas the class IDs have gaps in them. Storing the index of the template saves having to search the array for the particular class ID. Of course, it would have been possible to store the index corresponding to a class ID in a hash table or something.

As an incidental observation, doing it this way probably saves a few bytes of memory. The maximum size of the player's party is 128. The game allocates space for 64 deployed units on a map (10 of which are expected to be taken up by the player's units), giving us an approximate upper bound of 182 characters loaded somewhere in memory at once, with the possibility of a few more allocated elsewhere (there are character data entries for various NPCs in the main hub, and there is a wholly different array for characters sequestered in memory, but at the moment it seems that one is used instead of the party in the save game, when it does get used.) But let's be generous, and double it to 384. That translates to 384 extra bytes allocated. I'm going to ignore bytes lost alignment, because either there is none, e.g., the party ends at `0x089B5F38` and there is an array immediately after it. Or if it is aligned to 8 bytes: the character data structure's length is divisible by 8 as-is, so removing one byte would not translate to any gains. I could be wrong about this.

There are 362 entries in the template array in this version, and assuming we store the hash on 4 bytes, and the index on 2 bytes, that's 2172 bytes. Even if we store the hash on 2 bytes, that's 1448 bytes total. (You could go lower and permit hash collisions, but we're trying to avoid having to check the class IDs of array entries.)

But regardless, it was probably just easier. I don't think the developers were that constrained for memory that saving a couple hundred bytes mattered. Oh, and in a couple of places, the game linearly searches through the template array *anyway*, for example in a function to get the "base class" for a particular class ID.

----

The next post will either be more random findings, or will detail the chances of getting a consolation prize Mint Gum, in excruciating detail.

----

^[1] As before, these addresses are in RAM at runtime, in the European PSP release of Disgaea: Afternoon of Darkness

^[2] "Evility", probably a funnier pun in Japanese. Passive abilities were introduced in a limited form in the second game, and I believe it was only in the third game that they got properly integrated with other game systems. But evidently even the first game has a few passive abilities that exist as hardcoded rules keyed to Class IDs. Another one that comes to mind is Etna and Maderas' team attack chances being a fixed percentage. (IIRC, Maderas has a 100% chance of joining Etna in a team attack, while Etna has 0% chance of joining Maderas.) I actually didn't know that until reverse engineering the game, though it's written on the wiki too.

^[3] In practice, there are special classes that break the rule `baseClassID / 10 == actualClassID`, for example certain story characters based on existing monster classes. And there are some classes that are clearly related to others, but have their own ID and base class ID. These are possibly used to display portraits for different emotions in dialogue scenes, but their IDs are unrelated to the playable character's class ID. These differences might be why there's also a field for storing the base class ID, but I don't remember what actually checks that field at the moment of writing this. Coupled with the , there are at least 3 slightly different ways to determine the base class from a class ID: read the base class ID field if it's a character, zero the last base 10 digit, or use the function that searches the class templates mentioned above. To misquote Archer Sterling: "Do you want parser differentials? Because that's how you get parser differentials."

[Back to index](/breaking-videogames/)