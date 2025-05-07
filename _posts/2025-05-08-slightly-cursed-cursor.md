---
layout: default
title: "slightly-cursed-cursor"
date: 2025-05-08 02:30:00 -0000
tags: disgaea implementation_shortcut
---

Walking up the call tree from the various text output functions, I was able to find a couple of functions that seem responsible for actually implementing the corresponding game states (for example, the item shop menu). Going further up I was also able to locate what I think is the main game loop at `0x0881508c`, and the function that seems to implement/manage the game's state machine at `0x0886d070`. The global game state is stored in the byte at `0x088e9de0`.

Switching this byte to `0x15` activates the behavior seen in the castle that serves the main hub, being able "walk around" with the cursor, and talk to enemy NPCs, resulting in some placeholder lines being displayed:

![](/breaking-videogames/assets/battletalk.jpg)

I was kind of surprised that changing a single variable worked this well, because I remember reading on [TCRF](https://tcrf.net/Disgaea:_Hour_of_Darkness#In-Game) that a similar "Freely walk around any map" function is accessed via forcing a script to run via the game's debug menu.

This works in the other direction too. In the game's hub area, after switching the team of at least one NPC to `0x1` to prevent an instant victory and a subsequent crash (e.g.: change `0x088eddfd` from `0x2` to `0x1`,) setting the same address to `0xb` switches the game's behavior to battle mode. This results in a few interesting things. Hard to illustrate with screenshots, but Laharl starts hovering up and down, like the cursor normally does during battle.

![](/breaking-videogames/assets/laharlcursor0.jpg)
![](/breaking-videogames/assets/laharlcursor1.jpg)

This suggests that the character you're controlling in the hub is not a proper unit, but in fact your cursor.

This also exposes a trick the developers had to use to work around a limitation. When walking around in the hub, it's possible to interact with NPCs, but only from adjacent tiles. Talking to the NPCs behind the counter opens the item shop. However, this seemingly breaks the rule of interacting from adjacent tiles. The reason talking still works is because there are two invisible NPCs on the counter with identical portraits. These are the NPCs you're actually interacting with.

![](/breaking-videogames/assets/rosen0.jpg)

The ones behind the counter are just for show, and also display placeholder lines when talked to.

![](/breaking-videogames/assets/rosen1.jpg)
![](/breaking-videogames/assets/rosentalk.jpg)

-----

[Back to index](/breaking-videogames/)