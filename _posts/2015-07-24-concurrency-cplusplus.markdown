---
layout: post
title: "Concurrency in C++"
date: 2015-07-24 22:12
comments: true
categories:
 - Operating Systems
 - C++

tags:
- '2015'
---

This post is going to explore Concurrency using C++. These posts [Process synchronization in Operating Systems](http://distkeys.com/operating%20systems/2013/10/07/process-synchronization-in-os.html)and [Process synchronization in Linux Kernel](http://distkeys.com/operating%20systems/linux/2013/10/07/process-synchronization-in-linux-kernel.html) explains the theory in detail.

<br>

{% include toc %}

<br><br>
## Thread Basics

### Thread API's in C++

| Function | Description |
|:--------|:-------|
| t.join() | Waits until thread t has finished its executable unit. |
| t.detach() | Executes the created thread t independently of the creator. |
| t.joinable() | Returns true if thread t is still joinable. |
| t.get_id() | Returns the identity of the thread. |
| thread::hardware_concurrency() | Returns the number of cores, or 0 if the runtime can not determine the number. Indicates the number of threads that can be run concurrently.  |
| this_thread::sleep_until(absTime) | Puts thread t to sleep until the time point absTime. Needs a time point or a time duration as an argument. |
| this_thread::sleep_for(relTime) | Puts thread t to sleep for the time duration relTime. Needs a time point or a time duration as an argument. |
| this_thread::yield() | Enables the system to run another thread. |
| t.swap(t2) | Swaps the threads. |
| swap(t1, t2) | Swaps the threads. |

<br><br>
### Create a thread

Thread can be created by calling std::thread t1(). Thread needs a starting point of the execution which is particularly a function.

{% highlight c++ %}
// Thread creation with non-class function
std::thread t2(nonClassFunction, "Inside non-class function");

// Thread creation with class function
std::thread t1(&printing::classFunctionPrint, obj);

{% endhighlight %}

<br>

{% highlight c++ linenos %}
#include <iostream>
#include <thread>

using namespace std;

class printing {
private:
    string name;
public:
    printing() {}
    printing(string n):name(n) {}

    void classFunctionPrint() {
        cout << name << endl;
    }
};

void nonClassFunction(string n)
{
    cout << n << endl;
}


int main(int argc, const char * argv[]) {

    // Thread create for class function
    printing *obj = new printing("Inside class function");

    std::thread t1(&printing::classFunctionPrint, obj);

    t1.join();
    delete obj;

    // Thread create for non-class function
    std::thread t2(nonClassFunction, "Inside non-class function");
    t2.join();

    return 0;
}
{% endhighlight %}

<br><br>
### Thread ID and Cores

{% highlight c++ linenos %}
#include <iostream>
#include <thread>

using namespace std;

class printing {
private:
    string name;
public:
    printing() {}
    printing(string n):name(n) {}

    void classFunctionPrint() {
        cout << name << endl;
    }
};

void nonClassFunction(string n)
{
    cout << n << endl;
}


int main(int argc, const char * argv[]) {

    cout << "Total number of Cores = "<< thread::hardware_concurrency() << endl;

    // Thread create for class function
    printing *p = new printing("Inside class function");

    std::thread t1(&printing::classFunctionPrint, p);
    cout << "FROM MAIN: id of t1 " << t1.get_id() << endl;

    // Thread create for non-class function
    std::thread t2(nonClassFunction, "Inside non-class function");
    cout << "FROM MAIN: id of t2 " << t2.get_id() << endl;

    t1.join();
    delete p;
    t2.join();

    return 0;
}

{% endhighlight %}

{% highlight c++ %}
Output:
Total number of Cores = 4
FROM MAIN: id of t1 0x700008ac1000
FROM MAIN: id of t2 0x700008b44000
Inside class function
Inside non-class function
{% endhighlight %}


<br><br>
## Locks

### Without Lock

This code creates two threads and they both try to increment the counter without synchronization primitives.

{% highlight c++ linenos %}
#include <iostream>
#include <thread>

using namespace std;

class printing {
private:
    string name;
    int count;
public:
    printing() {}
    printing(string n, int c):name(n), count(c) {}

    void classFunctionPrint() {
        cout << name << endl;
    }

    void incrementAndPrint(string threadName) {
        cout << "thread: " << threadName << " incrementing count from " << count << " to " << count + 1 << endl;
        count++;
        cout << "thread: " << threadName << " incremented count from " << count - 1 << " to " << count << endl;
    }
};

void nonClassFunction(string n)
{
    cout << n << endl;
}


int main(int argc, const char * argv[]) {

    cout << "Total number of Cores = "<< thread::hardware_concurrency() << endl;

    // Thread create for class function
    printing *p = new printing("Inside class function", 0);


    std::thread t3(&printing::incrementAndPrint, p,"thread3");
    std::thread t4(&printing::incrementAndPrint, p,"thread4");


    t3.join();
    t4.join();
    delete p;


    return 0;
}
{% endhighlight %}

{% highlight c++ %}
Total number of Cores = 4
thread: thread3 incrementing count from 0 to 1
thread: thread: thread4thread3 incremented count from  incrementing count from 0 to 1 to 2
thread: thread41
 incremented count from 1 to 2
{% endhighlight %}

As we can see the output is all messed up.


<br>
### With Mutex

{% highlight c++ %}
#include <mutex>

std::mutex coutMutex;
coutMutex.lock();
coutMutex.unlock();
{% endhighlight %}

<br>

{% highlight c++ linenos %}
#include <iostream>
#include <thread>
#include <mutex>

using namespace std;

std::mutex coutMutex;

class printing {
private:
    string name;
    int count;
public:
    printing() {}
    printing(string n, int c):name(n), count(c) {}

    void classFunctionPrint() {
        cout << name << endl;
    }

    void incrementAndPrint(string threadName) {
        coutMutex.lock();
        cout << "thread: " << threadName << " incrementing count from " << count << " to " << count + 1 << endl;
        count++;
        cout << "thread: " << threadName << " incremented count from " << count - 1 << " to " << count << endl;
        coutMutex.unlock();
    }
};

void nonClassFunction(string n)
{
    cout << n << endl;
}


int main(int argc, const char * argv[]) {

    cout << "Total number of Cores = "<< thread::hardware_concurrency() << endl;

    // Thread create for class function
    printing *p = new printing("Inside class function", 0);


    std::thread t3(&printing::incrementAndPrint, p,"thread3");
    std::thread t4(&printing::incrementAndPrint, p,"thread4");


    t3.join();
    t4.join();
    delete p;


    return 0;
}

{% endhighlight %}
{% highlight c++ %}
Output:
Total number of Cores = 4
thread: thread3 incrementing count from 0 to 1
thread: thread3 incremented count from 0 to 1
thread: thread4 incrementing count from 1 to 2
thread: thread4 incremented count from 1 to 2
{% endhighlight %}


<br><br>
### Types of Mutexes

| Classes | Member Functions | Constants |
|:--------|:-------|:-------|
| <mark>Mutexes</mark> |  | |
| mutex | lock <br> unlock <br> try_lock |  |
| recursive_mutex | lock <br> unlock <br> try_lock |  |
| timed_mutex | lock <br> unlock <br> try_lock <br> try_lock_for <br> try_lock_until |  |
| recursive_timed_mutex | lock <br> unlock <br> try_lock <br> try_lock_for <br> try_lock_until |  |
| <mark>Locks</mark> |  | |
| lock_guard | | adopt_lock_t <br> defer_lock_t|
| unique_lock | | adopt_lock_t <br> defer_lock_t |


We have seen basic mutex above and other kind of mutexes are just the version of basic mutex with time properties.

Mutex help us get atomically inside critical section to perform operation but situation can be that in critical section thread t1 tries to acquire another lock L2 which is busy currently. In this case current thread t1 has to wait for lock L2 but holding lock L1. A deadlock can occur if other thread t2 is in same situation waiting for release of lock L1 from the thread t1 but holding lock L1.

Similar deadlock can also occur if thread t1 forgets to release lock possibly due to bug. In this situation, *lock_guard* comes to the rescue. *lock_guard* limits the life of lock until the scope of curly braces and then the lock is relesed.

```
{
  std::mutex m,
  std::lock_guard<std::mutex> lockGuard(m);
  sharedVariable= getVar();
}

mutex m will be released after curly braces even though unlock() is not called.
```

<br>
#### lock_guard vs unique_lock
At this point, we have two issues i.e.

- Lock entity itself (mutex) i.e. lock object and various functions which can act on lock object i.e. lock(), unlock()
- Management of locks i.e. acquiring and reliabily releasing locks, lifecycle of locks


As pointed above, deadlocks are primarily result of management of locks. To solve deadlock problem the approach is acquire all the locks together and if all the lock can not be acquired together then wait until it can be, until then no locks are acquired by the thread.

Once the locks are acquired the next step is ensure the lifecycle of locks are healty i.e. by end of our work locks are released.

For lifecycle and management we can use *lock_guard* or *unique_lock*.

- *lock_guard* - It is locked only once on construction and unlocked on destruction.
- *unique_lock* - You can lock and unlock a std::unique_lock any number of times in the method. This class guarantees an unlocked status on destruction.


We have two constants which works with above locking management

- *adopt_lock* - Assumes that mutex object is already locked by the current thread
- *defer_lock* - Makes it not to lock the mutex object automatically on construction
- If none are provided then mutex is locked by *lock_guard* or *unique_lock*


```
// Case 1
std::mutex m;
std::lock_guard<std::mutex> lockGuard(m);  <== Acquires lock

// Case 2
std::unique_lock<std::mutex> lockGuard(m); <== Acquires lock


// Case 3
{
    std::mutex m1;
    std::mutex m2;

    unique_lock<mutex> guard1(m1, defer_lock);
    unique_lock<mutex> guard2(m2, defer_lock);

    lock(guard1,guard2);
} <== Locks are released at the end of scope

// Case 4
{
    std::mutex m1;
    std::mutex m2;

    lock(guard1,guard2);
    unique_lock<mutex> guard1(m1, adopt_lock);
    unique_lock<mutex> guard2(m2, adopt_lock);
} <== Locks are released at the end of scope

// Case 5
{
    std::mutex m1;
    std::mutex m2;

    lock(guard1,guard2);
    std::lock_guard<std::mutex> guard1(m1, adopt_lock);
    std::lock_guard<std::mutex> guard2(m2, adopt_lock);
    ...
} <== Locks are released at the end of scope

// Case 6
{
    std::mutex m1;
    std::mutex m2;

    std::lock_guard<std::mutex> guard1(m1, defer_lock);
    std::lock_guard<std::mutex> guard2(m2, defer_lock);

    lock(guard1,guard2);
    ...
} <== Locks are released at the end of scope
```







<br><br><br><br><br><br>






