---
layout: default
title: "read-whence"
date: 2024-11-10 18:00:00 -0000
tags: vtmb source save crestore
---

Here's a small update. There is a structure in the SDK code that looks like it could be defining "primary" header for a block, including pointers to another header and the data:

```
struct SaveRestoreBlockHeader_t
{
	char szName[MAX_BLOCK_NAME_LEN + 1];
	int locHeader;
	int locBody;

	DECLARE_SIMPLE_DATADESC();
};
```

Two of the blocks from the save file:

```
Hex View  00 01 02 03 04 05 06 07  08 09 0A 0B 0C 0D 0E 0F
 
00008EC0  45 6E 74 69 74 69 65 73  00 00 00 00 00 00 00 00  Entities........
00008ED0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................
00008EE0  04 00 B2 2F 74 01 00 00  04 00 76 00 01 00 00 00  .../t.....v.....
00008EF0  3C 00 14 0F 04 00 09 09  03 00 00 00 20 00 E3 06  <........... ...
```

```
Hex View  00 01 02 03 04 05 06 07  08 09 0A 0B 0C 0D 0E 0F
 
00008FD0                           50 79 74 68 6F 6E 00 00          Python..
00008FE0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................
00008FF0  00 00 00 00 00 00 00 00  04 00 B2 2F D5 ED 00 00  .........../....
00009000  04 00 98 09 43 BC 0A 00  04 00 98 09 FF FF FF FF  ....C...........
```

Trying to overlay the struct definition shows that it isn't entirely consistent with the source code, but the build used for Vampire is a decades older, so we'll just ignore that for now:

```
45 6E 74 69 74 69 65 73 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 - szName, "Entities"
04 00 B2 2F - static?
74 01 00 00 - locHeader (offset)
04 00 76 00 - locBody?
01 00 00 00 3C 00 14 0F 04 00 09 09 03 00 00 00 20 00 E3 06 - datadesc?
```

```
50 79 74 68 6F 6E 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 - szName, "Python"
04 00 B2 2F - static?
D5 ED 00 00 - locHeader (offset)
04 00 98 09 - locBody?
BC 0A 00 04 00 98 09 FF FF FF FF - datadesc?
```

So far, I've been able to make things easy by referencing the source code, but the Python block handler is a Vampire-specific addition, and there's no associated code in the SDK. However, since the various `SaveRestoreBlockHandler`s derive from the same interface, it's still fairly easy to find the methods of interest.

Incidentally, setting a breakpoint on `CPython_SaveRestoreBlockHandler::ReadRestoreHeaders` as well as `CEntity_SaveRestoreBlockHandler::ReadRestoreHeaders` revealed that the Python block either does in fact get processed later than the entities block, or the game touches both blocks multiple times during loading.

We can take a shortcut around trying to comprehend the header format by noticing that `CSaveRestoreBlockSet::CallBlockHandlerRestore` calls `GetBlockHeaderLoc` and sets the read location on the `IRestore` instance that `CPython_SaveRestoreBlockHandler::Restore` gets as an argument. This gives us an offset into the save data file after it has been mapped into memory. 

----

[Back to index](/breaking-videogames/)