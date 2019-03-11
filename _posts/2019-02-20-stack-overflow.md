---
layout: post
title:  "Stack Buffer Overflows Explained"
categories: vulnerability vulnerability-analysis buffer-overflow exploitation
---

A stack buffer overflow is a specific type of vulnerability that involves coercing an application to write more data than it was expecting into a specific memory location.

At its core, a stack buffer overflow consists of forcing an application to write more data than it expected to write to a specific location.
As part of that write beyond expected bounds (overflow), an attacker controls the data being written in hopes of altering program state or flow.

Consider a champagne glass.
It has a small, finite capacity of champagne that it can hold.
Typically, when someone wants to drink out of a champagne glass, they fill the glass to almost full and then slowly drink from the glass.
The person pouring the champagne knows that they need to fill the glass with champagne until it is almost completely full so they do not overflow and spill the bubbly goodness.    


Now consider what happens when several champagne glasses are stacked in the form of an iconic champagne tower:

{% include image.html url="/assets/Champagne_tower.jpg" description="[ori2uru](https://www.flickr.com/photos/25035545@N04) [CC BY 2.0](https://creativecommons.org/licenses/by/2.0)" %}

Once again, each champagne glass has a finite capacity of champagne that it can hold.
While it is possible to fill each glass individually, there is a much more fun and entertaining way to do it!

1. Pour champagne in the glass at the top of the tower.
2. Fill the top most glass with champagne.
3. Don't stop pouring champagne into the top most glass until the glasses at the bottom of the tower are full.

By exploiting the finite capacity of the champagne glass and the structure of the system, the champagne tower, it is possible to cause a subcomponent of the system to perform in a way you would like that it was not necessarily designed for.
In this case, the top-most glass ends up serving a purpose other than holding its capacity of champagne.

This, in essence, is what a ***stack buffer overflow vulnerability*** takes advantage of within a computer program.
Somewhere, there is a resource with a finite capacity that an attacker can fill beyond its capacity to make a component of the software system behave in a way that it had not been designed to do.

## Program Memory

Typically, program memory is split into a few different parts, each with their own purpose.
For the sake of simplicity, this article will leave out some details up front.
While a real program that runs on most computer systems contains several regions of memory, we will limit our discussion to the following:

### Executable Instructions
This is typically referred to as the *text* section.
The actual executable program code. This consists of the binary instructions that the computer's processor will read and execute.
Binary instructions are the output from compiling human readable source code such as the following C snippet:
<div>
<caption markdown="1">**Listing 1**. Simple strdup call</caption>
{% highlight c++ %}
1. int main(int argc, char** argv)
2. {
3.     char programName[8];
4. 
5.     strcpy(programName, argv[0]);
6. 
7.     return 0;
8. }
{% endhighlight %}
</div>

At a high level, the generated binary instructions would have to do the following:
1. Reserve space for the local variable *programName* 
3. Prepare to call **strcpy** with the argument such that the function will copy the string ***argv[0]*** into *programName*
4. Call **strcpy**
6. Deallocate the space that was reserved for the local variable *programName*
7. Return from the function

### Program Stack
The program's stack can be considered scratch space used by a program to track the flow of execution, function arguments, and local variables.

Consider the code sample from the section above in **Listing 1**.


**Figure 1** shows the line number of execution and the what the program stack looks like at the end of each line.
For this example, we will assume that the program is running on a 32-bit architecture.
The implication of this is that the default size of operands the processor expects is 32-bits (or 4 bytes). 

This in turn means that the processor will expect all available memory to be addressable with 32-bits.

<table style="text-align:center">
    <caption markdown="1">**Figure 1**. Stack state during program execution</caption>
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
            <td>Address of Line 6</td>
            <td></td>
        </tr>
        <tr>
            <td>0x14</td>
            <td></td>
            <td>Address of argv[0]</td>
            <td></td>
        </tr>
        <tr>
            <td>0x18</td>
            <td></td>
            <td>programName</td>
            <td></td>
        </tr>
        <tr>
            <td>0x1c</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>'e' 'x' 'e' 0x00</td>
        </tr>
        <tr>
            <td>0x20</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>'f' 'o' 'o' '.'</td>
        </tr>
    </tbody>
</table>

Starting at Line 3, the program reserves space on the stack for a local variable, *programName*.
{% highlight c++ %}
3.     char programName[8];
{% endhighlight %}
This line creates a local character array of 8 characters. This reserves 8 bytes of memory.

Line 5 copies the string *argv[0]* to *programName*.
{% highlight c++ %}
5.     strcpy(programName, argv[0]);
{% endhighlight %}
There are several steps packed into this one line.

First, the function *strcpy* is given *programName* and *arg[0]* as arguments.
*argv[0]* contains the name of the program.
The convention for these sorts of programs is that the first argument is the name of the file that was run,
so if the code was compiled into the executable file **foo.exe**, the first argument would be *foo.exe*

The arguments to the function are pushed onto the stack from left to right and appears immediately above the space reserved for the local variable *programName*.
Once the arguments are all in place for the call to the function, the program saves the *return address*.

The *return address* is where the program will resume executing inside the current function when it returns from executing the called function.
In this case, once *strcpy* is finished, the program will continue executing at Line 6.

### Buffers
This article is about *stack* *buffer* *overflows*. 
We've already described what the program stack is in the previous section, and we've illustrated what an overflow is by way of champagne at the very beginning!
That's 2/3 of what we need.

So what is a buffer? 
<details markdown="1">
<summary>
An area of memory a program has reserved for some type of data. (click to expand)
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
In the above example, *programName* is a local stack buffer.

## Stack Buffer Overflows
Now that we know what a stack buffer is, we can start looking into how to overflow it and what this really means for the program.

Simply put, we overflow the buffer by forcing more data to be written than was reserved for the buffer.

In the example above, we invoked the program *foo.exe* which filled up the stack buffer *programName* as follows:

|Buffer Index   | Value |
| ----------    | ----  |
| 0             |   f   |
| 1             |   o   |
| 2             |   o   |
| 3             |   .   |
| 4             |   e   |
| 5             |   x   |
| 6             |   e   |
| 7             |   \0  |

Note: The final character in the array is set to the escape code for a ***NULL Byte***.
This byte has a value of 0 and is used to mark the end of a string in memory.

What would happen if our program was instead named *foobar.exe*?
Let's bring out the stack diagram:

<table style="text-align:center">
    <caption markdown="1">**Figure 2**. Stack state during program execution of foobar.exe</caption>
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
            <td>Address of Line 6</td>
            <td></td>
        </tr>
        <tr>
            <td>0x14</td>
            <td></td>
            <td>Address of argv[0]</td>
            <td></td>
        </tr>
        <tr>
            <td>0x18</td>
            <td></td>
            <td>programName</td>
            <td>'x' 'e' '\0' '\0'</td>
        </tr>
        <tr>
            <td>0x1c</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>'a' 'r' '.' 'e'</td>
        </tr>
        <tr>
            <td>0x20</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>'f' 'o' 'o' 'b'</td>
        </tr>
    </tbody>
</table>

The function *strcpy* does not perform any bounds checking internally and assumes that the buffer provided as the destination will be large enough to hold the copied string.

At best, a stack buffer overflow can do nothing and is totally harmless.

On the other end of things, a stack buffer overflow could force the program to crash.

At worst, a stack buffer overflow could be exploited by an attacker to force the program to execute different code entirely.

### Stack Buffer Overflow Exploits

In *Figure 2*, the stack buffer overflows but does not appear to have any major consequences.
However, this vulnerability is ***super*** exploitable.

What if we rename the compiled program to 'aaaaaaaaaaaaaaaaaaaa'?

<table style="text-align:center">
    <caption markdown="1">**Figure 3**. Stack state during program execution of aaaaaaaaaaaaaaaaaaaa</caption>
    <thead>
        <tr class="header">
            <th ></th>
            <th colspan="3">Line</th>
        </tr>
        <tr class="header">
            <th>Stack Address</th>
            <th>5</th>
            <th>Start of strcpy</th>
            <th>End of strcpy</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>0x10</td>
            <td>Address of Line 6</td>
            <td>Address of Line 6</td>
            <td>'a' 'a' 'a' 'a'</td>
        </tr>
        <tr>
            <td>0x14</td>
            <td>Address of argv[0]</td>
            <td>Address of argv[0]</td>
            <td>'a' 'a' 'a' 'a'</td>
        </tr>
        <tr>
            <td>0x18</td>
            <td>programName</td>
            <td>programName</td>
            <td>'a' 'a' 'a' 'a'</td>
        </tr>
        <tr>
            <td>0x1c</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>'a' 'a' 'a' 'a'</td>
        </tr>
        <tr>
            <td>0x20</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>0x00 0x00 0x00 0x00</td>
            <td>'a' 'a' 'a' 'a'</td>
        </tr>
    </tbody>
</table>

Prior to the program jumping to execute *strcpy*, the address of **Line 6** is saved to the stack so the program knows where to come back to in the function once *strcpy* is done.

However, the *strcpy* operation resulted in a buffer overflow in our example.
The overflow was large enough that it ***smashed the stack*** and overwrote the saved return address, the address of Line 6.

Now, when the program goes to return from *strcpy*, it will read the value at stack address 0x10, 'aaaa', and try to jump to that location in program memory.
The ASCII character 'a' maps to a value of '0x61', which means that the program will try to jump back to executing at address 0x61616161, which is most likely not the address of *Line 6*.

## Defending Against Stack Buffer Overflows

### Data Execution Prevention
This mitigation prevents the processor from execution instructions from the program stack.

# Hands-On
@TODO: Build example

@TODO: Gather screenshots of disassembly/decompilation

@TODO: Demonstrate Proof of Concept Exploit

@TODO: Describe mitigations and protections
