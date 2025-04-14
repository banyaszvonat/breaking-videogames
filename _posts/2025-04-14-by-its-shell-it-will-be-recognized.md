---
layout: default
title: "by-its-shell-it-will-be-recognized"
date: 2025-04-14 00:40:00 -0000
tags: disgaea strings
---

In [A Debugging Trick](/breaking-videogames/2023/12/05/a-debugging-trick.html) I mentioned a low-effort approach I like to use, which is to find strings in the executable, and see what references them. For Disgaea, I couldn't initially use this trick even after solving the encoding, for a seeming lack of static references to strings. But by chance, eventually I stumbled upon a function whose only purpose was to index an array holding pointers to all the strings the game uses. At `0x088839ec`, then at `0x08828a00`, then at `0x088aeb50`...

These are byte-for-byte identical, and decompile to something like the snippet below, and most importantly, all reference the same array:

```
ShiftJISChar * get_ui_string_by_index(int param_1)

{
  return UI_text_pointer_array[param_1];
}
```

I've finally gotten around to checking the cross-references to that array, and turns out there are 16 of them in total. I don't know why this is, but more experienced hackers suggested it might be a something the compiler added (thx Toma!). Notably, parts of the code calling the same version of the function are usually conceptually related, which I think lends credence to this idea.^[1] And/or perhaps it is a macro duplicated across different translation units.

Having all of them is really useful, because it's possible to get a rough idea of what a function is doing based on the strings it retrieves, just like before. I went through and named most of the functions calling these `get_ui_string_by_index` instances, and now I have a rough skeleton of the game. Or to use a better metaphor, a shell: as hinted by the above snippet, almost every string in the array is meant to end up on the user interface, with a couple of format strings^[2] and other miscellaneous things thrown in. So this provides a rough map of the portions of the code that directly interact with the user, and from there it's possible to proceed "inwards".

----

I was editing the probabilities post, noticed what might be an error in the formulas, and I realized I've been staring at it for so long that I can't make heads or tails of it anymore. So that's going back into the drawer, but here is the abridged version:

At address 0x0884d134, there is a function that appears to be responsible for randomly generating items. It takes a rank and an item category as a parameter, and randomly picks^[3] from the array of all item definitions 300 times, checking if the result is the desired rank/category.^[4] If after 300 times it doesn't get a match, it reduces the rank by one, and it tries 200 more times, and so forth until it reaches the 200th try for rank 1, when it gives up and gives you an ABC Gum.

There are 478 items in the game, so the probability of getting the item with the given rank is:
```
1 - (477/478)^300
```

Which, rounded to 4 decimal places, is `0.4665`, or 46.65%.

It's more convenient to compute the probability of failing the first 300 rolls, which is:

```
(477/478)^300
```

And then subtracting that from 1 gets the probability of its complement.

Also, if the first 300 rolls fail, it goes down one rank, then tries again 200 times. So the probability of getting an item one rank lower than the function was originally given is:

```
(477/478)^300 * (1 - (477/478)^200)
```

Then, if that fails too:

```
(477/478)^300 * (477/478)^200 * (1 - (477/478)^200)
```

And so on and so forth, until the lowest rank is reached. So the probability of getting an ABC Gum instead of an item of rank `v` should be:

```
(477/478)^300 * ((477/478)^200)^(v-1)
```

Maybe. Also, I haven't looked at where specifically this function is used.

----

^[1] As an example for what I mean by conceptually related: if one function uses `get_ui_string_by_index` #12 to retrieve a Music Shop related string, all the other callers of version #12 will also be retrieving Music Shop-related strings, presumably implementing the other menu screens for that feature.

^[2] One wonders about the wisdom of slapping user-controlled strings into printf when one remembers it is Turing-complete. But for now, that is beyond the scope of the current project.

^[3] Haven't solved how the RNG works yet, but for the purposes of the probability calculation, I'm assuming a uniform probability.

^[4] It also rolls again if the result is in category `0x12`, which contains special items with a fixed method of acquisition.


[Back to index](/breaking-videogames/)