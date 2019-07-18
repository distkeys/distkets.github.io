---
layout: post
title: "Fork - FreeBSD"
date: 2014-12-31 11:51
comments: true
categories:
 - Operating Systems

tags:
- '2014'
---

In computing, particularly in the context of the Unix operating system and its workalikes, fork is an operation whereby a process creates a copy of itself. It is usually a system call, implemented in the kernel. Fork is the primary (and historically, only) method of process creation on Unix-like operating systems.


> fork() creates a new process by duplicating the calling process. The new process, referred to as the child, is an exact duplicate of the calling process.

[FreeBSD source code][2]

{% highlight c linenos %}
Function fork(td, uap)
{
    ...
     error = fork1(td, RFFDG | RFPROC, 0, &p2);

     Return pid of child
     td->td_retval[0] = p2->p_pid;
}
{% endhighlight %}

<br>

{% highlight c linenos %}
Function fork1(td, flags, pages, procp)
{
     struct proc *p1, *p2, *pptr; //p1 = parent process, p2 = child process

     Parent process
     p1 = td->td_proc;

     ...
     ...
     Allocate a new process
     newproc = uma_zalloc(proc_zone, M_WAITOK);

     To run a process we need to have atleast one thread,
     so allocate a new thread
     td2 = thread_alloc();

     Linkup thread to process
     proc_linkup(newproc, td2);
     ...
     ...

     Now we have to gro through the process address space and copy each section for new process
     Sections of process address space are
     1. Text area (read only)
     2. Initialized area
     3. Uninitiaized area
     4. Stack/Heap area

     vm2 = vmspace_fork(p1->p_vmspace);

          Function vmspace_fork(p1->p_vmspace) {
                    Process virtual address space is as follows
                    /*
                    * struct vmspace {
                    *      struct vm_map vm_map;     /* VM address map */
                    *      struct shmmap_state *vm_shm;     /* SYS5 shared memory private data XXX */
                    *      segsz_t vm_swrss;     /* resident set size before last swap */
                    *      segsz_t vm_tsize;     /* text size (pages) XXX */
                    *      segsz_t vm_dsize;     /* data size (pages) XXX */
                    *      segsz_t vm_ssize;     /* stack size (pages) */
                    *      caddr_t vm_taddr;     /* (c) user virtual address of text */
                    *      caddr_t vm_daddr;     /* (c) user virtual address of data */
                    *      caddr_t vm_maxsaddr;     /* user VA at max stack growth */
                    *      int     vm_refcnt;     /* number of references */
                    *      /*
                    *       * Keep the PMAP last, so that CPU-specific variations of that
                    *       * structure on a single architecture don't result in offset
                    *       * variations of the machine-independent fields in the vmspace.
                    *       */
                    *      struct pmap vm_pmap;     /* private physical map */
                    * };
                    */

               We allocate the process address space
               struct vmspace *vm2;
               ...
               vm2 = vmspace_alloc(old_map->min_offset, old_map->max_offset);
               ...
               ...

               Now we have to go through each vm map entries one at a time and allocate each for new process
               while (old_entry != &old_map->header) {
                    case VM_INHERIT_NONE:
                         ...

                    case VM_INHERIT_SHARE:
                         ...

                    case VM_INHERIT_COPY:
                         This is our case where we have to clone the entries and link into the map
                         new_entry = vm_map_entry_create(new_map);
                         ..
                         vm_map_entry_link(new_map, new_map->header.prev, new_entry);
                         vmspace_map_entry_forked(vm1, vm2, new_entry);
                         vm_map_copy_entry(old_map, new_map, old_entry,
               }
          } // end vmspace_fork()


     We will be now locking the process tree as we need to give new process process id
     Process of allocating a new Pid is system will scan the existing pids and try to come
     up with a range of free pids. Once a range of free pids found system will use that range
     for next lets say "k" forks. Once the range is exhausted then system on next fork will
     again find the range and so on.

     sx_slock(&proctree_lock);

     Determine if we are exceeding process createlimit
     Check privledge to increase the limit

     //Try to allocate the pids
     trypid = lastpid + 1;

     ...

     Check if we have hit the end of the range
     if (trypid >= pidchecked) {
          We have exhausted the range so find a new range
          Scan the active and Zombie procs to check whether pid is in use
          ...
          ...
     }
     sx_sunlock(&proctree_lock);

     If we are within range then
     lastpid = trypid;

     p2 = newproc;
     p2->p_state = PRS_NEW;          /* protect against others */
     p2->p_pid = trypid;


     sched_fork(td, td2);
               Function sched_fork(struct thread *td, struct thread *child) {
                    sched_fork_thread(td, child);
                              Function sched_fork_thread(td, child) {
                                   Initialize schedule related values from parent to child
                                   Assign CPU estimate info and priority
                                   ...
                              }
               } // end sched_fork()

     Insert new process to process list
     LIST_INSERT_HEAD(&allproc, p2, p_list);

     Insert into PID HASH
     LIST_INSERT_HEAD(PIDHASH(p2->p_pid), p2, p_hash);


     Lock P1
     Lock P2
     Relase Lock for All proc since we took all proc lock to allocate new pid
     sx_xunlock(&allproc_lock);

     copy the blocks
     bcopy(&p1->p_startcopy, &p2->p_startcopy,
         __rangeof(struct proc, p_startcopy, p_endcopy));

     Zero out the blocks
     bzero(&p2->p_startzero,
         __rangeof(struct proc, p_startzero, p_endzero));

     ...
     Allocate new signals

     Copy File descriptors of all the open files by parent to child
     fd = fdcopy(p1->p_fd);


     In a new process, we created a thread now copy the blocks for this new thread
     bzero(&td2->td_startzero,
         __rangeof(struct thread, td_startzero, td_endzero));

     bcopy(&td->td_startcopy, &td2->td_startcopy,
         __rangeof(struct thread, td_startcopy, td_endcopy));

     bcopy(&p2->p_comm, &td2->td_name, sizeof(td2->td_name));

     ...
     ...

     Copy the vnode pointer of text from parent to child
     p2->p_textvp = p1->p_textvp;   // Vnode of executable. and increase the ref count
     vref(p2->p_textvp);

     ...


     Processing related to process group, child and parent will belong to same process group
     Insert process into process group list


     Associate child with Parent
     pptr = p1;
     p2->p_pptr = pptr;

     Insert child process into sibling list
     LIST_INSERT_HEAD(&pptr->p_children, p2, p_sibling);

     vm_forkproc(td, p2, td2, vm2, flags);
               Function vm_forkproc(td, p2, td2, vm2, flags) {
                    ...
                    ...

                    cpu_fork(td, p2, td2, flags);
                              Function cpu_fork(struct thread *td1, struct proc *p2, struct thread *td2, int flags)
                              {
                                   Copy PCB and the stack
                                   pcb1 = td1->td_pcb;

                                   Allocate PCB for child process
                                   pcb2 = (struct pcb *)(td2->td_kstack +
                                   td2->td_kstack_pages * PAGE_SIZE) - 1;
                                   td2->td_pcb = pcb2;

                                   Copy PCB1 to PCB2 (PCB struct is in pcb.h)
                                   bcopy(td1->td_pcb, pcb2, sizeof(*pcb2));

                                   Create a new stack for a new process
                                   td2->td_frame = (struct trapframe *)((caddr_t)td2->td_pcb - 16) - 1;
                                   bcopy(td1->td_frame, td2->td_frame, sizeof(struct trapframe));
                                   ...
                                   td2->td_frame->tf_eax = 0;          /* Child returns zero */
                                   ...
                                   ...

                                   In instruction pointer of child process we will set the address of trampoline code
                                   pcb2->pcb_eip = (int)fork_trampoline;

                                   //Fork_trampoline code will call fork_return
                                   pcb2->pcb_esi = (int)fork_return;     /* fork_trampoline argument */
                                   pcb2->pcb_ebx = (int)td2;          /* fork_trampoline argument */

                                             Looking into fork_trampoline in i386/exception.s
                                                  ENTRY(fork_trampoline)
                                                            pushl     %esp               /* trapframe pointer */
                                                            pushl     %ebx               /* arg1 */
                                                            pushl     %esi               /* function */
                                                            call     fork_exit
                                                  So, from fork_trampoline we are going to call fork_exit and its argument
                                                  will be above we mentioned.

                                                       //Handle the return of a child process from fork1()
                                                       Function fork_exit(callout, arg, frame)
                                                       {
                                                            ...
                                                            callout(arg, frame); // Here callout is fork_return, as we passed it
                                                                                          as a argument to fork_exit

                                                                 Function fork_return() {
                                                                      userret(td, frame);
                                                                 }
                                                       }

                                                  Once we return from fork_exit we will go to doreti
                                                  which will load our frame and we have will go back to
                                                  fork system call as a child. But the chils process will
                                                  not execute right now because we have not scheduled it
                                                  to run. This is the setup we are preparing and once
                                                  all the setup is done we will schedule the child process
                                                  and it will do as described above
                                                  jmp     doreti

                              } // end of cpu_fork()
               } //end of vm_forkproc


     Put child process on run queue
     sched_add(td2, SRQ_BORING);

     ...
     ...

} //end fork1()
{% endhighlight %}

<br>

## Call Tree

![center-aligned-image](/images/fork1.png)



<br><br>

[2]:	https://github.com/coolgoose85/FreeBSD/tree/master/sys
