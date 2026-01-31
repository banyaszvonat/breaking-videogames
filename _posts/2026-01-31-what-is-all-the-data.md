---
layout: default
title: "what-is-all-the-data"
date: 2026-01-31 23:00:00 -0000
tags: spyro psx overlays
---

Happy new year. I wanted to post something this month, but ran out of time to polish this one and remove inaccuracies. I'm still posting it, but most of what follows should be considered misinformation.

![](/breaking-videogames/assets/spreadmisinformation.jpeg)

Around when I made my last post, I found a tool on GitHub that could unpack Spyro's WAD files:

[https://github.com/altro50/wadtool/](https://github.com/altro50/wadtool/])

Turns out there is a convenient table of contents for what each section of the WAD file is used for:

[https://github.com/altro50/wadtool/blob/master/src/wads.rs](https://github.com/altro50/wadtool/blob/master/src/wads.rs)

This actually helped work backwards and figure out the code that loaded the overlays. So let's give a top level overview to try to corroborate these findings. (Spoilers: partial results)

[https://github.com/altro50/wadtool/blob/master/src/wads.rs#L195](https://github.com/altro50/wadtool/blob/master/src/wads.rs#L195)

The overlays are stored in `WAD.WAD` on the disc. This begins with a 2048 byte header composed of pairs of 4-byte numbers. These are offsets and sizes of each section. The header can hold 256 pairs in total, but in practice fewer slots are used. It was fairly obvious this file had all the game data, but I couldn't see where it was loaded until I realized it's not loaded by filename, but by CD sector address.

The function at `0x800127c0`, called from `main`, appears to be the engine's initialization function. Provisionally called it `InitGameEngine_MAYBE`. Among other things, it calls the function at RAM address `0x8001250c`, responsible for reading the file header (conveniently, it fits into one sector). To that end, it it uses the function at `0x80016500` (named this `seek_in_cd_and_do_dma_MAYBE`), which, no surprise, seems to be a high level "seek CD to this address and read via DMA" function. `InitGameEngine_MAYBE` then copies the first 816 bytes, the nonempty portion of the header, to a separate address. ^[1] The game uses this as an index to load other data from the WAD file.

```
void LoadWADIndex(void)

{
  DAT_80076b90 = 0x25;
                    /* function 0x80016500, 0x25 is the minute/second/sector address packed into an int.  */
  seek_in_cd_and_do_dma_MAYBE(0x25,&DAT_overlay_base_addr_PROBABLY,0x800,0,600);
					/* DAT_WAD_index == 0x8007a6d0 */
  memcpy_GUESS(DAT_WAD_index,&DAT_overlay_base_addr_PROBABLY,0x330);
  return;
}
```

The actual DMA happens in `0x80065dbc`, via the chain of calls `seek_in_cd_and_do_dma_MAYBE -> 0x8006606c (CdRead) -> 0x80065dbc`. `0x8006606c` is the `CdRead` function from the CD library.

As you can see in the source code of the tool above, for the most part, the WAD file can be further grouped into pairs of code and data sections.

Immediately after the WAD index is loaded, it's used to get the file offset of the next section, and copy it to another address:

```
/* &DAT_overlay_base_addr_PROBABLY is 0x8007aa38, and overlays get loaded here according to wadtool as well */
seek_in_cd_and_do_dma_MAYBE(DAT_80076b90,&DAT_overlay_base_addr_PROBABLY,0x800,DAT_WAD_index[3].Offset,600);
/* &DAT_section_3_copy_after_dma_MAYBE == 0x80076c00 */
memcpy_GUESS(&DAT_section_3_copy_after_dma_MAYBE,&DAT_overlay_base_addr_PROBABLY,0x1d0);
```

Next, we have some more DMAs:

```
/*
	INT_800755a4 is set to 0x800 in the binary, so this ends up being 0x801bf800
*/
ptr_dma_cdrom_madr_0_MAYBE = (u_long *)(0x801c0000 - INT_800755a4);
/*
	*DAT_section_3_copy_after_dma_MAYBE == 0x800
	Seems like section 3 itself has a header with offset-size pairs,
	and this skips to the start of the data using the first header field
*/
seek_in_cd_and_do_dma_MAYBE(DAT_80076b90,ptr_dma_cdrom_madr_0_MAYBE,0x40000,DAT_section_3_copy_after_dma_MAYBE + DAT_WAD_index[3].Offset,600);
/*
	DAT_WAD_index[1].CodeOverlaySize_MAYBE == 0x3800
*/
seek_in_cd_and_do_dma_MAYBE(DAT_80076b90,&DAT_overlay_base_addr_PROBABLY,DAT_WAD_index[2].Size,DAT_WAD_index[2].Offset,600);
/*
	0x801bf800 - 0x1b000 == 0x801a4800
*/
ptr_dma_cdrom_madr_MAYBE = (void *)((int)ptr_dma_cdrom_madr_0_MAYBE - DAT_WAD_index[0].Size);
seek_in_cd_and_do_dma_MAYBE(DAT_80076b90,ptr_dma_cdrom_madr_MAYBE,DAT_WAD_index[0].Size,DAT_WAD_index[0].Offset,600);
```

So we have our memory map:

```
0x80076c00-0x80076dd0: WAD section 3 header
0x8007a6d0-0x8007a9ff: DAT_WAD_index
0x8007aa38-0x8007e238: WAD section 2
0x801a4800-0x801bf7ff: WAD section 0
0x801bf800-0x805bf800: WAD section 3 data
```

It then displays the SCEA and Universal logos in and out. Interestingly, the opening logos seem to be hardcoded in the executable, and the fadeout effects are done by creating a copy and manipulating the pixel data before using `LoadImage`, `DrawSync` and `VSync` to render it.

Next up, it starts calling a function that incrementally loads items from section 3 again. This part was a bit too complicated, but one interesting thing I noticed when diffing memory and the WAD file, that offsets in "inner" headers sometimes got turned into pointers into RAM.

It appears the format can be nested, as a section can contain "files" with a 0x100 long header. The offsets in these get replaced with the corresponding pointers at runtime.

When loading data overlays, the offset into the disk is computed as `DAT_section_3_copy_after_dma_MAYBE + DAT_WAD_index[DAT_level_index_global * 2 + 10].Offset`. The first offset in the section 3 header is 0x800, the header's size. Seems like the idea here is that all data sections begin with a 2048 byte header, and the loading process skips it to DMA the data directly. During level transitions, the loading is done as sort of a state machine, with each call of the function executing the selected stage, and advancing the stage counter once finished. The second and third fields of WAD section 3's header (`0x80076c04`, `0x80076c08`) are used similarly, added to the offset extracted from the WAD index.

After some more DMAing, we end up with one more section in memory.


```
0x8018b800-0x801c57ff: WAD section 8 header+data
0x801c5bb8-0x801ff7ff: WAD section 8 data
```

Note that this partially overlaps with previously loaded WADs. Some of the sections loaded prior are immediately used to, maybe, play sound effects. More research is needed. Furthermore, eems like there are a few memory areas set aside for overlays, they're loaded as needed, and old data is otherwise left in memory. Inadvertently executing the wrong code/data would explain some erratic behavior and visual corruption that can be experienced when corrupting the main game state variable -- the game expects a particular memory layout at particular times, and the overlays get swapped in some gamestates.

After loading section 8, the game sets some values in RAM that seem game related, seeds the RNG with `1234`, and exits the initialization function, proceeding to the main game loop. Which more or less consists of the main game state machine, updating the pointer into the input ring buffer, and rendering the current frame.

So where are we with this? We've seen `0.wad`, `titlescreen_code.ovl` (WAD section 2), `titlescreen_data.wad` (WAD section 3) and `8.wad` loaded. What section 8 does remains a mystery.

P.S.: If you have your overlays loaded at the correct address in Ghidra, you can adjust existing call references to point at its namespace instead of the main namespace. Very convenient.

-----

^[1] I didn't expect this to be hardcoded.

[Back to index](/breaking-videogames/)