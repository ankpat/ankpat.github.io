---
layout: post
title:  "An Introduction to Use-After-Free Vulnerabilities Class"
categories: memory vulnerability cno exploit
---

The use-after-free (UaF) vulnerability class is a particularly troublesome
vulnerability that can appear in software with a [surprising](https://googleprojectzero.blogspot.com/2019/01/) [amount](https://googleprojectzero.blogspot.com/2019/08/a-very-deep-dive-into-ios-exploit.html) of [regularity](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=use+after+free).

A use after free vulnerability is triggered by a program attempting to access
memory that had been previously allocated for use but has since indicated the
memory is no longer needed.

## Primer on the Program Heap
***Disclaimer: This is a very high level overview of the program heap. Several
important and critical key details have been omitted with a healthy amount of
handwaving in the interest of time to get to the core of UaF***.

When a program runs on a computer, there are two important components that it needs:
1. Instructions to tell the processor what to do; and
2. Data that is operated upon

A program provides instructions to the processor which can then manipulate
data. The processor can only access data in memory, so a program must move any data
that needs to be manipulated into memory.

If a program knows exactly how much memory its data will require, the memory can be
*statically* allocated when the program executes.
For example, if a program randomly generates two integers and stores the result
in a third, then the memory
requirements are known ahead of time - the program will need storage for two
integers and their result.

*Dynamic* memory allocation is used when the program does not know how much
memory will be needed ahead of time. For example, a program that performs modifications
of files on disk must first read the file into memory prior to manipulating its
contents. Since files tend to vary in size, a program will need to *dynamically*
allocate space for the data. These *dynamic* memory allocations occur on the **heap**.
The heap offers a pool of memory that a program may request during its execution.
When a program makes such a request, the system, if it can do so, will reserve a
block of memory for the program and tell the program where this reserved block
is located.

The program can then use that block of memory as it wishes - it can read and write
to that memory as it sees fit. Once the program no longer needs that block of memory,
it frees the reserved memory. The system then reclaims that block of memory
and can hand it out the next time the program requests a memory allocation.

A few questions present themselves from the high-level overview:
1. How does the system know which block of memory to hand out?
2. How does a program request memory?
3. How does a program return memory?


### Understanding Heap Allocations
Heaps consist of blocks of memory that can be handed out to programs upon
request. Modern heaps consist of blocks of varying sizes which are grouped into
*zones*. For example, a heap could have a zone for blocks of memory that are
under 17 bytes in size and a zone for blocks of memory that are at least 17
bytes in size but smaller than 1,024 bytes. This allows the system optimize memory
allocation - if a program needs 100 8-byte allocations, it could allocate
100 elements from the 16-byte zone and use 1,600 bytes total. If all heap blocks
were 1,024 bytes in size, then 100 8-byte allocations would use up 102,400 bytes
of memory!

In order to keep track of what memory has been allocated to the program and
what is still available, the system maintains a heap *free list* which tracks
which blocks of the heap are unallocated. When an allocation is requested, the
system checks the free list for available blocks. If blocks are available,
the system can remove the block from the free list and provide the address
of the block to the program.

Once the program is done using the block of memory, it can notify the system
that is relinquishing control of it. The system will then
add the block back to the free list so it can be used again in the future.

Consider the following 8-byte heap zone that has no allocations:
<table>
<tr><th>Heap</th><th>Free List</th></tr>
<tr>
<td>
<table>
<tr><td>Address</td><td colspan="4"></td></tr>
<tr>
  <td>0x00</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
</tr>
<tr>
<td>0x20</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
</tr>
</table>
</td>
  <td>
    <ul>
    <li>0x00</li>
    <li>0x08</li>
    <li>0x10</li>
    <li>0x18</li>
    <li>0x20</li>
    <li>0x28</li>
    <li>0x30</li>
    <li>0x38</li>
    </ul>
  </td>
</tr>
</table>

Each element is 8 bytes in size.
Because there are no allocations, the free list contains every single block in
the heap zone. For the sake of efficiency, the free list is typically, but not always,
a Last-in-First-Out (or First-In-Last-Out) structure so that the most recently
added element to the free list is the next candidate when an allocation is requested.

### Requesting and Releasing Memory
A program can request memory from the system by calling `malloc` and related
allocation functions. The `malloc` function takes a single argument, how much
memory is being requested. If the system can find enough memory to satisfy the request,
it will tell the program where this memory is by returning the *address*
of the memory from `malloc`. The following code snippet will request the
system allocate enough space for an integer. If the system can find this space on
the heap, it will return the address of this memory and store it in `p_flag`.
```c++
int* p_flag = malloc(sizeof(int));
```
Using the heap illustration above, the last entry in the free list would be 0x38.
The entry is removed from the free list once it is allocated and can be passed
back to the program.
<table>
<tr><th>Heap</th><th>Free List</th></tr>
<tr>
<td>
<table>
<tr><td>Address</td><td colspan="4"></td></tr>
<tr>
  <td>0x00</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
</tr>
<tr>
<td>0x20</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#fff">p_flag</td>
</tr>
</table>
</td>
  <td>
    <ul>
    <li>0x00</li>
    <li>0x08</li>
    <li>0x10</li>
    <li>0x18</li>
    <li>0x20</li>
    <li>0x28</li>
    <li>0x30</li>
    </ul>
  </td>
</tr>
</table>

Once the program receives this address, it can do whatever it wishes with the
memory. It can read from it, write to it, and share the memory with other functions.

Memory, while plentiful, is still finite. A system can run out of heap memory if
the program requests lots of memory and never tells the system it is finished
using it. In order for a program to tell the system that it is done with memory,
it calls `free` on the allocated memory's address. This tells the system
that the memory is no longer in use and can be given out in the future if the program
requests more memory. A program that allocated `p_flag` above would free it as follows:
```c++
free(p_flag);
```
This `free` operation will return the heap to its previous clean state,
assuming no additional allocations were made in between calling `malloc` and `free`.
<table>
<tr><th>Heap</th><th>Free List</th></tr>
<tr>
<td>
<table>
<tr><td>Address</td><td colspan="4"></td></tr>
<tr>
  <td>0x00</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
  <td style="background-color:#555">&nbsp;</td>
</tr>
<tr>
<td>0x20</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
<td style="background-color:#555">&nbsp;</td>
</tr>
</table>
</td>
  <td>
    <ul>
    <li>0x00</li>
    <li>0x08</li>
    <li>0x10</li>
    <li>0x18</li>
    <li>0x20</li>
    <li>0x28</li>
    <li>0x30</li>
    <li>0x38</li>
    </ul>
  </td>
</tr>
</table>

## Understanding Use-After-Free

To highlight how a UaF vulnerability can be triggered, take a look at the listing below:
{% highlight c++ linenos %}
/** clang uaf.c -o uaf */
#include <stdlib.h>
#include <stdio.h>

struct foo
{
    int bar;
};

int main()
{
    struct foo* p_foo;
    int* p_flag = malloc(sizeof(int));

    *p_flag = 0;
    free(p_flag);

    p_foo = malloc(sizeof(struct foo));

    p_foo->bar = 0x1337;

    if (*p_flag == 0x1337)
    {
        printf("Use-After-Free in action! Value of p_flag: 0x%x\n", *p_flag);
    }
    else
    {
        printf("p_flag was not 0x1337!\n");
    }
}

{% endhighlight %}

This program declares a structure `foo` which consists of a single integer, `bar`.

When this program runs, it will request space for an integer from the heap on line 13.
If the system will consult the heap free list and provide the next available
block of memory that could hold an integer. The program will take the address of
this memory and store it in `p_flag`, which is a pointer to an integer.
This pointer to an integer is then used to write 0 to the memory location that it points to.

At this point, the program indicates that it is done with the memory referenced by
`p_flag` and calls `free`. The system takes that block of memory and adds it back
to the free list.

The program then requests space for `struct foo` which contains only one integer.
On success, the address of the available memory is stored in `p_foo`. The `bar`
element inside of `p_foo` is then assigned a value.

Then the value of the memory referenced by `p_flag` is checked against 0x1337.

Quite obviously, this is a use-after-free vulnerability. `p_flag` was freed on line 16,
but is used again on line 22.

It turns out, the system allocated the same block of memory for `p_flag` as it
did for `p_foo`. Since the program called free and told the system that the
memory being used for `p_flag` was no longer needed, the system added it back to
the free list. Then, when the program requested space for a `struct foo`, the
system consulted the free list and found the block of memory that has just been
freed for `p_flag` and gave it to the program.

## Detecting Use-After-Free
While this vulnerability class can be particularly dangerous, it has become the focus
of several tools to detect and identify this vulnerability class.

An easy one to use is the [Address Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer].
The Address Sanitizer allows detection of common vulnerability classes related to
memory errors in programs including UaF vulnerabilities.

Try enabling the Address Sanitizer when compiling the program above and running it:
```
$> clang uaf.c -o uaf -g -fsanitize=address
$> ./uaf
=================================================================
==6403==ERROR: AddressSanitizer: heap-use-after-free on address 0x6020000000d0 at pc 0x0001040acd66 bp 0x7ffeebb53a20 sp 0x7ffeebb53a18
READ of size 4 at 0x6020000000d0 thread T0
    #0 0x1040acd65 in main uaf.c:22
    #1 0x7fff6c0bf7fc in start (libdyld.dylib:x86_64+0x1a7fc)

0x6020000000d0 is located 0 bytes inside of 4-byte region [0x6020000000d0,0x6020000000d4)
freed by thread T0 here:
    #0 0x10411194d in wrap_free (libclang_rt.asan_osx_dynamic.dylib:x86_64h+0x6194d)
    #1 0x1040accb0 in main uaf.c:16
    #2 0x7fff6c0bf7fc in start (libdyld.dylib:x86_64+0x1a7fc)

previously allocated by thread T0 here:
    #0 0x104111793 in wrap_malloc (libclang_rt.asan_osx_dynamic.dylib:x86_64h+0x61793)
    #1 0x1040acc48 in main uaf.c:13
    #2 0x7fff6c0bf7fc in start (libdyld.dylib:x86_64+0x1a7fc)

```

There's a lot of data in the output and it has been truncated above.
What matters is that the address sanitizer has identified
that a UaF has occurred because a block of memory that was allocated
on line 13 and freed on line 16 was dereferenced and read from on line 22.

## Preventing UaF

### Prevention with Code
While the Address Sanitizer can detect UaF at runtime, it is preferable to avoid
having a situation where a UaF can arise in the first place. Take a look at the
following modification of the program above:
{% highlight c++ linenos %}
/** clang uaf.c -o uaf -g -fsanitize=address*/
#include <stdlib.h>
#include <stdio.h>

struct foo
{
    int bar;
};

int main()
{
    struct foo* p_foo;
    int* p_flag = malloc(sizeof(int));

    *p_flag = 0;
    free(p_flag);
    p_flag = NULL;

    p_foo = malloc(sizeof(struct foo));

    p_foo->bar = 0x1337;

    if (p_flag && *p_flag == 0x1337)
    {
        printf("Use-After-Free in action! Value of p_flag: 0x%x\n", *p_flag);
    }
    else
    {
        printf("p_flag was not 0x1337!\n");
    }
}
{% endhighlight %}

There are two changes that have been introduced in this modification. The first
is on line 17 and assigns `p_flag` to NULL after it has been freed. This removes
the dangling reference to the heap memory block that was previously held by
`p_flag` after calling free. Any attempt to access the memory referenced by `p_flag`
will cause the system to attempt to dereference address 0 which will result in crash.

Additionally, the use of `p_flag` later on line 23 include an explicit check for
`p_flag` being a non-null value. Only if `p_flag` is non-null will the program
attempt to dereference it.

This works well in this one example, but becomes increasingly more difficult
as the complexity of the code grows.

### Advanced UaF Mitigations
These will be covered in depth in future posts, but some will be mentioned for now:
* Free List Randomization
* Delay Free
