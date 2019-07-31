---
title: "Process synchronization in Operating Systems"
date: 2013-10-07 11:54
comments: true
categories:
 - Operating Systems
tags:
 - '2013'
---
{% include toc %}

<br>
### What is a Process?

Operating system(OS) objective is to keep as many as of the computer resources as busy as possible. It is used to keep track of all the things an OS must remember about the state of the user program.

The process is like a box, a complete entity in itself which does a step by step task written in the program. More formally it is called a program in execution.

Let's consider a fundamental operating system with very least complexity. This operating system can run only one process at a time. Since only one process is working at a time, all the resources occupied by the process may not be used at the same time. To maximize resource utilization, we need to have entities running at the same time. For multiple entities, it is logical that either we need to have multiple processes running at the same time or lightweight multiple entities running inside process as a part of the process.


> Process = Code + Allocated Resources + Bookkeeping information.



Let's explore the second option, now consider process is like a box, and it has resources inside the box. We create multiple children of the process, which is called thread.

Thread is a child of process, and hence, it uses resources of process. Theoretically, there is no limit on the number of child threads a process can have, but it seems logical that process should have enough resource for the administrative purpose for these threads.

Once there are multiple threads, they are going to ask for the same resource at the same time. For example, if two children are in one room, then they always fight for the same toy. Same applies to threads.

<br><br>
### 3 Issues with Sharing
1. How to Share data?
2. How to ensure threads in a process, executes one at a time?
3. How to ensure proper sequencing of events?

To understand it better, lets take a real world example.

<br><br>
### Carl’s Jr. Restaurant
Process

1. Customer arrives
2. Employee takes order
3. Employee cooks food
4. Employee bag food
5. Employee takes money
6. Customer gets food and leaves

If a single employee is doing steps from 1-6, then all other customers have to wait in line, and it's going to be a long wait. Instead, let's have multiple employees for taking the order, cook food, bag food, take money. Each of these ‘employee’ is multiple threads on Process ‘Restaurant.’ Each thread is responsible for doing a specialized task.

Lets associate 3 issues in current situation

1. What is shared data? - In step 2-3, Quantity of food. In step 3-4, how much food to bag
2. Does sequence matters? -  Cook can't cook food until order arrives. Employee can't bag food until it is cooked. So, sequencing matters.

Shared data can be passed for sharing either using message passing or storing that data in global memory of the process, and each thread read from that memory location.

The next logical question is how to ensure threads in a process executes one at a time, i.e., in exclusion?
More formally there are three types of solution categories

1. Algorithmic approach
2. Software Primitives
3. Concurrent programming construct

<br><br>
### Algorithmic approach
The algorithmic approach to process synchronization does not use any assistance from the computer architecture or the OS kernel. Instead it uses an arrangement of logical conditions to satisfy the desired synchronization requirements. <a href="http://books.google.com/books/about/Operating_Systems.html?id=kbBn4X9x2mcC" target="_blank">[Dhamdhere]</a>

* Two process algorithms
* <a href="http://en.wikipedia.org/wiki/Dekker's_algorithm" target="_blank">Dekker's Algorithm</a>
* <a href="http://en.wikipedia.org/wiki/Peterson's_algorithm" target="_blank">Peterson's Algorithm</a>
* n process algorithm
* <a href="http://en.wikipedia.org/wiki/Lamport's_bakery_algorithm" target="_blank">Bakery's Algorithm</a>

<br><br>
### Software Primitives
A set of software primitives for mutual exclusion, e.g., Semaphore, Locks, etc. were developed to overcome the logical complexity of algorithmic implementations. This is implemented using some special architectural features of computer systems. However, ease of use and correctness remained the major obstacle in the development of large concurrent systems.

<br><br>
### Semaphores
It is a shared integer variable with non-negative values that have initialization, wait and signal as a indivisible operation.

{% highlight c linenos %}
class Semaphore {
    public:
	//Constructor
	Semaphore(char *debugName, int initialValue);

	//Destructor
	~Semaphore();

    private:
	int value;
	Queue *waitQueue;
	char *name;
};
{% endhighlight %}


{% highlight c linenos %}
Semaphore(char * debugName, int initialValue) {
    name      = debugName;
    value     = initialValue;
    waitQueue = new Queue;
}
{% endhighlight %}

{% highlight c linenos %}
~Semaphore() {
    delete waitQueue;
}
{% endhighlight %}

{% highlight c linenos %}
//P() - Semaphore Wait
Semaphore::P() {
    //Disable interrupts
    IntStatus oldLevel = interrupt->SetLevel(IntOff);

    //Semaphore not available
    while (value == 0) {
	waitQueue->Append((void *)currentThread);
	currentThread->Sleep();
    }

    //Semaphore now availble
    value--;
    (void)interrupt->SetLevel(oldLevel);
}
{% endhighlight %}

{% highlight c linenos %}
//Semaphore Signal
Semaphore::V() {
    Thread *thread;

    IntStatus oldLevel = interrupt->SetLevel(IntOff);

    //Remove first thread from wait queue
    thread->(Thread *)waitQueue->Remove();

    if (thread != NULL) {
	scheduler->ReadyToRun(thread);
    }

    value++;

    (void)interrupt->SetLevel(oldLevel);
}
{% endhighlight %}

<br><br>
### Locks
The basic idea is to close/acquire a lock at the start of critical section or an indivisible operation and open/release it at the end of the critical section or the indivisible operation.

Locks solves how to ensure threads in a process executes one at a time but not the sequencing problem.

Lock Class

{% highlight c linenos %}
class Lock {
    public:
	Lock (char *debugName);
	~Lock();

	char* getName() { return name; }

	void acquire();
	void release();
	bool isHeldByCurrentThread;

    private:
	char*   name;
	Queue*   lockWaitQueue;
	bool    lockFree;
	Thread* currentLockThread;
};
{% endhighlight %}

{% highlight c linenos %}
//Lock Constructor
Lock::Lock(char * debugName) {
    name              = debugName;
    currentLockThread = NULL;
    lockFree          = TRUE;
    lockWaitQueue     = new Queue;
}
{% endhighlight %}

{% highlight c linenos %}
//Lock Destructor
~Lock() {
    delete lockWaitQueue;
}
{% endhighlight %}

{% highlight c linenos %}
Lock::acquire() {
    //Disable interrupts
    IntStatus oldLevel = interrupt->SetLevel(IntOff);

    //Check if current thread is an owner
    if (currentThread == currentLockThread) {
	//Already owner
	(void)interrupt->SetLevel(oldLevel);
	return;
    }

    if(lockFree == TRUE) {
	lockFree = FALSE;
	currentLockThread = currentThread;
    } else {
	lockWaitQueue->Append((void*) currentThread);
	currentThread->Sleep();
    }

    (void)interrupt->SetLevel(oldLevel);
}
{% endhighlight %}

{% highlight c linenos %}
Lock::release() {
    Thread* waitingThread;
    IntStatus oldLevel = interrupt->SetLevel(IntOff);

    if (!isHeldByCurrentThread()) {
	//Thread is not valid owner of lock its
	//trying to release
	DEBUG("Not a lock owner");

	(void)interrupt->SetLevel(oldLevel);
	return;
    }

    waitingThread = (Thread*)lockWaitQueue->Remove();

    if (waitingThread != NULL) {
	scheduler->ReadyToRun(waitingThread);
	currentLockThread = waitingThread;
    } else {
	lockFree = TRUE;
	currentLockThread = NULL;
    }

    (void)interrupt->SetLevel(oldLevel);
}
{% endhighlight %}

{% highlight c linenos %}
bool Lock::isHeldByCurrentThread() {
    return ((currentThread != currentLockThread) ?  FALSE : TRUE);
}
{% endhighlight %}

<br><br>
### Concurrent Programming Construct
Locks can only solve mutual exclusion problem, they can not solve sequencing problem. We need another mechanism i.e. Monitors

Monitors is a programming language construct that supports both data access synchronization and control synchronization.

Monitors have 3 parts

1. Lock for mutual exclusion
2. 1 or more condition variables for sequencing
3. Monitor variables to make sequencing decisions i.e. Shared data

<br><br>
### Condition Variables
Each condition variable is only associated with one lock.

{% highlight c linenos %}
class Condition {
    public:
	Condition(char *debugName);
	~Conditon();

	char* getName() { return name; }

	void wait(Lock* conditionLock);
	void signal(Lock* conditionLock);
	void broadcast(Lock* conditionLock);

    private:
	char* name;
	Queue* cvQueue;
	Lock* cvLock;
};
{% endhighlight %}

{% highlight c linenos %}
Condition::Condition(char * debugName) {
    name    = debugName;
    cvQueue = new Queue;
    cvLock  = NULL;
}
{% endhighlight %}

{% highlight c linenos %}
Condition::~Condition() {
    delete cvQueue;
}
{% endhighlight %}

{% highlight c linenos %}
void Condition::wait(Lock* conditionLock) {
    IntStatus oldLevel = interrupt->SetLevel(IntOff);

    if (cvQueue->isEmpty()) {
	//This lock is now associated with CV and
	//only removed when last entry is removed
	//from cvQueue.
	cvLock = conditionLock;
    }

    if (conditionLock == NULL) {
	interrupt->SetLevel(oldLevel);
	return;
    }

    conditionLock->release();
    cvQueue->Append((void*) currentThread);
    currentThread->Sleep();

    //Acquire lock when get up
    conditionLock->acquire();
    interrupt->SetLevel(oldLevel);
}
{% endhighlight %}

{% highlight c linenos %}
void Conditon::signal(Lock * conditionLock) {
    IntStatus oldLevel = interrupt->SetLevel(IntOff);

    //If nobody to signal, return
    if (cvQueue->Empty()) {
	interrupt->SetLevel(oldLevel);
	return;
    }

    //Verify right lock is signalled
    if (cvLock != conditionLock) {
	interrupt->SetLevel(oldLevel);
	return;
    }

    thread = cvQueue->Remove();
    if (thread != NULL) {
	scheduler->ReadyToRun(thread);
    }

    if (cvQueue->isEmpty()) {
	cvLock = NULL;
    }

    interrupt->SetLevel(oldLevel);
}
{% endhighlight %}

<br><br>
### Producer-Consumer Problem

Lets consider we have infinite buffer

monitor variable  => int itemCount = 0;<br>
monitor lock      => monitorLock;<br>
monitor condition => needItem;<br>

{% highlight c linenos %}
while (true) {
    monitorLock.acquire();

    //Produce item
    //Put in a buffer
    itemCount++;

    needItem.signal(&monitorLock);

    monitorLock.Release();
}
{% endhighlight %}

{% highlight c linenos %}
while (true) {
    monitorLock.acquire();

    while(intemCount == 0) {
	needItem.wait(&monitorLock);
    }

    //Buffer has atleast one item
    itemCount--;

    monitorLock.Relase();
}
{% endhighlight %}

<br><br>
### Producer-Consumer Print FooBar Problem
{% highlight c++ linenos %}
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;

int produced = 0;

class FooBar {
private:
    int n;

public:
    FooBar(int n) {
        this->n = n;
    }

    void printFoo() { cout << "Foo"; }
    void printBar() { cout << "Bar"; }

    void foo(function<void()> printFoo) {
        std::unique_lock<std::mutex> lck(mtx);

        for (int i = 0; i < n; i++) {
            if(produced == 1){
              cv.wait(lck);
            }

            printFoo();

            produced = 1;
            cv.notify_one();
        }
    }

    void bar(function<void()> printBar) {
        std::unique_lock<std::mutex> lck(mtx);

        for (int i = 0; i < n; i++) {
            if(produced == 0) {
                cv.wait(lck);
            }

            printBar();

            produced = 0;
            cv.notify_one();
        }
    }
};
{% endhighlight %}


### Recommended reading

* <a href="https://www.dropbox.com/s/8naej9kd0612gkr/implementingcvs.pdf" target="_blank">Implementing CV using semaphore</a>
* <a href="https://www.dropbox.com/s/gaallrwximrm14g/Monitors.pdf" target="_blank">Monitors by C.A.R Hoare </a>
* <a href="http://distkeys.com/blog/2013/10/07/process-synchronization-in-linux-kernel/" target="_blank">Process synchronization in Linux Kernel</a>
