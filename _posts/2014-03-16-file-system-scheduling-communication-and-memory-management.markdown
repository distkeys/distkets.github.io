---
layout: post
title: "File System, Scheduling, Communication and Memory Management"
date: 2014-03-16 18:26
categories:
 - Operating Systems
 - Distributed Systems

tags:
- '2014'
---
{% include toc %}

The scope of this document is limited to the study of placement of following services in **Mach, V Distributed System, Denali, XEN and UNIX Systems** <br><br>
•    File System<br>
•    Scheduling<br>
•    Communication<br>
•    Memory Management<br>


### The V Distributed System

The V operating system is a **microkernel** designed to work for a cluster of computer machines connected by a high-performance network. The V system was motivated by the availability and functionality of high performing computers and networks. The main idea was that Kernel is distributed in the sense that a separate copy of Kernel executes on each particular network node and each copy interacts with each other to provide single system abstraction of process. <br><br>

The design of the V system was desired to provide:<br>

•    High-performance communication<br>
•    The protocol defines the system<br>
•    Small operating system kernel implementing basic protocols and services<br>


The V System provides services at the process level in machines and in network-independent fashion. Following are some services provided by the V system:<br>

•    **Process Scheduling** – Kernel process server (which is User space as well as Kernel space). Scheduling is **provided by Kernel** by only priority-based scheduling but the second level of scheduling is performed **outside the Kernel.**<br>
•    **Communication** – Provided in Kernel Space.<br>
•    **Memory Management** – Kernel as well as Kernel memory server. The Kernel must implement some level of memory support to protect the **integrity.** Kernel serves as **binding, caching** and provides **consistency** mechanism for regions and open files. The **Kernel memory server** supports file-like read and write access to address space using UIO interfaces.<br>
•    **File System** – File system service is provided by V file server.<br>


**Kernel Servers:** The V System provides the concept of Kernels server in which above services is implemented by a separate kernel module which is replicated across nodes. Each module is registered by IPC and **invoked from process level.**

<hr style="border-top: 1.5px dotted black"/><br><br>

### Mach

Mach system is **microkernel** operating system with design goals to provide portability, extensibility, security in Kernel, support transparent distributed operation and reduce the number of features in Kernel make it less complex.<br><br>

•    **Process Scheduling** – Kernel space to ensure fairness of system.<br>
•    **Communication** – Provided in Kernel Space using ports.<br>
•    **Memory Management** – Kernel as well as Userspace. The Kernel must manage page tables. User-level space decides the page replacement algorithm. User-level memory manager use system calls to communicate with Kernel for memory mapping, page-in, page-out, and page-level locking.<br>
•    **File System** – File system service is managed outside the Kernel. <br>

<hr style="border-top: 1.5px dotted black"/><br><br>

### Xen

Xen is a virtual machine monitor for x86 architecture, which allows multiple operating systems to share the same hardware with a safe and orderly manner without losing any performance. Xen is a hypervisor, which can be argued as a **microkernel**. Although it provides services like microkernel it is **not a microkernel**. Soon, it is expected to be Xen emerging as a Microkernel.<br><br>

Xen is implemented on the concept of para virtualization which requires changes to be made in guest OS, but it does not require any change in application binary interface which gives freedom to the guest application from the change.<br><br>

Xen provides a solution to various challenges for virtualization. Each guest OS assumes that it has the highest privilege in the system. All guest OS is hosted and managed by Xen, Xen has to be in the highest privilege in the system. Xen take advantage of the x86 architecture and modify the guest OS from Ring 0 to Ring 1 and Xen resides in Ring 0.<br><br>

•    **Scheduling** – Kernel space to ensure fairness of the system. Xen uses Borrowed Virtual Time scheduling algorithm.<br>
•    **Communication** – Service is provided in Xen as well as Guest OS space. Xen provides the abstraction of a virtual firewall router where each domain has one or more network interfaces logically connected to VFR. For transmitting packet guest OS enqueue descriptor and Xen copies the descriptor and to ensure safety copies the packet header and execute any matching filter rules.<br>
•    **Memory Management** – Guest OS is responsible for allocating and managing hardware page tables. In this case, relative to Xen Guest OS can be called in Userspace although Guest OS is in Ring 1 and applications run in Ring 3. Xen involvement is to ensure safety and isolation. When Guest OS requires new page table it allocates and initializes from its memory reservations and register it with Xen.<br>
•    **File System** – File system service is managed outside the Kernel. <br>

<hr style="border-top: 1.5px dotted black"/><br><br>

### Denali

Denali **isolation** kernel is the x86 based operating system that isolates untrusted software service in the separate protection domain. It is a small kernel operating system architecture targeted at hosting multiple applications with little data sharing. Small kernel sometimes resembles to **microkernel.**  <br><br>

Following are the Denali design Principles:
•    Expose low-level resources rather than high-level abstractions.<br>
•    Prevent direct sharing by exposing only private, virtualized namespaces.<br>
•    Scalability<br>
•    Modify the virtualized architecture for simplicity, scale, and performance.<br>

Denali system focuses on executing service in a separate VM, which provides stronger isolation.

•    **Scheduling** – Scheduling is done by Kernel in Denali.<br>
•    **Communication** – Service is provided in Denali as well as Guest OS space. Denali Ethernet has been simplified so that it requires only one PIO to send and receive the packet, which improves performance. To get this benefit Guest OS device driver has been modified. <br>
•    **Memory Management** – Guest OS is responsible for accessing a portion of physical address space allowed to Guest OS. Denali involvement is to ensure safety and isolation. The isolation kernel allocates swap region. Upon page fault isolation kernel take care by verifying the faulting VM has not accessed illegal virtual address.<br>
•    **File System** – File system service is managed outside the Kernel. <br>

<hr style="border-top: 1.5px dotted black"/><br><br>

### UNIX

Unix is the **monolithic** kernel operating system, which means all the services, has to be implemented and provided by the OS kernel. Unix philosophy has been described as follows :<br><br>
1.    **Rule of Modularity:** Write simple parts connected by clean interfaces.<br>
2.    **Rule of Clarity:** Clarity is better than cleverness.<br>
3.    **Rule of Composition:** Design programs to be connected to other programs.<br>
4.    **Rule of Separation:** Separate policy from mechanism; separate interfaces from engines.<br>
5.    **Rule of Simplicity:** Design for simplicity; add complexity only where you must.<br>
6.    **Rule of Parsimony:** Write a big program only when it is clear by demonstration that nothing else will do.<br>
7.    **Rule of Transparency:** Design for visibility to make an inspection and debugging easier.<br>
8.    **Rule of Robustness:** Robustness is the child of transparency and simplicity.<br>
9.    **Rule of Representation:** Fold knowledge into data so program logic can be stupid and robust.<br>
10.    **Rule of Least Surprise:** In interface design, always do the least surprising thing.<br>
11.    **Rule of Silence:** When a program has nothing surprising to say, it should say nothing.<br>
12.    **Rule of Repair:** When you must fail, fail noisily and as soon as possible.<br>
13.    **Rule of Economy:** Programmer time is expensive; conserve it in preference to machine time.<br>
14.    **Rule of Generation:** Avoid hand hacking; write programs to write programs when you can.<br>
15.    **Rule of Optimization:** Prototype before polishing. Get it working before you optimize it.<br>
16.    **Rule of Diversity:** Distrust all claims for “one true way.” <br>
17.    **Rule of Extensibility:** Design for the future, because it will be here sooner than you think.<br>


•    **Scheduling** – Scheduling is done by kernel in Unix.<br>
•    **Communication** – Systems calls are be called in user space but the authorization is provided by Kernel.<br>
•    **Memory Management** – Kernel Space.<br>
•    **File System** – Kernel Space.<br>

<br><br><br>

### References

<a href="http://www.faqs.org/docs/artu/ch01s06.html" target="_blank">Basics of the Unix Philosophy</a>


