---
layout: post
title: "Boot FreeBSD"
date: 2014-07-08 22:30
categories:
 - Operating Systems

tags:
- '2014'
---
{% include toc %}

This post is about the sequence and the amount of work happens under the hood when computer is switched on and the login prompt appears on the screen.

### Bird Eye View

1. Boot Command starts by initializing<br>
 - CPU<br>
 - Virtual address translations is turned off<br>
 - Disable hardware interrupts<br>

1. Boot command loads FreeBSD kernel

1. FreeBSD kernel now goes through different stages of hardware and software initialization<br>
 - Set up initial stages of CPU<br>
 - Setup run time stack<br>
 - Setup virtual memory<br>
 - Machine dependent initialization<br>
   • Setting up mutexes<br>
   • Setting virtual memory page tables<br>
   • Configure I/O devices<br>
 - Machine independent initialization<br>
   • Mounting root file system<br>
   • Initializing myriad system data structures.<br>


1. System processes are created and executed<br>
1. User level programs are brought in to execute<br>
1. System is now ready to run normal applications<br>


<br><br>

### Detailed View

1. When computer is powered on the first thing gets invoked is BIOS. <br>
1. Startup procedure stored in BIOS, reads a general purpose standalone program in boot disk. <br>
1. This general purpose standalone program executes *FreeBSD boot program*.<br>
1. *FreeBSD boot program* loads default device and program name to well known memory location.<br>
1. *FreeBSD boot program* blocks all interrupts.<br>
1. *FreeBSD boot program* disables hardware address-translation facility so that all memory references are to physical memory locations.<br>
1. Now FreeBSD kernel is started and it will starts the preparations.<br>
1. FreeBSD kernel initilization process is divided into 3 stages<br>
  - [Assembly language code](http://distkeys.com/blog/2014/07/08/boot-freebsd/#assembly-language-startup) to set up the hardware<br>
  - Load and [initialize kernel modules](http://distkeys.com/blog/2014/07/08/boot-freebsd/#kernel-module-loadinitialization)<br>
  - Start system resident processes and execute [user level startup](http://distkeys.com/blog/2014/07/08/boot-freebsd/#user-level-initialization) scripts<br>

  <br><br>

### Assembly language startup

1. Machine dependent.<br>
1. Setting up the run-time stack.<br>
1. Identifying the type of CPU on which the system is executing.<br>
1. Calculating the amount of physical memory on the machine.<br>
1. Enabling the virtual-address-translation hardware.<br>
1. Initializing the memory-management hardware.<br>
1. Setting up tables for SMP operation, if necessary.<br>
1. Crafting the hardware context for process 0.<br>
1. Invoking the initial C-based entry point of the system.<br>
1. The kernel is usually loaded into contiguous physical memory, so the translation is simply a constant offset that can be saved in an index register. <br>
1. Identify CPU. For older CPU the kernel must emulate the missing hardware instructions in software.<br>

<br><br>

### Kernel Module Load/Initialization

After the assembly-language code has completed its work, it calls the first
kernel routine written in C: the mi_startup() routine. The mi_startup() routine
first sorts the list of modules that need to be started and then calls the function routine
of each.

FreeBSD set up following services<br>

1.  Static mutex and then dynamic mutexes<br>
1.  Lock manager <br>
1.  Virtual memory system - All memory allocations by the kernel or processes are for virtual addresses that are translated by the memory management hardware into physical addresses. Memory subsystem initialization is to set limits on the resources used by the kernel virtual memory system.<br>
1.  Event handler module to register functions to be called by the kernel when an event occurs. <br>
1.  Kernel module loader - uses event handler to load dynamic kernel modules into the system at boot or run time.<br>
1.  Kernel Thread Initialization<br>
    3 processes are created<br>
    - Swapper - First process with PID 0.<br>
    - Init - Second process with PID 1.<br>
    - Idle - Created after Init. Halts the CPU when no work for it to do.<br>
1.  Device module initialization<br>
    - Network interfaces - Setup mbufs i.e. small mbufs and mbufs clusters.<br>
    - Interrupt handler - So far hardware interrupts are disabled now kernel set <br>up interrupt threads to handle interrupts when the system begins to run.
    - Device file system.<br>
1.  Initialize VFS (Virtual file system)<br>
    - Initialize vnode subsystem.<br>
    - Initialize name cache in VFS.<br>
    - Initialize Pathname translation sub system - maps pathname to inodes.<br>
    - Initialize names pipes as part of the VFS.<br>
    - Initialize clocks.<br>
1.  Start rest of kernel threads<br>
    - pagezero<br>
    - pagedaemon, bufdaemon, vnlru etc.<br>
1.  Run scheduler to start scheduling kernel threads and user-level processes.<br>

<br><br>

### User Level Initialization

After kernel initialization, at user level **init** program is executed. This is not the Init process it is **init** program located in /sbin/init.

1. Init calls fsck to check disk consistency if required and other user level scripts/processes are loaded like /etc/rc.conf, cron, getty.<br>
1. Getty finally reads a login name and invokes the /usr/bin/login program to complete a login sequence.<br><br>
