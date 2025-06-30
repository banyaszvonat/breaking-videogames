---
layout: default
title: "tony-hawks-pro-skatyr"
date: 2025-06-30 16:25:00 -0000
tags: tony_hawks_pro_skater thps tyr bug nintendo64
---

Momentum on the Disgaea investigation has slowed down, but I think I'm pretty close to a breakthrough. I just need to find out where a texture gets sent to the GE, but I haven't been taking the most productive approach. In the meantime, I'm going to talk about a project on the backburner.

The Nintendo 64 version of Tony Hawk's Pro Skater has a documented oddity; inputting certain names into the profile corrutps the profile text, as well as every other string on the UI.

[TCRF link](https://tcrf.net/Tony_Hawk%27s_Pro_Skater_(Nintendo_64)#Unknown_Feature)

It's hard to tell if this is an intended feature, a bug, or both. But curiously, there is a pattern to the corruption. A while ago I took the [list of names affected](https://gist.github.com/notwa/cf13e59d7978068c0ac103776d9e9e97), dropped them into an excel sheet, and stared at it for a while.

Sample:

```
AANGELEG      -> HRFXKLYVFFB
AANGENOMENEN  -> HRFXKNRIKZO
AANGEROL      -> HRFXKRDKWTV
AANSTUIVING   -> HRFIIVUPSXT
AANTRAD       -> HRFJBNGVICJ
AARSKOERNING  -> HRJVLROLITS
AARTSBRO      -> HRJWHOVNTOG
AASE          -> HRKWLPTIGOG
ABADIA        -> HSH BLGXZSU
ABAISSERA     -> HSHEKHWJVEX
ABATO         -> HSHPDWSPNZW
ABATTIRENT    -> HSHPIMBCZDL
ABBAIERA      -> HSILOXLEXAH
ABBI          -> HSITEZCKMIN
ABFAERBENDER  -> HSMEROYUXFW
ABGEERNTETER  -> HSNXXVZOSWN
ABLINGS       -> HSSIQAUYPLN
ABROTINE      -> HSYKCURTJXE
ABSTEIGE      -> HSZD VTRMDH
AC   A        -> HTVUG NJPDD
ACABDAR       -> HTWKYJEKXAP
ACCABLERAIENT -> HTYTPZHGYVS
ACCUSAVATE    -> HTYMVKAXZNK
ACIDIZE       -> HTDJUNCSZNK
```


The interesting part is that it almost behaves like a cipher. Looking at the first character in isolation, you can treat it as a Caesar/ROT cipher, but if I remember correctly, subsequent characters' offset depends on the preceding characters, and eventually it was pretty hard to eyeball. The above gist also seems to have a [decoding function](https://gist.github.com/notwa/cf13e59d7978068c0ac103776d9e9e97#file-thps1-c-L58) for them.

Someday, I want to take another stab at figuring out why this happens, i.e.: whether this is a bug, buggy feature, or something intentional. The PSX version of the game has a notorious bug that got us [tonyhax](https://orca.pet/tonyhax/), so I feel like this is likely a bug that happens to behave very interestingly.

I have relatively little experience reverse engineering N64 code, so I want to offer a symbolic bounty: if you can locate the code handling profile name entry, i.e. where this corruption happens, and point me to the appropriate location in any applicable regional version of the binary, I will award you one (1) banyaszvonatcoin[1]. Send an e-mail to banyaszvonat@proton.me .

I will also potentially publish your results on the blog with credit, if desired (and it's original research). If this task can be cleared by a simple google search, I will post a link to the research and a shoutout for the tip. In this case both the tipster and the publisher of the original research will get a banyaszvonatcoin.

-----

[1] banyaszvonatcoins are lovingly hand-crafted and possibly cryptographically signed PNG tokens of recognition that are awareded to specific usernames, trade names, or any other self-applied or commonly-known identifier (you do not have to be people to be eligible for a banyaszvonatcoin). They carry no value whatsoever.

[Back to index](/breaking-videogames/)