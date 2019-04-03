---
layout: post
title:  "Intel Assembly Primer"
---
This primer is intended to provide a reader with almost zero experience with assembly enough background to understand the following concepts:
- CPU Registers
- Function Calling Conventions
- Common Intel Assembly Instructions
- Shellcode
- How to Write Your Own Shellcode

An important caveat is that this article will not dig deep into processor internals.
There is going to be lots of hand-waving and statements that are "true enough" to get the point across.
The goal is to provide enough background knowledge to get through the Stack Overflow Series and development your own tailored exploit for a real-world stack buffer overflow vulnerability.

It will definitely help if you've had some experience with C at some point.
It doesn't have to be much, in fact, just being able to write and run a "Hello, World" program would be enough.

* TOC
{:toc}

## Computer Processors
Let's start with computers. They do things.
How do they know what to do?

At the core of the computer is the processor.
The processor fetches instructions from wherever they might be stored, decodes the raw binary instruction, and executes some action based on the decoded instruction.

A common way to describe processors is by the size of operands they can handle. There are processors that can handle operands of sizes anywhere from 8-bits to 64-bits at a time.
A processor that can process 64-bits at once is a 64-bit processor.
For that architecture, it can be said the **word size** of the system is 64-bits.

### Registers

When the processor needs to keep track of things, such as where to fetch the next instruction from in memory or keep track of operands for mathematical operations, it uses local, volatile memory.
This memory is finite and divided into discrete components called registers.
(This isn't entirely true, but it's true enough).

On Intel systems, and others, there are two common registers, the Stack Pointer and the Instruction Pointer (sometimes referred to as the Program Counter).

The instruction pointer keeps track of where in memory the current instruction resides.
It increases as instructions are executed.

To make this a bit more complicated, the Intel architecture uses variable length instructions.
Some instructions are just one byte long, whereas others can be several bytes.
This only becomes a problem if you're trying to guess what the value of the instruction pointer will be after executing some number of instructions; it matters what those instructions are.
So... don't do that.

The stack pointer keeps track of the next location in memory the program can use for its scratch space, referred to as the stack.
If you haven't done so already, take a peek at [Stack Buffer Overflows Explained]({% post_url 2019-02-20-stack-overflow %}) for some a high level explanation of the program stack.
Later on in this article, we're going to dig deep into the 64-bit Stack.

In addition to these special purpose registers, there are also general purpose registers that can be used for any purpose.
Think of them as scratch space to hold values such as intermediate results for operations and arguments to a function.

The 64-bit Intel architecture provides the following registers: RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP, RFLAGS, RIP, R8, R9, R10, R11, R12, R13, R14, R15.

There's some baked in history that comes with these register names which I do encourage you to look into (Wikipedia has a pretty good writeup on [x86](https://en.wikipedia.org/wiki/X86)).

The important part is there are quite a few general purpose registers available for use.
*RFLAGS*, *RIP*, *RBP*, and *RSP* serve pretty specialized purposes:

|Register| Purpose|
| ------ | ------ |
| RFLAGS | Contains the processor state |
| RIP | Instruction Pointer |
| RBP | Base Pointer - Points to the start of the stack frame |
| RSP | Stack Pointer - Points to the top of the program stack |

## Processor in Action
Let's do this by examples!

### A Simple Function

{% highlight c++ %}
 1. int sum(int a, int b)
 2. {
 3.     int c = 0;
 4.     c = a + b;
 5.     return c;
 6. }
 7. int main()
 8. {
 9.     int foo = 0;
10.     
11.     foo = sum(1, 100);
12. 
13.     return foo;
14. }
{% endhighlight %}

The code above is a very simple program.
First, it defines a function ***sum*** which takes two arguments, **a** and **b**.
The function declares a local variable **c**, initialized to 0, which is used to hold the result of adding **a** and **b**.
The function then returns the result.

The next function is ***main***, which is the program's entry point.
Inside this function, a local variable **foo** is declared and initialized to 0.
**foo** is then assigned the result of calling ***sum*** on the values of 1 and 100.

The function then returns **foo**, which in turns terminate the program.


