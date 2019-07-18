---
layout: post
title: "Signal code in FreeBSD"
date: 2014-12-31 13:00
categories:
 - Operating Systems

tags:
- '2014'
---

Signals are a limited form of inter-process communication used in Unix, Unix-like, and other POSIX-compliant operating systems. A signal is an asynchronous notification sent to a process or to a specific thread within the same process in order to notify it of an event that occurred. Signals have been around since the 1970s Bell Labs Unix and have been more recently specified in the POSIX standard.

{% include toc %}


## Kill System Call
kill - send signal to a process

> int kill(pid_t pid, int sig);

The kill() system call can be used to send any signal to any process group or process.

**If pid is greater than zero:** <br>
 The sig signal is sent to the process whose ID is equal to pid.

**If pid is zero:**  <br>
 The sig signal is sent to all processes whose group ID is equal
 to the process group ID of the sender, and for which the process
 has permission; this is a variant of killpg(2).

**If pid is -1:**  <br>
 If the user has super-user privileges, the signal is sent to all
 processes excluding system processes (with P_SYSTEM flag set),
 process with ID 1 (usually init(8)), and the process sending the
 signal.  If the user is not the super user, the signal is sent to
 all processes with the same uid as the user excluding the process
 sending the signal.  No error is returned if any process could be
 signaled.

{% highlight c linenos %}
Argument to Kill system call (kern_sig.c)
struct kill_args {
     int pid;
     int signum;
};

If process id is greater than 0 then signal has to be send to that particular process else it is either broadcast
signal or own process group etc.

// If process id  > 0
if (uap->pid > 0) {
     Find the process
     if ((p = pfind(uap->pid)) == NULL) {
          If its a zombie process
          if ((p = zpfind(uap->pid)) == NULL) {
               return (ESRCH);
          }
     }
}

Determine if we can send the signal to process

error = p_cansignal(td, p, uap->signum);

In Function p_cansignal(). This function will determine if thread can send signal to processes
cr_cansignal(struct ucred *cred, struct proc *proc, int signum)
{
     Check if process to whom signal is to be send is in Jail semantics

     Check the creds i.e. otherUids, otherGids

     Allow only following signals, for other signals needs special privs
          case 0:
          case SIGKILL:
          case SIGINT:
          case SIGTERM:
          case SIGALRM:
          case SIGSTOP:
          case SIGTTIN:
          case SIGTTOU:
          case SIGTSTP:
          case SIGHUP:
          case SIGUSR1:
          case SIGUSR2:
               break;
          default:
               Need priv

     More priv checks

     Finally yes you can send signal
}
{% endhighlight %}
<br>

![center-aligned-image](/images/kill1.png)

<br><br>

# Post Signal
If we are allowed to send signal to other process then send the signal

Now, it can happen that in multiprocessor env other process may be running or may not. So, we post the signal and when the process come to life it will receive the signal and act appropriately.

{% highlight c linenos %}
//Post signal
psignal(p, uap->signum);
Function tdsignal(struct proc *p, struct thread *td, int sig, ksiginfo_t *ksi)
{
     Determine the property of signal
     //Signal property table is in kern_sig.c
     // #define     SA_KILL          0x01          /* terminates process by default */
     // #define     SA_CORE          0x02          /* ditto and coredumps */
     // #define     SA_STOP          0x04          /* suspend process */
     // #define     SA_TTYSTOP     0x08          /* ditto, from tty */
     // #define     SA_IGNORE     0x10          /* ignore by default */
     // #define     SA_CONT          0x20          /* continue if suspended */
     // #define     SA_CANTMASK     0x40          /* non-maskable, catchable */
     // #define     SA_PROC          0x80          /* deliverable to any thread */
     //static int sigproptbl[NSIG]

     Post Signal to thread in a process
     td = sigtd(p, sig, prop);
     Function sigtd(struct proc *p, int sig, int prop) {
          Check if we are current process
          For each thread in process check if thread is registered for signal handling

          If we can’t find any thread to post signal then just pick the first thread in process
          return the thread pointer
     }

     So far, we have signal to post and the thread pointer where to post
     There is a signal queue. So get the pointer to signal queue.
     There are 2 signal queues
          1. Signal queue for process
          2. Signal queue for thread

     if (SIGISMEMBER(td->td_sigmask, sig))
               sigqueue = &p->p_sigqueue;
     else
               sigqueue = &td->td_sigqueue;

     Check if the signal we will be sending is in our ignore list. If it is in our ignore list then do not
     process further and drop it

     We will now check the preferences/flags marked in the target thread regarding signals
     It can happen that thread have masked all the signals right now i.e. thread may be processing some signal
     and want all other signals to hold until it finishes processing
     if (SIGISMEMBER(td->td_sigmask, sig))
          action = SIG_HOLD;

     If signal is register to be catch
     else if (SIGISMEMBER(ps->ps_sigcatch, sig))
         action = SIG_CATCH;

     Check the property of Signal
     If property of signal is to continue then go through signal queue and delete the signal which will stop
     the execution
     if (prop & SA_CONT)
          sigqueue_delete_stopmask_proc(p);
     else if (prop & SA_STOP) {
          // Stop running the process

     Finally, Add the signal to the signal queue
     ret = sigqueue_add(sigqueue, sig, ksi);

     Notify thread that Signal has been posted
     signotify(td);
     In function signotify() we check that what are all the pending signals for the thread and for which
     signal handler have not masked that means target thread signal handler wants to accept that signal.
     Then we enable the flags so that the target thread can to running state if suspended and when awake
     knows that some signal is posted for the thread
     td->td_flags |= TDF_NEEDSIGCHK | TDF_ASTPENDING;

     ...

     If target process is in sleep state or sleep interruptible state then wake it up
     tdsigwakeup(td, sig, action, intrval);
               Function tdsigwakeup()
               {
                    bump up the thread priority
                    sched_prio(td, PUSER);

                    If thread is in the sleep queue and interruptible
                    then wakeup the thread
                    wakeup_swapper = sleepq_abort(td, intrval);
                                  Function sleepq_abort(struct thread *td, int intrval) {
                                         ...
                                         ...
                                        Enable thread flags
                                        td->td_intrval = intrval;
                                        td->td_flags |= TDF_SLEEPABORT;
                                        ...
                                        get the thread wait channel
                                        wchan = td->td_wchan;

                                        Sleep queue of wait channel where thread resides currently when sleeping
                                        sq = sleepq_lookup(wchan);

                                        //Wake up the thread
                                        return (sleepq_resume_thread(sq, td, 0));
                                                             Whole idea is that there is a hash of all the wait channel
We have already for the wait channel and now there are all the processes
waiting on this wait channel

//Removes a thread from a sleep queue and makes it runnable
Function sleepq_resume_thread(struct sleepqueue *sq, struct thread *td, int pri) {
     ...
     ...
     Remove process/thread from wait channel
     TAILQ_REMOVE(&sq->sq_blocked[td->td_sqqueue], td, td_slpq);
     ...
     ...

     Clear the sleeping flag and set it runnable
     if (TD_IS_SLEEPING(td)) {
          TD_CLR_SLEEPING(td);
          return (setrunnable(td));
                              Function setrunnable(struct thread *td)
          {

               ...
               ...

               sched_wakeup(td);
                    Function sched_wakeup(struct thread *td) {    ====> sched_ule.c
                         This function let schedular know that thread needs to resume
                         ...
                         ...
                         sched_add()
                    }
          } // end setrunnable
     } // end sleepq_resume_thread

} // end sleepq_resume_thread

                                   } // end sleepq_abort
               } //end tdsigwakeup()
} // end tdsignal
{% endhighlight %}

<br>

![center-aligned-image](/images/kill2.png)

<br>

<hr style="border-top: 1.5px dotted black"/>
<br>

# Signal Delivery

So far, in above processing we have loaded the target process to to the run queue and eventually our process run time slice will over and now another process will run.

{% highlight c linenos %}
The switch of process will happen at
Function mi_switch
// The machine independent parts of context switching.

Function mi_switch(int flags, struct thread *newtd)
{
     ...
     ...
     // Schedular come into the picture to switch the thread
     sched_switch(td, newtd, flags);
               Function sched_switch(struct thread *td, struct thread *newtd, int flags)
               {
                    Save information related to thread before context switching
                    cpuid = PCPU_GET(cpuid);
                    tdq = TDQ_CPU(cpuid);
                    ts = td->td_sched;
                    mtx = td->td_lock;
                    ts->ts_rltick = ticks;
                    td->td_lastcpu = td->td_oncpu;
                    td->td_oncpu = NOCPU;
                    td->td_flags &= ~TDF_NEEDRESCHED;
                    td->td_owepreempt = 0;
                    tdq->tdq_switchcnt++;
                    ...
                    ...

                    choose new thread to run
                    newtd = choosethread();
                         Function choosethread() {
                              ...
                              td = sched_choose();
                                   Function sched_choose() {
                                        Choose next high priority thread to run from the same CPU
                                        tdq = TDQ_SELF();
                                        td = tdq_choose(tdq);
                                             Function tdq_choose(struct tdq *tdq) {
                                                  Pick the thread from realtime thread queue
                                                  td = runq_choose(&tdq->tdq_realtime);

                                                  If its NULL, then pick from Timshare queue
                                                  td = runq_choose_from(&tdq->tdq_timeshare, tdq->tdq_ridx);

                                                  If its NULL, then pick from Idle queue
                                                  td = runq_choose(&tdq->tdq_idle);
                                                            In function runq_choose we simply go to queue and get the
                                                            first item from the queue

                                             } //end tdq_choose

                                        ...
                                        return the thread

                                   } //end sched_choose
                         } //end choosethread

                    So, newtd is the newthread we choose to run
                    This function will switch the threads and newtd will be running after this function call
                    This function is a assembly level code
                    cpu_switch(td, newtd, mtx);

               } //end sched_switch
} //end mi_switch

Now we have target thread running.
When we were posting the signal we enabled the flag TDF_ASTPENDING in function
     signotify(td);
     In function signotify() we check that what are all the pending signals for the thread and for which signal
     handler have not masked that means target thread signal handler wants to accept that signal.
     Then we enable the flags so that the target thread can to running state if suspended and when awake
     knows that some signal is posted for the thread

     td->td_flags |= TDF_NEEDSIGCHK | TDF_ASTPENDING;

Since, this flag is enable when the target process comes into like it will check this flag and if it is enable
then it will call function AST (Asynchronous software trap)
{% endhighlight %}

<br>

![center-aligned-image](/images/ast.png)

<br>

{% highlight c linenos %}
Function ast(struct trapframe *framep)
{
     ...
     ...
     ...
     Check if any signals are posted
     if (flags & TDF_NEEDSIGCHK) {
          PROC_LOCK(p);
          mtx_lock(&p->p_sigacts->ps_mtx);

          This function will get the signal posted and which is unmasked
          We will process all the pending signals one-by-one
          while ((sig = cursig(td)) != 0)
               postsig(sig);
               mtx_unlock(&p->p_sigacts->ps_mtx);
               PROC_UNLOCK(p);

                    Function cursig(struct thread *td) {
                         ...

                         Return if there is signal to process or 0 when nothing to process
                         return (SIGPENDING(td) ? issignal(td) : 0);
                                   Function issignal(td) {
                                        Find any pending signal
                                        sigpending = td->td_sigqueue.sq_signals;

                                        Get first pending signal
                                        sig = sig_ffs(&sigpending);

                                        Handle special cases of SIGSTOP or SIGCONT

                                        Get properties of signal
                                        prop = sigprop(sig);

                                        Handle other cases i.e. Default Signal, Ignore Signal etc

                                        return signal
                                   } //end of issignal(td)
                    } //end of cursig(struct thread *td)


                    // This function take action for specified signal
                    Function postsig(sig) {
                         register struct proc *p = td->td_proc;
                         ...
                         ...
                         Get signal action
                         action = ps->ps_sigact[_SIG_IDX(sig)];

                         Now we will check action and process accordingly
                         //Default action which is kill the process
                         if (action == SIG_DFL) {
                              sigexit(td, sig);

                                   Function sigexit(td, sig) {
                                        ...
                                        exit1(td, W_EXITCODE(0, sig));
                                   }
                         } else {
                              ...
                              get signal mask
                              returnmask = td->td_sigmask;
                              ...
                              // Send signal to signal handler
                              // In process structure
                              // struct sysentvec *p_sysent;     /* (b) Syscall dispatch info. */
                              (*p->p_sysent->sv_sendsig)(action, &ksi, &returnmask);

                                        // i386/machdep.c
                                        Function sendsig(sig_t catcher, ksiginfo_t *ksi, sigset_t *mask) {
                                             struct sigframe sf;
                                             ...
                                             When stack is context switched it will save the stack frame,
					     register contents etc
                                             Get the stack frame pointer of thread
                                             regs = td->td_frame;
                                             ...

                                             Make a copy of stack frame because we will be changing the
					     frame to handle the signal.
                                             Once the signal is handled and the processing is done this
					     stack frame will be brought back.
                                             bcopy(regs, &sf.sf_uc.uc_mcontext.mc_fs, sizeof(*regs));

                                             Allocate space for signal handler context and put that in stack frame
                                             sp = (char *)regs->tf_esp - sizeof(struct sigframe);
                                             ...

                                             Build argument list for signal handler
                                             sf.sf_signum = sig;

                                             Pointer to signal context
                                             sf.sf_ucontext = (register_t)&sfp->sf_uc;

                                             We are in kernel space currently, we will be going to user space soon
                                             and we want the signal catcher code to be executed in user space
                                             We are assigning the address of catcher function to be called from user space
                                             sf.sf_ahu.sf_action = (__siginfohandler_t *)catcher;
                                             ...
                                             ...

                                             Copy the signal frame to user's stack
                                             if (copyout(&sf, sfp, sizeof(*sfp)) != 0) {
                                                  sigexit(td, SIGILL);
                                             }

                                             Stack frame pointer we got above "regs" will copy sigtramp() to
					     instruction pointer. So, after returning from here we think
					     we will return from ast() and go to user code back but
                                             instead now we will be going to trampoline code.

                                             regs->tf_eip = PS_STRINGS - szfreebsd4_sigcode;
                                        } // end sendsig
                         }
                    } //end postsig()
     }
} //end of ast()

In i386/locore.s code for signal trampoline in user space
NON_GPROF_ENTRY(sigcode)
     calll     *SIGF_HANDLER(%esp)
     leal     SIGF_UC(%esp),%eax     /* get ucontext */
     pushl     %eax
...
...
     movl $SYS_sigreturn,%eax

Function sigreturn()
Function sigreturn(td, uap)
{
     regs = td->td_frame;
     ...
     ...
     lot of snaity checking
     ...
     ...
     bcopy(&ucp->uc_mcontext.mc_fs, regs, sizeof(*regs));
     ...

}
{% endhighlight %}

<br><br><br>

[1]:	http://en.wikipedia.org/wiki/Unix_signal
