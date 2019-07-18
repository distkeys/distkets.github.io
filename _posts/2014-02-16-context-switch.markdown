---
layout: post
title: "Context Switch"
date: 2014-02-16 14:53

categories:
 - Operating Systems
 - Distributed Systems

tags:
- '2014'
---


The scope of this post is limited to the study of context switches in various communication and synchronization primitives in distributed systems mainly focus on the following:

•    Message Passing<br>
•    Remote Procedure Calls (RPC) <br>
•    Lightweight Procedure Calls (LRPC)<br>
•    Distributed Shared Memory (DSM)<br>

{% include toc %}
<br>

### Message Passing

Message passing is the communication primitive based on the Client-Server architecture involving communication only through a message passing. In message passing architecture, the user process generates the datagram and interrupts the kernel for sending a datagram to the receiver. This process needs a context switch. <br><br>

At receiving end, after receiving a datagram by the service, the data is sent to the application in User Space from Kernel module, which involves the context switch. The same process is done back and forth to send and receive data with respective context switching in Message passing scheme of communication.<br><br>

Synchronization is achieved by blocking send and receives which may result in blocking the process in execution, while it receives a response from the receiver, and volunteering giving up the CPU which might be argued as a context switch but in this post this mechanism is not treated as a context switch.
<br><br>

### Remote Procedure Calls (RPC)

RPC communication models aim to achieve the results of a regular procedure call but in a distributed environment. RPC is a level of abstraction on top of message passing, and since, the nature of procedure call is continuous RPC follows a synchronous communication methodology. Four components in RPC are:<br>
•    User – In user space<br>
•    User Stub and RPC Communication Package – In Kernel Space<br>
•    Server Stub and RPC Communication Package – In Kernel Space<br>
•    Server – In user space on remote machine<br>

User makes an RPC call like a regular procedure call which result in calling User stub and later the RPC communication package which is called the RPC runtime often. This process involves the context switching, and the kernel takes care of RPC request using user stub and RPC runtime for marshaling data and sending to the receiver.

At the receiving end, the receiving and unmarshal of a data packet is done by RPC runtime and server stub. The data is sent to the required user process for execution. This is the point of context switch at the receiving side. The same procedure happens while sending the result back to the sender for the RPC request. The whole process involves the four context switches. Four context switches are because the interrupt handler receives the incoming packets and delivers to the correct process. It can be reduced to even two context switches if an incoming message can be delivered directly to the correct process without the intervention of interrupt handler.

Binding is the process of knowing the machine names and location which sender machine can connect for RPC operation. It can lead to an increase in context switch as it is itself implemented as a separate module. In RPC paper, Grapevine is used as a database for binding.  Sending and receiving communication can lead to two additional context switches on the sender side. Same is for the receiver side.
<br><br>

### Lightweight Remote Procedure Call

This communication model focuses on communication between protection domains on the same machine. In a high-level view, LRPC client makes a procedure call to server procedure by kernel trap, which leads to a context switch. Kernel processes the request and when the called procedure completes result is returned to the client from the kernel which again leads to the context switch.<br>

Moving to fine granularity of LRPC, during the binding process, the client makes import interface request via kernel. The kernel sends the request to server’s waiting clerk, and in response, waiting clerk sends a response to the kernel with information. The kernel then return the binding Object to the client back, and the whole process requires four context switches.<br>

LRPC minimize the use of shared data structures which internally implements its locks, so no explicit lock is required for synchronization. LRPC implements the optimization by reducing the number of context switches by caching domains on idle processors. The kernel looks for the idle processor for the client request, and if found one the request is routed to the processor without any context switch. Same is done when returning the result back to the client. The kernel looks for any idle client process and uses it without any context switch. If no idle domain can be found, then a single processor context switching is done.
<br><br>

### Distributed Shared Memory – IVY

Distributed shared memory is the mode of communication in which single address space is shared by all the processor. The processor can access any memory location at any time. In this process, page size plays an important factor in performance. If the page size is big, the two processes are accessing different section of the same page; it reduces the performance by generating a page fault. <br>

False sharing increases the context switch. Context switching and synchronization varies in various methodologies to handle Memory Coherence problem. <br>

Page invalidation approach for Page Synchronization invalidates all the copies of the page. Next time when another process requires this page, it generates the page fault hence increase the number of context switches. <br>

In write broadcast approach, fault handler updates each copy. Next time process doesn’t generate the page fault because it has an updated copy hence reduces the context switch. In various page ownership approaches like *fixed, dynamic with a combination of invalidating and write broadcast* context switches are required just from passing control from user space to kernel space and vice versa. Then the execution for *centralized and distributed* approaches can be carried away in kernel mode with different strategies for message passing.

<br><br>

### Further reading

1. Message Passing
2. Distributed Shared Memory in <a href="https://www.dropbox.com/s/ukj7np5c78161at/shared%20virtual%20memory%20system.pdf?dl=0" target="_blank">Ivy</a> (Integrated shared Virtual memory at
Yale)
3. <a href="https://www.dropbox.com/s/1ktdgouptq41fve/2.ImplementingRPC.pdf?dl=0" target="_blank">Birrel and Nelson Remote Procedure Call</a>
4. <a href="https://www.dropbox.com/s/7i4kvjg741r5idz/LRPC.pdf?dl=0" target="_blank">Lightweight Remote Procedure Call</a>
