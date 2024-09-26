---
layout: default
title: "the-plot-thickens"
date: 2024-09-26 23:00:00 -0000
tags: vtmb source save blockhandler
---

>(Unfortunately, this is not yet conclusive. You'll notice I forgot to add a null byte after `weapon_physcannon`, which could offset the first field. Additionally, while the format relies on data descriptions compiled into the application, but something tells me the `e` in `execv` could've been interpreted as the type of the following field.)

So, my speculation turned out to be wrong, and the engine does rely on data maps after all. The `ETABLE` string visible in the error is passed in as an argument to `CRestore::ReadFields`, by `CEntitySaveRestoreBlockHandler::ReadRestoreHeaders`, where it's declared as a literal:

[Link on GitHub](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/mp/src/game/shared/saverestore.cpp#L2544)

At the time this error message occurs, the engine is processing the `Entities` block. A string related to this can also be seen on the stack:

![](/breaking-videogames/assets/entities_block_on_stack.jpg)

As for why this rules out type fields? I can give the following reasons:

- As mentioned, `ReadFields` takes the type to expect as a parameter.
- The save file is processed by block, and each `SaveRestoreBlockHandler` uses `Save` and `Restore` to implicitly define the structure of their blocks. It passes in the type description parameter to `ReadFields`, so by the time it's called, both the address to read from and the expected type is known. And finally:
- Trust me bro

However, scrolling through the stack area that's no longer allocated reveals something interesting:

![](/breaking-videogames/assets/python_block_already_processed.jpg)

The engine had already gone over the Python block, and moved on to the entities block. Yet the buffer pointer is seemingly in the Python block:

![](/breaking-videogames/assets/whichblockisthis.jpg)

These are in the Python block... right? Right?

![](/breaking-videogames/assets/headersandentities.jpg)

I am glad I didn't speculate on the anatomy of a VALV container yet. Let's do that now. If you recall:

```
.sav file
|
|-> .sav block "foo.hl1"
|   |
|   |-> zlib stream |
|   |-> zlib stream |---> VALV container
|   |-> zlib stream |     |
|                         |-> VALV block "Foo"
|                         |-> VALV block "Bar"
```

Given that names like `weapon_physcannon` and `worldspawn` in fact correspond to ingame entities, I think there is a possibility that the headers are split from the contents of the block. Zooming in, perhaps the container looks like this:

```
VALV container
|
|-> Block name "Foo"
|-> Header "Foo"
|-> Block name "Bar"
|-> Header "Bar"
|-> "Foo" block data
|-> "Bar" block data
```

At this point, this is just a guess.

----

[Back to index](/breaking-videogames/)