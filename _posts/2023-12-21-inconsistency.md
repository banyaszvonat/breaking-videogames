---
layout: default
title: "inconsistency"
date: 2023-12-21 00:00:00 -0000
tags: vtmb kaitai strings parsing save
---

Whatever is serializing these can't make up its mind whether to terminate strings with one or two null bytes.

![](/breaking-videogames/assets/mysterious0.jpg)

![](/breaking-videogames/assets/mysterious1.jpg)

I started with the assumption that there are two null bytes after every string, and this is thrown into question. Which wouldn't be a problem, but the end of the container is apparently signified by THREE null bytes, which already required an awkward hack. My one guess for explaining this: perhaps the second `0x00` is an empty string that got written out, but that would imply every other string is empty in whatever structure this comes from. Mysterious.

[Back to index](/breaking-videogames/)
