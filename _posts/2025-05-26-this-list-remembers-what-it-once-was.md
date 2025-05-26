---
layout: default
title: "this-list-remembers-what-it-once-was"
date: 2025-05-26 02:00:00 -0000
tags: disgaea display_lists
---

Sometimes discoveries can be written into a neat narrative. Unfortunately, sometimes I forget to note how I found something, until later it turns out to be significant. So this post is going to be more of an infodump.^[1]

Sometime these last few months of reversing Disgaea: AoD, I determined that a particular address holds a display list. I didn't make much headway with using it for further analysis, so I moved on and forgot about it. Conveniently, Ghidra sometimes renames locals in accordance to parameters of a function they get passed to. Meaning the display_list identifier got propagated outwards to callers/callees, which allowed locating some other code that worked with display lists. If I had to guess how I found it, PPSSPP knows about most of the syscalls in the PSP's API, including `sceGeListEnqueue()`. Which also happens to have a parameter known to represent a display list. This is called by the function at `0x088049b8`, which passes one of its arguments into it with some extra steps. I now have a pretty good guess as to what this is, but we'll get back to that.

One of the extra steps is filling out a structure I've named DisplayListInfo. The passed-in value is saved to two different fields in it, which are then passed to `sceGeListEnqueue()` as `list_ptr` and `stall_ptr`.

Shortly after, I also found a global at `0x088e90c4`, that seems to be an array of these structs. `0x088e90c0` also holds a pointer to one of the array elements, and it is referenced by some of these functions that I knew either worked on display lists, or called functions that did.

Eventually a pattern emerged: you'd have these functions that get a `DisplayListInfo` pointer, get the current stall pointer (essentially the bottom of the list, as far as I can tell), do a bunch of bit-shifting, ORs and ANDs on the other arguments to the function, write the result to `stall_ptr`, and increment it by a multiple of 4. The MSB of the data written tended to be a constant, with different functions using different constants. Alternatively, a function would get the address at `0x088e90c0`, and call one of the functions of the previous type, passing it in as a `DisplayListInfo` pointer.

So obviously, `list_ptr` and `stall_ptr` are pointers into the actual command buffer for the GE, the constants are GE command opcodes, and the rest of the data is the parameters required by the opcodes. By the way, most GE "instructions"/display list commands fit into 4 bytes, but there are some quirky decisions. Such as, setting a world/view/projection matrix requires executing the same command 4 times. Whereas the dither matrix (which uses a different data type) has four different commands for its 4 rows.

I ended up going through the functions that referenced `0x088e90c0`, and documenting the GE opcodes in the function(s) they called, as well as naming them after the opcodes they used. This became somewhat inconvenient when it came to functions doing several commands. (Also, the GE commands' parameterization gave clues about the meaning of various arguments to these functions.) I'd go into more detail, but... we're about to get back to that too. There is a compelling reason not to name them things like `insert_alpha_test_enable_in_current_display_list`.

One of the more complex functions was using the `BASE` (Base Address Register) opcode, which made me wish I knew more about the specifics of the command set. So I headed over to the (unofficial) PSPSDK source code, and searched around for any uses of the command. What I found is the compelling reasons not to list these functions one by one: the decompile I was looking at turned out to be almost identical to their implementation of [sceGuDrawArray](https://github.com/pspdev/pspsdk/blob/85e3577cb5ce86709e0c1a6e07061961e8411c79/src/gu/sceGuDrawArray.c).

For comparison, here is the decompile:

```
void sceGuDrawArray_PROBABLY(int primitive_type,int vertex_type,int count,void *iaddr_ptr,void *vaddr_ptr)

{
  DisplayListInfo *pDVar1;
  uint *puVar2;

  pDVar1 = current_display_list_info_pointer;

  if (vertex_type != 0) {
    puVar2 = (uint *)current_display_list_info_pointer->stall_ptr;
                    /* 0x12 VTYPE - Vertex Type */
    *puVar2 = vertex_type & 0xffffffU | 0x12000000;
    pDVar1->stall_ptr = puVar2 + 1;
  }

  if (indices != (void *)0x0) {
    puVar2 = (uint *)pDVar1->stall_ptr;
                    /* 0x10 BASE - Base Address Register */
                    /* 0x2 IADDR - Index List (BASE) */
    *puVar2 = ((uint)((int)iaddr_ptr << 4) >> 0x1c) << 0x10 | 0x10000000;
    pDVar1->stall_ptr = puVar2 + 2;
    puVar2[1] = (uint)iaddr_ptr & 0xffffff | 0x2000000;
  }

  if (vertices != (void *)0x0) {
    puVar2 = (uint *)pDVar1->stall_ptr;
                    /* 0x10 BASE - Base Address Register */
                    /* 0x1 VADDR - Vertex List (BASE) */
    *puVar2 = ((uint)((int)vaddr_ptr << 4) >> 0x1c) << 0x10 | 0x10000000;
    pDVar1->stall_ptr = puVar2 + 2;
    puVar2[1] = (uint)vaddr_ptr & 0xffffff | 0x1000000;
  }

  puVar2 = (uint *)pDVar1->stall_ptr;
  /* 0x04 PRIM - Primitive Kick */
  *puVar2 = (primitive_type & 7U) << 0x10 | count | 0x4000000;
  pDVar1->stall_ptr = puVar2 + 1;
  update_stall_address_in_ge_for_current_display_list();
  return;
}

```

I think some differences are partially attributable to inlining. Otherwise, it sends the same sequence of `VTYPE`, `BASE`, `IADDR`, `BASE`, `VADDR`, `PRIM` commands as the SDK function. So it turned out that these display list wrangling functions are actually just the graphics utility library (gu, sceGu\*). Thus, listing them under a bespoke name would've been pointless. I haven't fully mapped every function to the names used by the GU API, but I could guess the unnamed function mentioned earlier in the post. Based on the commands queued, and the calls to `sceKernelCpuSuspendIntr()`, the function at `0x088049b8` is likely to be `sceGuStart`, though it takes an extra parameter compared to the one in the unofficial SDK.

Anyway, starting from the bottom up was interesting here. Had I known beforehand this was the GU library, I probably would've skipped out on looking at the GE commands until much later. I didn't expect the functions essentially being thin wrappers over display list/GE commands. Though I also don't know if e.g. OpenGL worked in a similar way, or whether being on a platform with fixed hardware eliminated most of what would go behind the graphics API on a more flexible platform.

By the way, the title is a reference to [This Dust Remembers What It Once Was](https://www.beneaththewaves.net/Software/This_Dust_Remembers_What_It_Once_Was.html), a reverse engineering tool by Ben Lincoln. I haven't yet used it, but appreciate the unique naming scheme.

-----

^[1] I'm tempted to say it's not as important to drag readers through the exact same journey of discovery.

-----

[Back to index](/breaking-videogames/)