---
layout: post
title:  "Finding Memory Leaks on macOS"
categories: macos debugging memory 
---

Long story short, just use [Valgrind](http://valgrind.org/).

The Apple developer documentation has a full section devoted to describing [Finding Memory Leaks](https://developer.apple.com/library/content/documentation/Performance/Conceptual/ManagingMemory/Articles/FindingLeaks.html).
The *leaks* tool described in the documentation can provide information about programs to help identify and fix memory leaks.
For each memory leak discovered, the *leaks* tool prints:
* the address
* the number of leaked bytes
* the contents of the leaked buffer

Consider the simple program:

{% gist 744b3009f567254207015e599ab4a056 %}

Quite obviously, there is a memory leak - space for 512 integers is allocated using *malloc* but never freed before the program terminates.

Assuming XCode is installed and clang is available for use, the following command can be used to compile the code above.
{% highlight bash %} 
clang leaky.c -g -o leaky
{% endhighlight %}

Like most memory leaks, nothing special happens when the program is run. However, running the program with the leaks tool gives us some important information.

    $> leaks -atExit -- ./leaky
    Process:         leak [28373]
    Path:            /Development/ankpat.github.io/code/leaky/leak
    Load Address:    0x10895f000
    Identifier:      leak
    Version:         ???
    Code Type:       X86-64
    Parent Process:  leaks [28372]
    
    Date/Time:       2018-05-22 23:42:15.891 -0400
    Launch Time:     2018-05-22 23:42:15.701 -0400
    OS Version:      Mac OS X 10.13.4 (17E202)
    Report Version:  7
    Analysis Tool:   /Applications/Xcode.app/Contents/Developer/usr/bin/leaks
    Analysis Tool Version:  Xcode 9.3.1 (9E501)
    
    Physical footprint:         376K
    Physical footprint (peak):  376K
    ----
    
    leaks Report Version: 3.0
    Process 28551: 158 nodes malloced for 15 KB
    Process 28551: 1 leak for 2048 total leaked bytes.
    
    Leak: 0x7f8e8f001000  size=2048  zone: DefaultMallocZone_0x10b770000
    	0x00000000 0x00000001 0x00000002 0x00000003 	................
    	0x00000004 0x00000005 0x00000006 0x00000007 	................
    	0x00000008 0x00000009 0x0000000a 0x0000000b 	................
    	0x0000000c 0x0000000d 0x0000000e 0x0000000f 	................
    	0x00000010 0x00000011 0x00000012 0x00000013 	................
    	0x00000014 0x00000015 0x00000016 0x00000017 	................
    	0x00000018 0x00000019 0x0000001a 0x0000001b 	................
    	0x0000001c 0x0000001d 0x0000001e 0x0000001f 	................
    	...
    

From the leaks Report summary, 2048 bytes have leaked from the default [malloc zone](https://developer.apple.com/library/content/documentation/Performance/Conceptual/ManagingMemory/Articles/MemoryAlloc.html).

The *-hex* parameter given to the leaks tool provides a small hex dump of the leaked memory to help with debugging.
In this example, the buffer contains sequential integers starting at 0, which indicates it is probably the allocated array.

Unfortunately, leaks does not provide much else to help find the source of the leak.
When writing Linux software, I typically use [Valgrind](http://valgrind.org/) to detect memory leaks such as buffers that were not freed.
While it is not shipped as part of XCode, it is trivial to build.

Valgrind uses the debugging information built into the binary (by way of the *-g* flag to clang), to help identify where memory allocations occur.

    $> /opt/valgrind/bin/valgrind --leak-check=full ./leaky
    ==28585== Memcheck, a memory error detector
    ==28585== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
    ==28585== Using Valgrind-3.14.0.GIT and LibVEX; rerun with -h for copyright info
    ==28585== Command: ./leaky
    ==28585== 
    ==28585== 
    ==28585== HEAP SUMMARY:
    ==28585==     in use at exit: 20,254 bytes in 164 blocks
    ==28585==   total heap usage: 185 allocs, 21 frees, 28,702 bytes allocated
    ==28585== 
    ==28585== 72 bytes in 3 blocks are possibly lost in loss record 26 of 43
    ==28585==    at 0x1000ABFF2: calloc (vg_replace_malloc.c:714)
    ==28585==    by 0x10075A7E2: map_images_nolock (in /usr/lib/libobjc.A.dylib)
    ==28585==    by 0x10076D7DA: objc_object::sidetable_retainCount() (in /usr/lib/libobjc.A.dylib)
    ==28585==    by 0x100007C64: dyld::notifyBatchPartial(dyld_image_states, bool, char const* (*)(dyld_image_states, unsigned int, dyld_image_info const*), bool, bool) (in /usr/lib/dyld)
    ==28585==    by 0x100007E39: dyld::registerObjCNotifiers(void (*)(unsigned int, char const* const*, mach_header const* const*), void (*)(char const*, mach_header const*), void (*)(char const*, mach_header const*)) (in /usr/lib/dyld)
    ==28585==    by 0x10022571D: _dyld_objc_notify_register (in /usr/lib/system/libdyld.dylib)
    ==28585==    by 0x10075A075: _objc_init (in /usr/lib/libobjc.A.dylib)
    ==28585==    by 0x1001AFB84: _os_object_init (in /usr/lib/system/libdispatch.dylib)
    ==28585==    by 0x1001AFB6B: libdispatch_init (in /usr/lib/system/libdispatch.dylib)
    ==28585==    by 0x1000BE9C2: libSystem_initializer (in /usr/lib/libSystem.B.dylib)
    ==28585==    by 0x100019A79: ImageLoaderMachO::doModInitFunctions(ImageLoader::LinkContext const&) (in /usr/lib/dyld)
    ==28585==    by 0x100019CA9: ImageLoaderMachO::doInitialization(ImageLoader::LinkContext const&) (in /usr/lib/dyld)
    ==28585== 
    ==28585== 2,048 bytes in 1 blocks are definitely lost in loss record 38 of 43
    ==28585==    at 0x1000AB545: malloc (vg_replace_malloc.c:302)
    ==28585==    by 0x100000EEC: main (leaky.c:9)
    ==28585== 
    ==28585== LEAK SUMMARY:
    ==28585==    definitely lost: 2,048 bytes in 1 blocks
    ==28585==    indirectly lost: 0 bytes in 0 blocks
    ==28585==      possibly lost: 72 bytes in 3 blocks
    ==28585==    still reachable: 200 bytes in 6 blocks
    ==28585==         suppressed: 17,934 bytes in 154 blocks
    ==28585== Reachable blocks (those to which a pointer was found) are not shown.
    ==28585== To see them, rerun with: --leak-check=full --show-leak-kinds=all
    ==28585== 
    ==28585== For counts of detected and suppressed errors, rerun with: -v
    ==28585== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 12 from 12)

The relevant section for this example is right before the **LEAK SUMMARY**. 2048 bytes are lost (leaked) due to an allocation at line 9 of ***leaky.c***.

While the *leaks* tool could indicate that there is a leak somewhere, *valgrind* is able to indicate where leaked memory was allocated.

As part of a typical CI workflow, or even local builds without CI, running leak checking tools can be incredibly useful for preventing memory issues before they get a chance to surface.
