# Summary for LKL: The Linux Kernel Library

# Index
- [1. Introduction](#introduction)

# Introduction
Linux Kernel: One of the largest software projects in the world, high quality, complex <br>

Some projects (that found themselves in need of functionality implemented in some of the Linux subsystems) => Reimplement it from scratch <br>

Number of ext2 implementation written without directly using code from Linux kernel: <br>

Windows drivers Ext2 FSD, Ext2 IFS, Explore2fs Windows GUI explorer, Haiku Ext2 file system driver, ext2lib library for Windows <br>

Same kind of functionality => Reimplemented in different languages for different OS as a kernel module, as a application and a library <br>

=> But with different levels of support, performance, bug => Hard to reuse when re-implementing other file systems or other subsystems of the Linux kernel <br>

Extracting portions of the Linux kernel:

1. Selecting necessary part, separating + maintaing => Consume resources and require highly-skilled developer
2. Once separated code => stagnates, Upstream kernel introdced very slowly including bug fixes or features (Reverse is also true)

**The Linux kernel library project** => Organize the Linux code in a form such that it can be easily reused by an application <br>

Effort spent on getting access to Linux kernel functionality => Reduced to compiling the Linux Kernel (including patches) and linking resulting library into applications <br>

1. Developers can concentrate on creating new applications based on functionality provided by Linux kernel code
2. Upgradeing LKL -> Automatically brings bug fixes and new features updated in the main Linux tree => Freeing developers from monitoring changes done on kernel and subsystems 

Applications can make use of full Linux kernel subsystems like
1. Virtual file system
2. Networking stack
3. Task management, scheduling
4. (Even) Memory management to some extent

LKL -> Can run in any environment satisfying a few basic primitives, Can be used in user space (Linux, Windows) and kernel space (Windows) <br>

![Arch](./images/LKL_1.png)

# Architecture
Main goal => Provide simple, maintainable way that applications can reuse Linux Kernel code <br>

Should be 
1. usable from both user, kernel space
2. allow easy tracking of Linux kernel tree
3. require minimum **glue code** from the application and provide stable, easy way to use API

In order to be able to easily upgrade LKL to future Linux kernel releases => Require a mechanism to cleanly separate the LKL specific components from the mainline kernel <br>

**User-mode Linux** => Implement LKL as a port of the kernel to a virtual computuer architecture, named **lkl** => No need to change any of the core kernel components <br>

Instead, the applications needs to provide implementations for a small set of envrionment dependent primitives => 
**native operations** from hereafter, are used by the generic lkl architecture to build the virtual machine upon which the Linux kernel will execute <br>

LKL system call interface => API based on the Linux system call interface, Offer a stable and familiar API to applications 

## The LKL Architecture
LKL port => Interact with applications via an interface which includes the LKL native operations => **LKL system call**, **Interrupr-like API** (notify the Linux kernel about external events) <br>

Not all primitives (that need to be ofered by an architecture layer for a full Linux port) were required <br>

For example... <br>

Only using Linux kernel for a single application => Do not need **address space separation** and protection between user and kernel or multiple user address spaces <br>

Do not even need some of the user space abstractions that the kernel offers (like user processes, process address spaces or signals) <br>

=> Allow simplifying the interface between the application and LKL. <br>

Most visible effect: Can directly link LKL into the application

### Memory Managment Support
