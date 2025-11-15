---
layout: default
title: "banishment-by-renaming"
date: 2025-11-15 14:45:00 -0000
tags: disgaea psp filler
---

What do you do when you don't want to bother with zeroing an array of about 100 kilobytes on every map change? Just flag entries bz clearing their name string. One byte per entry, very economical.

What happens when you store text characters on two bytes?

![](/breaking-videogames/assets/raveshop.jpg)

rave shop

This isn't a bug or anything. My text decoding filter falls back on ASCII the character value is outside the Shift-JIS ranges it recognizes, and I found this particular string funny.

-----

[Back to index](/breaking-videogames/)