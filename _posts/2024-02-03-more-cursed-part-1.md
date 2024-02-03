---
layout: default
title: "more-cursed-part-1"
date: 2024-02-03 17:00:00 -0000
tags: vtmb console cursed python
---

Today, I had an idea for debugging the crash from the previous post. I said:

> I would know more if I found out how to enable the logs it's writing,

But I remembered there's an even easier way. There's a tool called [API Monitor](https://www.rohitab.com/apimonitor), that lets you capture API calls made to DLLs, even supporting automatic extraction of the API. Shoutout to Buherátor, who years ago introduced me to this tool.

[Archive](https://web.archive.org/web/\*/https://www.rohitab.com/apimonitor), in case your browser doesn't support TLS below 1.2.

This paid off immediately, behold:

![](/breaking-videogames/assets/missing_dll.png)

So it turns out the cause of the crash was simple user error: I moved the DLL to the reverse engineering folder, instead of copying and pasting. Whoops. I still think it's slightly cursed that the game engine keeps going even when the DLL load fails, only truly crashing when it tries to access the interface created from it. But maybe this is standard practice.

----

Moving on to the next cursed thing: one of the problems with the Python REPL is that it's very prone to causing the game to lock up. For example, executing `locals()` in the console while a game is loaded will freeze the Python thread.

![](/breaking-videogames/assets/deadlock.jpg)

Looking at ApiMon again, we can see a bunch of `fwrite` calls, until the thread comes to a halt on a final `NtWriteFile()` that hangs. I suspected the problem is with writing large amounts of text to the console, since it executes just fine in the main menu, where `locals()` returns a much smaller dictionary:

![](/breaking-videogames/assets/working_locals.jpg)

Investigating in API Monitor again:

![](/breaking-videogames/assets/deadlock_apimon.jpg)

We can see that it's trying to write the text representation of the locals dictionary to a file (presumably) representing stdout. In this case, the source console. I thought it was due to the large buffer being written by `NtWriteFile()`, but prior to that, there is a successful write to the same file, also with 4096 bytes. The difference between those two calls is that in the latter case `fwrite()` gets 4603 bytes to write, a larger number than 4096, while the other one gets 43.

![](/breaking-videogames/assets/working_fwrite.jpg)

Maybe that's relevant and the source of the deadlock is somewhere in the buffering logic. This is about where I gave up on debugging this issue.

----

The scope of the next one is a fair bit larger, and I'm still verifying the details. So I decided to split things up and post this part early. If I'm correct about the next one, it's going to be more cursed than all these put together. Stay tuned.


[Back to index](/breaking-videogames/)