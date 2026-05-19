---
layout: default
title: "intermission"
date: 2026-05-20 00:01:00 -0000
tags: borderlands easter_egg unreal_engine ue3 willow
---

Unexpectedly, I fell into a rabbit hole. It started with installing a couple of Borderlands 2 mods, and gradually escalated to looking at the game's internal APIs to see what can be hooked. I was looking through the WillowPlayerPawn class and:

![](/breaking-videogames/assets/easteregg1.png)

Hello, what is this? The only other time this property gets referenced in the main package is when getting into a vehicle, where it's also set to `0.0`. A full exploration will have to wait for another day, because I do believe I have a different RPG to get back to. But if you happen to know anything about the numbered `EasterEgg` properties in `WillowPlayerController`, you can always reach me at `banyaszvonat@proton.me`.

----

[Back to index](/breaking-videogames/)