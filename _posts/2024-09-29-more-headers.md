---
layout: default
title: "more-headers"
date: 2024-09-29 00:30:00 -0000
tags: vtmb source save crestore
---

Quick update: we are in `CRestore::SetReadPos`, having been called from `CSaveRestoreBlockSet::ReadRestoreHeaders`:

![](/breaking-videogames/assets/crestore_setreadpos.jpg)


![](/breaking-videogames/assets/csaverestoreblockset_readrestoreheaders.jpg)

[The code in the SDK source](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/mp/src/game/shared/saverestore.cpp#L3195)

`edx` holds the location of the `Entities` block header in the in-memory buffer. In the hex dump on the bottom left, you can see this is between the `Python` and `worldspawn` strings from the previous post.

`CEntitySaveRestoreBlockHandler::ReadRestoreHeaders` tells us that the first field is an integer, `nEntities`:

[SDK source](https://github.com/ValveSoftware/source-sdk-2013/blob/0d8dceea4310fde5706b3ce1c70609d72a38efdf/mp/src/game/shared/saverestore.cpp#L2533)

The field in question is highlighted here in green, and according to this, we should expect 916 entities.

![](/breaking-videogames/assets/nentities.jpg)

Counting the entries by searching the range `0x9000`-`0x17CEE` [^1] for ASCII strings of at least 3 characters that start with 6 gets us...

![](/breaking-videogames/assets/stringsearch.jpg)

... 907 entries. Ehh, close enough.

This suggests a game plan for finding the Python data:

- Set a conditional breakpoint in `CSaveRestoreBlockSet::ReadRestoreHeaders` to stop when `Python` is being passed to `GetBlockHeaderLoc`
- Step through `CRestore::SetReadPos` to get the header's location in the in-memory buffer
- Locate it in the unpacked save file
- Reverse engineer `CPythonSaveRestoreBlockHandler::ReadRestoreHeaders` to see what fields it contains
- To get the actual data, do the same with `CSaveRestoreBlockSet::CallBlockHandlerRestore` and `GetBlockDataLoc`

----

[^1] `nEntities` is at offset `0x9010` in the file, while the last string that falls into the pattern of starting with a 6 is at `0x17BF2`. 

----

[Back to index](/breaking-videogames/)