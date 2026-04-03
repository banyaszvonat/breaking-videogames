---
layout: default
title: "twice-the-axing-price"
date: 2025-04-03 12:30:00 -0000
tags: disgaea psp graphics ge debugging textures palette
---

Back to Disgaea: Afternoon of Darkness factoids. While finishing up research for the Big Disgaea Post, I was stepping through the graphics engine commands, and in an intermediate state between two draw calls, I saw this:

![](/breaking-videogames/assets/exported_previous_palette_big.png)

This is one of the axes in the game, rendered with the Ghost (Tier 1 of the Spirit class) monster's palette, since it was dumped just before its own CLUT was loaded. Compare it with the axe graphic from the sprite rips available^[1] [here](https://www.spriters-resource.com/playstation_2/disgaea/asset/7956/):

![](/breaking-videogames/assets/axebasic_big.png)

The interesting thing to notice is that the axe's blade is made up of different color indices, despite being the same color in the final graphic. Taking another look at the sprite sheet, I noticed something I never realized in almost 20 years of playing the game on and off. Consider the following,

![](/breaking-videogames/assets/axeduplicate0_big.png)
![](/breaking-videogames/assets/axeduplicate1_big.png)

This sprite (and in fact, every Axe, Staff, Spear, and possibly other sprite), is pulling double duty: the first item's pixels are (nearly) a subset of the second's. Cleverly assigning the 16 available indices not only enables different colors for different rarities via palette swaps, but creating different shapes for another item (with its own set of 3 palettes for different rarities.)

I wonder how much of a nightmare it was for the artists to figure the color assignments out.

----

^[1] I wanted to rip the textures directly from memory myself, to supplement the Big Disgaea Post, but "Let's vibe code a quick Python script" turned into "Cross-referencing unofficial documentation and emulator source code for the swizzling algorithm" and I gave up instead.

[Back to index](/breaking-videogames/)
