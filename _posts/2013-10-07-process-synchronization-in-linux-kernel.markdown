---
layout: post
title: "Process synchronization in Linux Kernel"
date: 2013-10-07 20:56

categories:
 - Operating Systems
 - Linux
tags:
 - '2013'
---
{% include toc %}

As we have discussed earlier process synchronization, let's discuss process synchronization primitives offered in Linux Kernel.

This blog is a summary of this [article](https://1drv.ms/b/s!AqRlTEkVrXvJg6RRka15OxeKT_uQ3A).


### Synchronization Primitives

1. Per-CPU variables
2. Atomic Operation
3. Memory barrier
4. Spin Lock
5. Semaphore
6. SeqLocks
7. Local Interrupt disabling
8. Local softirq disabling
9. Read-Copy-Update

&nbsp;

### Summary of Synchronization Primitives


| Technique | Description | Scope |
|:--------|:-------:|--------:|
| Per-CPU variables | Duplicate a data structure among the CPUs | All CPUs |
| Atomic operation | Atomic read-modify-write instruction to a counter |  All CPUs  |
| Memory barrier |  Avoid instruction reordering |   Local CPU  or  All CPUs  |
| Spin lock |  Lock with busy wait |   All CPUs  |
| Semaphore |  Lock with blocking wait (sleep) |   All CPUs  |
| Seqlocks |  Lock based on an access counter |   All CPUs  |
| Local interrupt disabling |  Forbid interrupt handling on a single CPU |   Local CPU  |
| Local softirq disabling |  Forbid deferrable function handling on a single CPU |   Local CPU  |
| Read-copy-update (RCU) |  Lock-free access to shared data structures through pointers |   All CPUs  |


&nbsp;
### Per-CPU variables

The best synchronization technique consists in designing the kernel to avoid
the need for synchronization in the first place.


> Basically, a per-CPU variable is an array of data structures, one element per each CPU in the system.


A CPU should not access the elements of the array corresponding to the other CPUs.

**Pros**

- Freely read and modify its element without fear of race conditions.
- It avoids cache line snooping and invalidations, which are costly operations.

&nbsp;
<mark>Snooping and Invalidations</mark> - Snooping is the process where the individual caches monitor address lines,  for accesses to memory locations that they have cached. When a write operation is observed to a location that a cache has a copy of, the cache controller invalidates its copy of the snooped memory location.

&nbsp;
**Cons**

- It can only be used when it makes sense to split the data across the CPUs of the system logically.
- Do not protect against access from asynchronous functions such as interrupt handlers and deferrable functions.
- Per-CPU variables are prone to race conditions caused by kernel preemption, both in uniprocessor and multiprocessor systems.

<mark>Problem</mark> What would happen if a kernel control path gets the address of its local copy of a per-CPU variable, and then it is preempted and moved to another CPU: the address still refers to the element of the previous CPU.

As a general rule, a kernel control path should access a per-CPU variable with kernel preemption disabled.

<hr style="border-top: 1.5px dotted black"/><br>

### Atomic Operations

<mark>Problem</mark> Several assembly language instructions are of type “read-modify-write” i.e. memory location is accessed twice first to read the old value and second time to write a new value

**Detailed explanation**<br>
Suppose that two kernel control paths running on two CPUs try to “read-modify-write” the same memory location at the same time by executing nonatomic operations.

&nbsp;
At first, both CPUs try to read the same location, but the memory arbiter (a hardware circuit that serializes accesses to the RAM chips) steps in to grant access to one of them and delay the other. However, when the first read operation has completed, the delayed CPU reads the same (old) value from the memory location.

&nbsp;
Both CPUs then try to write the same (new) value to the memory location; again, the bus memory access is serialized by the memory arbiter, and eventually both write operations succeed. However, the global result is incorrect because both CPUs write the same (new) value. Thus, the two interleavings “read-modify-write” operations act as a single one.
&nbsp;

<mark>Solution</mark> Reason for the problem here is that Both CPU’s try to access memory location at the same time. If the operation of “read-modify-write” can be made atomic, then problem would be solved i.e. Every such operation must be executed in a single instruction without being interrupted in the middle and avoiding accesses to the same memory location by other CPUs.

* Read-modify-write assembly language instructions are atomic if no other processor has taken the memory bus after the read and before the write. Memory bus stealing never happens in a uniprocessor system.
* For multiprocessor system, Read-modify-write assembly language instructions whose opcode is prefixed by the **Lock Byte** (0xf0) are atomic.
* Other processors cannot access the memory location while the locked instruction is being executed.

Sample functions: atomic-read(v), atomic-set(v,i), atomic-add(i,v), atomic-sub(i,v) etc…

<hr style="border-top: 1.5px dotted black"/><br>

### Optimization Barriers & Memory Barriers

Compilers optimize the code for efficient usage of CPU time and memory, but these optimizations could be disastrous sometimes when dealing with shared data.

It can never be guaranteed that instructions will be performed in the same order in which they appear in source code mainly because reordering by the compiler to optimize and CPU executing several instructions in parallel. This might lead to a reordering of memory access patterns.

To avoid this behavior, we need a synchronization primitive at two levels

* Synchronization primitive at Compiler level - Optimization barrier
* Synchronization primitive at CPU level      - Memory barrier

In linux *barrier()* macro acts as an *optimization barrier*. <br>


> Optimization barrier primitive ensures that assembly language instructions of C statements mentioned before barrier() and assembly language instructions of C statements mentioned after, remains in the same sequence.


However, Optimization barrier cannot control in which fashion instructions will be executed by CPU.

A *memory barrier* primitive ensures that the operations placed before the primitive are finished before starting the operations placed after the primitive. Read *memory barrier* act only on instructions that read from memory, while Write *memory barriers* act only on instructions that write to memory.

<hr style="border-top: 1.5px dotted black"/><br>

### Semaphores

Please visit <a href="http://distkeys.com/blog/2013/10/07/process-synchronization-in-os/#semaphores">Semaphore Page</a>

### Spin Locks

When a kernel control path must access a shared data structure or enter a critical section, it needs to acquire a lock.<br>

&nbsp;
Spin Locks are a special kind of Lock designed to work in a multiprocessor environment.

> If kernel control path finds Spin Lock Open it acquires the Lock and continue execution else it Spin around repeatedly executing tight instruction loop until the Lock is released.


The waiting kernel control path keeps running on the CPU, even if it has nothing to do besides waste time. Kernel preemption is disabled in CR protected by Spin Locks.

&nbsp;
Spin locks are usually convenient because many kernel resources are locked for a fraction of a millisecond only; therefore, it would be far more time-consuming to release the CPU and reacquire it later.<br>

&nbsp;
Kernel preemption is still enabled during the busy-wait phase, thus a process waiting for a spin lock to be released could be replaced by a higher priority process.


<table>
<tr>
<td>Macro</td><td>Description</td>
</tr>
<tr>
<td>spin-lock-init()</td><td>Set the spin lock to 1 (unlocked)</td>
</tr>
<tr>
<td>spin-lock()</td><td>Cycle until spin lock becomes 1 (unlocked), then set it to 0 (locked)</td>
</tr>
<tr>
<td>spin-unlock()</td><td>Set the spin lock to 1 (unlocked)</td>
</tr>
<tr>
<td>spin-unlock-wait()</td><td>Wait until the spin lock becomes 1 (unlocked)</td>
</tr>
<tr>
<td>spin-is-locked()</td><td>Return 0 if the spin lock is set to 1 (unlocked); 1 otherwise</td>
</tr>
<tr>
<td>spin-trylock()</td><td>Set the spin lock to 0 (locked), and return 1 if the previous value of the lock was 1; 0 otherwise</td>
</tr>
</table><br>


Spin Lock with kernel preemption

~~~
//Invokes preempt-disable() to disable kernel preemption.

; Intel syntax

locked:                      ; The lock variable. 1 = locked, 0 = unlocked.
dd      0

spin-lock:
mov     eax(new), 1     ; Set the EAX register to 1.

xchg    eax(new), [locked]
; Atomically swap the EAX register with
;  the lock variable.
; This will always store 1 to the Lock, leaving
;  previous value in the EAX register.

test    eax(new), eax(prev)
; Test EAX with itself. Among other things, this will
;  set the processor's Zero Flag if EAX is 0.
; If EAX is 0, then the Lock was unlocked and
;  we just locked it.
; Otherwise, EAX is 1 and we didn't acquire the Lock.

//   Enable kernel preemption preempt-enable()

jnz     spin-lock       ; Jump back to the MOV instruction if the Zero Flag is
;  not set; the Lock was previously locked, and so
; we need to spin until it becomes unlocked.

ret                     ; The Lock has been acquired, return to the calling
;  function.

spin-unlock:
mov     eax, 0          ; Set the EAX register to 0.

xchg    eax, [locked]   ; Atomically swap the EAX register with
;  the lock variable.

ret                     ; The Lock has been released.
~~~


In the case of kernel preemption, the first step is to disable kernel preemption, preempt-disable()
In the above example, Lock is 1 and Unlock is 0

* In EAX register we push “1” as we wish to acquire Lock.<br>
* xchg command will exchange value from EAX register to lock variable but not vice versa. So by step 2, EAX register has a value of “1” and lock variable also has a value of “1”. This is an atomic operation.<br>
* Step 3, test or validate the current value of lock. If the current value of Lock (EAX prev) is “1“ i.e. processor zero flag is already set; hence Lock is not available. <br>
* Enable kernel preemption and Jump(Spin) back.<br>
* In Step 3, if the current value of Lock (EAX prev) is “0” then set it with “1,” i.e. Lock has been acquired.

It might happen that the process can spin for a long time. If the break-lock field is set, then the owning process can learn whether there are other processes waiting for the Lock. It may decide to release it prematurely to allow other processes.

#### Read/Write Spin Locks

Read Spin Locks is used to increase the concurrency within the kernel.


> Idea is that allow several kernel control path to access the same data structure for “Read” using Read Spin Locks. No kernel control path can modify the data structure using Read Spin Locks.


To modify the shared data structure kernel control path needs to acquire Write Spin Locks. Write Spin Lock is granted when no kernel control path have Read Spin Locks.

Reader and Writer Spin Locks have the same priority in this case.

<hr style="border-top: 1.5px dotted black"/><br>

### SeqLocks

In Read/Write spin locks reader and write have same priority. The reader must wait until the writer has finished and vice versa.<br>

> SeqLocks are similar to read/write locks except that they give a much higher priority to writers: in fact a writer is allowed to proceed even when readers are active.


The writer never waits, but the reader may be forced to read the data several times until it gets a valid copy. Each reader must read sequence counter twice i.e., before and after reading the data and then check whether sequence counter values are the same. If it's not equal then it means that writer must has become active and has increased the sequence counter, thus implicitly telling the reader that the data just read is not valid.

Every time writer acquires and release sequence lock, it must increment a sequence counter. When a counter is odd means writer is in progress and when the counter is even means writer is done.<br>
When a reader enters a critical region, it does not need to disable kernel preemption; on the other hand, the writer automatically disables kernel preemption when entering the critical region, because it acquires the spinlock.

SeqLocks must not be used for

* The data structure to be protected does not include pointers that are modified by the writers and dereferenced by the readers (otherwise, a writer could change the pointer under the nose of the readers)
* The code in the critical regions of the readers does not have side effects (otherwise, multiple reads would have different effects from a single read)

<hr style="border-top: 1.5px dotted black"/><br>

### Read Copy Update (RCU)

This technique is designed to protect data structures that are mostly accessed for reading by several CPUs.<br>
The key idea consists of limiting the scope of RCU as follows:

1. Only data structures that are dynamically allocated and referenced by means of pointers can be protected by RCU.
2. No kernel control path can sleep inside a critical region protected by RCU.

**Writer**

* When a writer wants to update the data structure, it dereferences the pointer and makes a copy of the whole data structure.<br>
* Next, the writer modifies the copy.<br>
* Once finished, the writer changes the pointer to the data structure so as to make it a point to the updated copy.<br>

Because changing the value of the pointer is an atomic operation, each reader or writer sees either the old copy or the new one: no corruption in the data structure may occur.

A memory barrier is required to ensure that the updated pointer is seen by the other CPUs only after the data structure has been modified.

Spinlock is coupled with RCU to forbid the concurrent execution of writers.

<mark>Problem</mark> Old copy of the data structure cannot be freed right away when the writer updates the pointer. In fact, the readers that were accessing the data structure when the writer started its update could still be reading the old copy. The old copy can be freed only after all (potential) readers on the CPUs have executed.
