---
layout: default
title: "moreoverlays"
date: 2026-06-04 23:30:00 -0000
tags: psx iso lba bin cue
---

Currently testing out a workflow for identifying PlayStation overlays containing executable code, using reverse engineering tools and a CD image browser.

- Step 1: Set a breakpoint on a call to a function located in an overlay. Ideally, you want the function at the lowest address belonging to that overlay. In Ghidra, overlay areas generally show up as undefined, so you can just scroll up to check if there is another function label before the one you've found with static analysis. (Since auto-analysis will label these via calls from the main binary.)
- Step 2: Inspect memory at runtime when the breakpoint is hit. Grab a bunch of bytes at the function's address.
- Step 3: Open the .bin disc image in your favorite hex editor and search for the bytes you've just grabbed. If you're lucky, the start doesn't contain anything like offsets that need to be patched up at runtime. I've had to search for about 18 bytes before I got an unambiguous match.
- Step 4: Scroll up/seek backwards in hex editor. Find the sync pattern `00 FF FF FF FF FF FF FF FF FF FF 00`.
- Step 5.1: Note the next 4 bytes. They are going to contain the BCD-encoded timecode for the sector, and the sector mode. The order of bytes is `MM SS FF XX`, where `MM` is the minutes, `SS` is the seconds, `FF` is the frames and `XX` is the code of the sector mode. You can convert this to LBA using the formula `(MM * 60 * 75) + (SS * 75) + FF`. Again, be sure to convert from BCD before doing the math.
- Step 5.2: Alternatively, divide the starting offset of the sync pattern by the CD sector size, `2352`.
- Step 6: The two values may disagree, but this is not necessarily a problem. If the LBAs from timecodes and the LBAs derived from the file offset differ by a consistent amount, that's the number of lead-in sectors at the start of the disc. The LBA derived from timecodes will include the lead-in sectors, whereas the file offset-derived LBA will not. Depending the tools you're using, you might be seeing one or the other. You can also use this difference between LBAs to convert between time codes and file offsets: `(LBA_timecode - LBA_diff) * 2352 = file_offset`

If you have an overlay loaded in memory, but want to locate it on the disc, this procedure will let you map that overlay in memory to an LBA. Conversely, the PlayStation API relies on timecodes to seek the disc, and it seems developers generally ignore file names entirely, so getting the `CdlLOC` timecode given to `CdControl(CdSeekP, ...)` can tell you where to look in the image. Done, right? Not really. For extracting the sectors directly, you'll need the file's size, which can be extracted from the TOC. I honestly recommend trying to string search for extensions like ".STR", ".TIM", ".BIN" instead of trying to guess where it is. The TOC appears to be in the [ISO9660 format](https://wiki.osdev.org/ISO_9660#Directories). But for once, I didn't create extra work for myself and used a CD image browser.

- Step 7: Opening the .bin file in e.g. CDMage lets you see the LBAs associated with particular files in the directory tree. Sadly, there doesn't seem to be a way to jump to an LBA, but with a bit of luck, the folder structure will roughly correspond to the data layout on the disc. You can extract the file in question directly. On Linux, you may need to convert .bin/.cue to .iso using `bchunk`, get the TOC with `isoinfo -l -i image.iso`, then extract with `iso-read`.^[1]

Side note, in CDMage, you can verify the difference between the two LBAs. Go to `Edit -> Data Track Properties` and see the `Misc info` tab. The difference between the `Total disc time` and `Size (M:S:F)` should correspond to the difference, when converted using the `(MM * 60 * 75) + (SS * 75) + FF` formula.

Limitations: This assumes that the disc is a single-track, single-session disc. If you have multiple tracks, I'm not sure the file offsets and the LBAs will stay lined up. I think the PSX API doesn't support multiple tracks, so it should be fine for the purpose of locating overlays.

----

^[1] Sleight of hand warning: I don't think it's guaranteed that the entire file is mapped as an overlay, but once you've moved to the file abstraction from raw byte chunks, you can just get away with mapping the entire file into a separate address space in Ghidra. Similarly, the file may contain a header before the actual code, so you'll need to line the function prologue in the file up with the address it's supposed to be at.

[Back to index](/breaking-videogames/)
