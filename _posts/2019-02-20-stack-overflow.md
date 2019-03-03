---
layout: post
title:  "Stack Buffer Overflows Explained"
categories: vulnerability vulnerability-analysis buffer-overflow exploitation
---

A stack buffer overflow is a specific type of vulnerability that involves coercing an application to write more data than it was expecting into a specific memory location.

## Terminology
<details markdown="1">
<summary>Definitions and explanations of terms terms used in this article.</summary>
### Buffer
<details markdown="1">
<summary>
An area of memory a program has reserved for some type of data.
</summary>

In C, the following can be considered buffers:

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

</details>

### Stack

<details markdown="1">
<summary>Scratch space used by a program to track the flow of execution, function arguments, and local variables.</summary>

Consider the following example on a 32-bit system. (32-bit refers to the standard size of a memory block, or word).

{% highlight c++ %}
1. int main(int argc, char** argv)
2. {
3.     char* programName = NULL;
4. 
5.     programName = strdup(argv[0]);
6. 
7.     return 0;
8. }
{% endhighlight %}

<table style="text-align:center">
    <thead>
        <tr class="header">
            <th ></th>
            <th colspan="3">Line</th>
        </tr>
        <tr class="header">
            <th>Stack Address</th>
            <th>3</th>
            <th>5</th>
            <th>6</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>0x10</td>
            <td></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td>0x14</td>
            <td></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td>0x18</td>
            <td></td>
            <td>Address of Line 6</td>
            <td></td>
        </tr>
        <tr>
            <td>0x1c</td>
            <td></td>
            <td>Address of argv[0]</td>
            <td></td>
        </tr>
        <tr>
            <td>0x20</td>
            <td>0x00000000</td>
            <td>0x00000000</td>
            <td>Address of Copy of argv[0]</td>
        </tr>
    </tbody>
</table>

In the example above, the C code is a mostly useless program that calls *strdup* on the first argument to the function (*argv[0]*).
The convention for these sorts of programs is that the first argument is the name of the file that was run,
so if the code was compiled into the executable file **foo.exe**, the first argument would be *foo.exe*

The table shows the line number of execution and the what the program stack looks like at the end of each line.

Starting at Line 3, the program reserves space on the stack for a local variable, *programName*.
This is a pointer so it requires 32-bits, or 4 bytes, be reserved to store its value.

Line 5 assigns the result of *strdup* to *programName*.
The function *strdup* is given a pointer to *arg[0]* which contains the name of the program.
The argument to the function is pushed onto the stack first and, thus, appears immediately above the space reserved for the local variable *programName*.
Once the arguments are all in place for the call to the function, the program saves the *return address*.

The *return address* is where the program will resume executing inside the current function when it returns from executing the called function.
In this case, once *strdup* is finished, the program will continue executing at Line 6.
</details>
</details>
## Program Memory

Typically, program memory is split into a few different parts, each with their own purpose.
Regardless of architecture or operating system, it is common for there to be regions of memory containing:
* All code that will be executed to run the program, 
* All of the initialized global variables. 
* The Program Stack (scratch space including local variables program state, function arguments, and local variables)
* Memory that is dynamically allocated when requested by the program (such as when the program needs to create a new buffer to store data of variable size like a file)

The regions and how they used vary between operating systems and architectures, but the four above are very generic descriptions of some common ones.
