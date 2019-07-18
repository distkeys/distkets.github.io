---
layout: post
title: "System Call"
date: 2014-08-03 10:36
categories:
 - Operating Systems

tags:
- '2014'
---

In computing, a system call is how a program requests a service from an operating system’s kernel. This may include hardware related services (e.g. accessing the hard disk), creating and executing new processes, and communicating with integral kernel services (like scheduling). System calls provide an essential interface between a process and the operating system.

<br>

### System Call

Whenever system call is executed from user level a trap is generated and it gets handled at file <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/i386/i386/exception.s#L255" target="_blank">Exception.S (i386\i386)</a>

{% highlight c linenos %}
/*
 * Even though the name says 'int0x80', this is actually a TGT (trap gate)
 * rather then an IGT (interrupt gate).  Thus interrupts are enabled on
 * entry just as they are for a normal syscall.
 */
     SUPERALIGN_TEXT
IDTVEC(int0x80_syscall)
     pushl     $2               /* sizeof "int 0x80" */
     subl     $4,%esp               /* skip over tf_trapno */
     pushal
     pushl     %ds
     pushl     %es
     pushl     %fs
     SET_KERNEL_SREGS
     FAKE_MCOUNT(TF_EIP(%esp))
     pushl     %esp
     call     syscall   ==> from here syscall function is called after pushing some prereq
     add     $4, %esp
     MEXITCOUNT
     jmp     doreti ==> do return from interrupt
{% endhighlight %}

<br>

> *syscall()* is a function in <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/i386/i386/trap.c#L977" target="_blank">Trap.c (i386\i386)</a> is a machine dependent code area.<br>

<br>
Function syscall() is called with trap frame which has input values to system call.

{% highlight c linenos %}
/*
 * Exception/Trap Stack Frame
 */

struct trapframe {
     int     tf_fs;
     int     tf_es;
     int     tf_ds;
     int     tf_edi;
     int     tf_esi;
     int     tf_ebp;
     int     tf_isp;
     int     tf_ebx;
     int     tf_edx;
     int     tf_ecx;
     int     tf_eax;
     int     tf_trapno;
     /* below portion defined in 386 hardware */
     int     tf_err;
     int     tf_eip;
     int     tf_cs;
     int     tf_eflags;
     /* below only when crossing rings (e.g. user to kernel) */
     int     tf_esp;
     int     tf_ss;
};
{% endhighlight %}

<br>

Function syscall() now gets all the values of system call input and store in kernel so that kernel code can access the values while processing system call.

{% highlight c linenos %}
	params = (caddr_t)frame->tf_esp + sizeof(int);
     code = frame->tf_eax;   // code is system call code
     orig_tf_eflags = frame->tf_eflags;
{% endhighlight %}

<br>

Then check we have direct or indirect system call. Indirect system call is one which is like loadable kernel module which loads system call. Once loaded then it can pass the system call number as argument which can be injected in here.

<br>
Then given the system call code we check if we have the code as valid number. If system call code is valid then lookup in system call table which system call is referred else simply store the pointer to indirect system call which is system call 0.
<br>
{% highlight c linenos %}
if (code >= p->p_sysent->sv_size)
           callp = &p->p_sysent->sv_table[0];
       else
           callp = &p->p_sysent->sv_table[code];
{% endhighlight %}
<br>
where **sv_table** entry will have following values

{% highlight c linenos %}
struct sysent {               /* system call table */
     int     sy_narg;     /* number of arguments */
     sy_call_t *sy_call;     /* implementing function */
     au_event_t sy_auevent;     /* audit event associated with syscall */
     systrace_args_func_t sy_systrace_args_func;
                    /* optional argument conversion function. */
     u_int32_t sy_entry;     /* DTrace entry ID for systrace. */
     u_int32_t sy_return;     /* DTrace return ID for systrace. */
};
{% endhighlight %}

<br>
Once we know which system it is then we determine how many arguments it have

> narg = callp->sy_narg;

<br>
Then we copy all the user space arguments into kernel space using function copying which validates the addresses of source and destination etc.

{% highlight c linenos %}
  /*
      * copyin and the ktrsyscall()/ktrsysret() code is MP-aware
      * copy data from params to args
      */
     if (params != NULL && narg != 0)
          error = copyin(params, (caddr_t)args,
                        (u_int)(narg * sizeof(int)));

{% endhighlight %}

<br>
Now, check if we saw any error so far. If we did find the error then we need to return the error but, at this time in kernel it does not know how to return in user space. So, we set the register 0 to be non zero which is the error number and we set another register which is carry bit(eflag). <br>

So, when system call is returned, C lib which issued the system call will check if carry bit(eflag) is set or not. If it is set then it will get the error number from register 0 and map it to human readable error *errorno* and then override the register 0 with -1. <br>
So, say if open system call encounters error then its going to get the error value from register 0; translate it to meaningful error and then override the register 0 with -1. So open returned the value -1 which is failure.

{% highlight c linenos %}
if (error == 0) {
          td->td_retval[0] = 0;
          td->td_retval[1] = frame->tf_edx; // return another value in register 1 if we have anything to return
{% endhighlight %}

Now we call the actual system call
> error = (*callp->sy_call)(td, args);


Here,
*td* is the thread pointer
*args* are the arguments we have copied above from user space to kernel space

We check error and if all fine then we return the values back to user space

{% highlight c linenos %}
switch (error) {
     case 0:
          frame->tf_eax = td->td_retval[0];
          frame->tf_edx = td->td_retval[1];
          frame->tf_eflags &= ~PSL_C;
          break;

else we handle the error
          frame->tf_eax = error;
          frame->tf_eflags |= PSL_C;
{% endhighlight %}

Now we fall back to assembly program from where this function was called.

