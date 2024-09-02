---
layout: default
title: "save-splicing"
date: 2024-09-03 00:01:00 -0000
tags: vtmb source save
---

Sorry for the hiatus. It will happen again. In the last post I alluded to something yet more cursed lurking in the engine. I thought I found a flaw with a trivial route to exploit, but it turns out it's going to require more than two braincells. Maybe it'll make a good story for FailNight.

This is not to say it has been pointless. Today's post will be about the save file structure, and the failed first attempt exploiting. However, this also yielded some information about the loading process, which I'll be continuing with in a later post. Most of this is likely specific to Vampire, since the engine leaves a lot of room for developers to implement their own data structures.

I touched on how it works in [a previous post](/breaking-videogames/), but this is going to be a little more detailed. At the basic level, a save file is organized into blocks. [The Valve Developer Wiki has some information on this.](https://developer.valvesoftware.com/wiki/Save_Game_Files)

VtMB has one or more blocks of data belonging to particular maps. These are named `sp_foo.hl1`, `sp_foo.hl2`, and so on. For now, it's unclear what exactly controls the number of blocks, but the size of the data is a probable factor. I was somewhat inaccurate when I said each block starts with a zlib header: the block itself has a header with the name of the block and some size information. As an additional twist, a block may contain multiple zlib streams. Seemingly, this is due to a limit of 0x80000 bytes on the data to be compressed. If the data exceeds that size, the excess is split off and compressed into a new zlib stream. The streams are stored in a length-value format with no padding in between. Put together it looks something like this:

```
header1 (block/map name, e.g. sp_foo.hl1) 
padding? (236 bytes)
total size of decompressed data (4 bytes)
stream 1 compressed length (4 bytes)
zlib stream 1
stream 2 compressed length (4 bytes)
zlib stream 2
header2 (e.g. sp_foo.hl2)
...
```

I'll be referring to a block in the .sav file as a ".sav block".

Decompression is fairly straightforward. Iterate through the .sav block, decompress each individual stream, and concatenate the results. Since I didn't know beforehand whether concatenation needs to happen before or after decompression, I manually extracted the streams into intermediate files using the length information, then decompressed them with this Python script:

```
import zlib
import sys

if len(sys.argv) < 2:
    print(f"Usage: {sys.argv[0]} <filename>")

with open(sys.argv[1], "rb") as f:
    compressed = f.read()
    decompressed = zlib.decompress(compressed)
    outfilename = sys.argv[1] + ".decompressed"
    with open(outfilename, "wb") as wf:
        wf.write(decompressed)

```

Once this was done, I just concatenated the decompressed chunks.

Alternatively, VPKTool from the [Unofficial Bloodlines SDK](https://www.moddb.com/mods/vtmb-unofficial-patch/downloads/bloodlines-sdk) also has a function for extracting save files.

----

As a minor note, after doing some Offset Numerology, it seems very possible the game is using a static, fixed size buffer to write the header, because it leaks parts of filenames from previous blocks:

![](/breaking-videogames/assets/genesisdevice_hl2.jpg)
![](/breaking-videogames/assets/genesisdevice_hl3.jpg)
![](/breaking-videogames/assets/sp_theatre.jpg)

`_found_range.wav` is probably a fragment of a filename included in the Unofficial Patch, `tgt_discipline_no_target_found_range.wav`. The length of this string would also line up exactly with the start of the header.

236 is pretty close to 256, a round number in hexadecimal. If we assume a fixed size of 256 for the block name, here is how a block would look:

```
header1 (block/map name, 256 bytes)
unknown/always 0? (4 bytes)
total size of decompressed data (4 bytes)
stream 1 compressed length (4 bytes)
zlib stream 1
stream 2 compressed length (4 bytes)
zlib stream 2
header2
[...]
```

----

After decompressing, we have another block-based format on our hands:

I'll be calling the resulting container "VALV container" based on the magic value at the start. To disambiguate from .sav blocks, I'll be calling these inner blocks "VALV blocks".

Here's an ASCII rendition of how the containers are nested inside each other:

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
|
|-> .sav block "foo.hl2"
|   |
|   |-> zlib stream |
|   |-> zlib stream |---> VALV container
|   |-> zlib stream |
```

----

Now we get to what I thought was an easy exploit. After finding a VALV block called "Python", I also noticed the game calls `pickle.load()` during loading. Said block contains strings that correspond to names of entities in the game. 

![](/breaking-videogames/assets/pythonblock.jpg)

Vampire uses Python for mission scripting, so this started to look interesting.

I briefly tried to match these to the pickle protocol, but realized it'd be faster to just try to see if I can get the game to load something I inserted. Under the assumption that these are pickled Python objects, I generated a simple exploit payload:


```
class Exploit():
    def __reduce__(self):
        import nt
        return(nt.execv, ('C:\\Windows\\system32\\calc.exe',['x']))
        
import pickle

with open("sampleexploit.dump", "wb") as fout:
    pickle.dump(Exploit(), fout, protocol=1)
```

The resulting `sampleexploit.dump`:

```
Hex View  00 01 02 03 04 05 06 07  08 09 0A 0B 0C 0D 0E 0F
 
00000000  63 6E 74 0A 65 78 65 63  76 0A 71 00 28 58 1C 00  cnt.execv.q.(X..
00000010  00 00 43 3A 5C 57 69 6E  64 6F 77 73 5C 73 79 73  ..C:\Windows\sys
00000020  74 65 6D 33 32 5C 63 61  6C 63 2E 65 78 65 71 01  tem32\calc.exeq.
00000030  5D 71 02 58 01 00 00 00  78 71 03 61 74 71 04 52  ]q.X....xq.atq.R
00000040  71 05 2E                                          q..
```

This didn't look like it could have come from the savefile, but I chalked this up to there being different versions of the protocol. Something to try adjusting later, if it fails.

Though there's a visible structure in the VALV block, the boundary between serialized objects is ambiguous: they either end or begin with 0x1C (ASCII '6').

![](/breaking-videogames/assets/6infront.jpg)
![](/breaking-videogames/assets/6atend.jpg)

With no way to tell for now, this is another parameter to account for later. The potential versions are starting to add up already:

```
                       |  protocol=1 | protocol=2 |
---------------------------------------------------
objects start with '6' |      x      |            |
---------------------------------------------------
objects end with '6'   |             |            |
---------------------------------------------------
```

Anyway, after replacing an object in the extracted and concatenated VALV container, all that's left is to pack it back up again. To do so, the edited stream needs to be split up again into 0x80000 byte chunks, compressing the them separately with zlib, e.g.:

```
import zlib
import sys

if len(sys.argv) < 2:
    print(f"Usage: {sys.argv[0]} <filename>")

with open(sys.argv[1], "rb") as f:
    decompressed = f.read()
    compressed = zlib.compress(decompressed, level=9)
    outfilename = sys.argv[1] + ".compressed"
    with open(outfilename, "wb") as wf:
        wf.write(compressed)
```

The new compressed streams can be put in the .hl1 block in length-value format. Since the size of the VALV container has changed, the field after the header/padding indicating the length of the decompressed data also needs to be changed.

Upon loading the save:

![](/breaking-videogames/assets/expected_ETABLE.jpg)


...turns out this wasn't Python at all. I'm going to have to apologize: I led you down a garden path to an anticlimactic ending. But this is useful information. First of all, pickle protocol versions turn out to be a non-concern, because this is a Source Engine entity. And we learn where it starts reading:

![](/breaking-videogames/assets/read_offset.jpg)

(Unfortunately, this is not yet conclusive. You'll notice I forgot to add a null byte after `weapon_physcannon`, which could offset the first field. Additionally, while the format relies on data descriptions compiled into the application, but something tells me the `e` in `execv` could've been interpreted as the type of the following field.)

Finally, the error message itself is unique, and ultimately this was what helped me locate `CRestore::ReadFields` in the compiled binary. More on this and data descriptions later.

----

Bonus:

![](/breaking-videogames/assets/ctriggerelectricbugaloo.jpg)

[Back to index](/breaking-videogames/)