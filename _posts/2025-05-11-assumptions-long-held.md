---
layout: default
title: "assumptions-long-held"
date: 2025-05-11 16:30:00 -0000
tags: windows critical_section 24h2 gta_san_andreas sid_meiers_alpha_centauri gtasa smac
---

Bit of an extraordinary post -- I am not the one breaking video games this time. Recently, Microsoft released update 24H2 for Windows 11. One of the changes seem to be in the implementation of critical sections. And just today, I've already seen two posts about older games breaking in hard to debug ways in the most recent update.

First, from the author of SilentPatch, a suite of compatibility and QoL enhancements for Grand Theft Auto, an analysis of a bug in GTA: San Andreas that causes a specific vehicle type to fail to ever appear if running this version of Windows:

[How a 20 year old bug in GTA San Andreas surfaced in Windows 11 24H2](https://cookieplmonster.github.io/2025/04/23/gta-san-andreas-win11-24h2-bug/)

Second, a video by Nathan Baggs, about fixing a crash bug in Sid Meier's Alpha Centauri: Alien Crossfire:

[How Windows 11 Killed A 90s Classic (& My Fix)](https://www.youtube.com/watch?v=0nEy4iAdbME)

The common thread between these bugs seems to be (probably unintentional) reliance on undefined behavior, specifically, uninitialized variables having values that ended up not causing issues. Presumably this is what allowed the bugs to go unnoticed until now. Both of these occur around calls to `EnterCriticalSection()`/`LeaveCriticalSection()`. The UB just so happened to have limited ramifications all the way up to the recent update. The fault is clearly not in the OS update, but the change seems to have brought them to light after decades.

I think this could have wider consequences. These are just games breaking in funny ways, but imagine all the legacy, critical software running in more serious applications. I don't want to be alarmist, but I'm thinking there is a small chance this could be apocalyptic. I'm writing this down so I can be like "called it", if more things start breaking.

Happy debugging.

-----

[Back to index](/breaking-videogames/)