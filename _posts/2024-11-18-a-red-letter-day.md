---
layout: default
title: "a-red-letter-day"
date: 2024-11-18 01:00:00 -0000
tags: hl2 source
---

Recently, it was the 20 year anniversary of Half-Life 2's release. (This coincides with VtmB's 20 year anniversary, as they were released on the same day.) The anniversary update added developer commentary similar to what the Episodes shipped with.

![](/breaking-videogames/assets/commentary.jpg)

HL2 and/or Source engine fans have poured much time into analyzing the game, and compared to Vampire, there are a lot more trivia available online. For example, YouTube user Pinsplash has a series of videos that collect various facts relating to the game:

[Half-Life 2 facts (better quality) on YouTube](https://www.youtube.com/playlist?list=PLGRg3UhvfFNRJutMcsMvG_MkQ_iEV2FHG)
[Half-Life 2 facts (old) on YouTube](https://www.youtube.com/playlist?list=PLGRg3UhvfFNQisqmpUplI87MZx79OOPWS)

Consequently, I don't think I can show you anything novel that hasn't been posted before. However, I noticed an interesting implementation detail while playing through the commentary mode.

Throughout the game, there are screens that show you video from a camera in other parts of the scene. If I recall, this was particularly innovative at the time. In Kleiner's lab, you have this wall of monitors, with one of them allowing you to cycle through different camera views. The other monitors on the wall show a scrolling image.

![](/breaking-videogames/assets/monitors_circled.jpg)

You'd think this is implemented using the older technique of applying an animated texture on the surface of the screen, but surprisingly, this is likely also using an ingame camera. Just behind the monitor wall, there's a prism whose insides are textured with the scrolling image displayed on the screens.

![](/breaking-videogames/assets/monitors2.jpg)

Why? Well, why not?

----

[Back to index](/breaking-videogames/)