---
layout: default
title: "custom-encoding"
date: 2024-11-23 12:00:00 -0000
tags: psp savegame disgaea
---

Quick update: the decryption was, in fact, successful. There exist [guides](https://gamefaqs.gamespot.com/ps2/589678-disgaea-hour-of-darkness/faqs/35073) for editing the PS2 version's saves. The offsets are slightly different, but more importantly, this guide has a code table for text characters. Here's the encoding the save files use for letters in character names:

```
|A|0x8260| |N|0x826D| |a|0x8281| |n|0x828E|
|B|0x8261| |O|0x826E| |b|0x8282| |o|0x828F|
|C|0x8262| |P|0x826F| |c|0x8283| |p|0x8290|
|D|0x8263| |Q|0x8270| |d|0x8284| |q|0x8291|
|E|0x8264| |R|0x8271| |e|0x8285| |r|0x8292|
|F|0x8265| |S|0x8272| |f|0x8286| |s|0x8293|
|G|0x8266| |T|0x8273| |g|0x8287| |t|0x8294|
|H|0x8267| |U|0x8274| |h|0x8288| |u|0x8295|
|I|0x8268| |V|0x8275| |i|0x8289| |v|0x8296|
|J|0x8269| |W|0x8276| |j|0x828A| |w|0x8297|
|K|0x826A| |X|0x8278| |k|0x828B| |x|0x8298|
|L|0x826B| |Y|0x8279| |l|0x828C| |y|0x8299|
|M|0x826C| |Z|0x827A| |m|0x828D| |z|0x829A|
```

Using this, we can identify the strings "Laharl" and "Demon Prince" in the decrypted save file:

![](/breaking-videogames/assets/charnames.jpg)

----

[Back to index](/breaking-videogames/)