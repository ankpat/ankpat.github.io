---
layout: post
title:  "Stack Buffer Overflows Explained"
categories: vulnerability vulnerability-analysis buffer-overflow exploitation
---

A stack buffer overflow is a specific type of vulnerability that involves coercing an application to write more data than it was expecting into a specific memory location.

Terminology
-----------

*Buffer* - An area of memory a program has reserved for some type of data. In C, the following can be considered buffers:

{% highlight c++ %}
// A buffer containing 4 bytes
char buf[4];

// A buffer containing one unsigned 32-bit integer (4 bytes)
uint32_t integerValue;

{% endhighlight %}

The first line is a character array of four elements.
The second line is a 32-bit integer value.
Both of these will be backed by some sort of memory when a program is run as illustrated below.
The first four bytes are reserved for the ***buf*** and the next four are for ***integerValue***.

<table style="text-align:center">
    <thead>
        <tr class="header">
            <th></th>
            <th>0x0</th>
            <th>0x1</th>
            <th>0x2</th>
            <th>0x3</th>
            <th>0x4</th>
            <th>0x5</th>
            <th>0x6</th>
            <th>0x7</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th>0x0</th>
            <td colspan="4" style="background-color:#afa">buf</td>
            <td colspan="4" style="background-color:#aaf">integerValue</td>
        </tr>
    </tbody>
</table>


*Stack* - Scratch space used by a program to track the flow of execution, function arguments, and local variables.


Program Memory
--------------

Typically, program memory is split into a few different parts, each with their own purpose.
Regardless of architecture or operating system, it is common for there to be regions of memory containing:
* All code that will be executed to run the program, 
* All of the initialized global variables. 
* The Program Stack (scratch space including local variables program state, function arguments, and local variables)
* Memory that is dynamically allocated when requested by the program (such as when the program needs to create a new buffer to store data of variable size like a file)

The regions and how they used vary between operating systems and architectures, but the four above are very generic descriptions of some common ones.
