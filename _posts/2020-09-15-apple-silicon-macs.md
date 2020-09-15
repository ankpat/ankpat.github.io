---
layout: post
title:  "System Architecture of Apple Silicon Macs"
categories: macOS Apple ARM Apple-Silicon PAC Pointer-Authentication SecureBoot iBoot
---

In June of 2020, Apple announced that the Mac line will [transition from using Intel
based processors to Apple Silicon](https://developer.apple.com/news/?id=ywv74y7m). After years of speculation, Apple finally
confirmed that they are indeed planning to release a Mac system running on an ARM
processor.

Apple uses similar ARM processors for their mobile device offerings, including
the iPhone and iPad. In fact, during WWDC2020, the demonstration Mac was running
on an A12Z processor, the same chip that powers the 2020 line of iPad Pros!

What I'm looking forward to most are the performance and security benefits that
might come from the switch to Apple Silicon.

## Security Features
Apple mentions 4 major security features that will be immediately available for
Macs running on Apple Silicon.

### Write XOR Execute
This feature has been present on iOS for a long time and prevents memory from
being marked as Read-Write-Execute (RWX). Finding or creating a RWX block of
memory from the perspective of an attacker or malware writer provides a space to
write any code they desire and be able to execute it.

With Write XOR Execute (W^X), Memory can be Read/Execute or Read/Writeable, but
never Writeable AND Executable (hence the W^X).

Typically, when a program is written, the code is immutable and does not need
to change during execution, so there is no reason to mark code as writeable.

However, there are classes of compilers call Just-in-Time (JIT) compilers that can take
interpreted code, such as Python or Java Bytecode, and compile them into native code
in memory. In this situation, the JIT generated code would need to be written to
memory and the same memory would need to be executable.

Apple has introduced a new function call ```pthread_jit_write_protect_np``` which
can be used to toggle permissions of memory regions between RX and RW. What is
noteworthy is that the protection applies *per thread*. This means that two threads
in the same application can have different views of the same memory block.

#### Basic JIT code generation
Apple has written a [short guide](https://developer.apple.com/documentation/apple_silicon/porting_just-in-time_compilers_to_apple_silicon) on how to develop applications that use JIT
compilation.
1. Call mmap with MAP_JIT to allocate a memory region that can contain compiled code
2. Call ```pthread_jit_write_protect_np``` to disable JIT write protection for the memory region
and make the region RW
3. Write compiled instructions to the memory region.
4. Call ```pthread_jit_write_protect_np``` to restore executable permissions to make the memory
region RX
5. Call ```sys_icache_invalidate```! ***Instruction caches are not coherent with
data caches on Apple silicon***. Weird and potentially bad things can happen if the
processor tries to execute the generated instructions without first invalidating the cache.
6. Jump to the first instruction in the memory region.


### Kernel Integrity Protection
Kernel integrity protection has been present in iOS in various incarnations. These
have all been features put in place to prevent modification of kernel code, such as Kernel Patch Protection (KPP)
and Kernel Text Readonly Region (KTRR). Apple has indicated that a Kernel Integrity Protection (KIP) will be
present on Macs running on Apple Silicon.

KIP also prevents new pages allocated in the kernel from being made executable.

### Pointer Authentication
Pointer Authentication is a security feature from ARM v8.3 that was included in
Apple's A12 line of chips. A pointer is a reference kept by a program to a specific
memory address. If a pointer is modified somehow, either by accident or maliciously,
this could cause the system to perform an operation on memory it was not supposed to.

With pointer authentication, the upper bits of each pointer are used to encode a
Pointer Authentication Code (PAC) which cryptographically "signs" the pointer.
If a pointer is overwritten, then when the processor attempts to validate the PAC,
the verification will fail and the system will halt the operation of the thread
because of a PAC failure.

By default, PAC on Macs running Apple Silicon will be enabled in the kernel, for system applications,
and system services. PAC will not be enabled for user space applications.

However, for those that are curious, there is a way to turn on support for userspace PAC
by adding the "-arm64e_preview_abi" boot argument.

### Device Isolation
Devices that get plugged into a Mac need to be able to communicate with the system.
Some devices, such as high-speed Thunderbolt devices, need to talk to the system
at the lowest level to realize super fast speeds. In order to do this, Thunderbolt
is directly attached to the PCIe bus, which has full direct memory access (DMA) to the system.

A malicious device with DMA can modify sensitive data and compromise a system.
One mitigation for this is to introduce an input-output memory management unit (IOMMU).
An IOMMU provides virtual addressing for devices to prevent DMA vulnerabilities.

With Device Isolation available in Apple Silicon, each devices gets its own IOMMU.
On Intel systems, devices share memory for all devices. A per-device IOMMU further
restricts what a malicious device can do.

## Kernel Extensions

For the longest time, the only way to add additional functionality to the system
to support new devices was to extend the kernel with Kernel Extensions.
As of macOS 10.15, [kernel extensions were considered deprecated](https://developer.apple.com/support/kernel-extensions/) in favor of System Extensions which reside in User space.

macOS on Apple Silicon will still support Kernel Extensions, but it will be a painful experience.

Kernel extensions must have PAC enabled and the overall security process of the OS installation
must be reduced to support loading and running kernel extensions.

## Want to Learn More?
With macOS and iOS now sharing a similar architecture, I have an opportunity to do something
I've always wanted to do, and probably something a lot of iOS researchers have wanted.

There will now be a way to poke at iOS system internals from macOS to better help make
the devices we all use safer for everyone.

More importantly, it provides an opportunity for me to use Macs to develop and organize
a class on iOS and macOS internals with hands on labs designed for security researchers!

Interested in the class and want to know about future offerings? Reach out to me or [Nissint Technologies](https://www.nissint.com/).
