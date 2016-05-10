# 28 锁

在引入的并发介绍中，我们看到并行编程的根本问题的之一：我们想执行一系列的原子指令，但由于在单个处理器中断现象（或同时在多个处理器上执行多个线程），我们不能。在本章中，我们这样直接攻克这个问题，通过引入称为锁的东西。程序员用锁包围源代码，把他们放在临界区中，从而确保任何这样关键部分执行就好像它是一个单一的原子指令。

## 28.1 锁：基本想法

下面例子，假设我们的临界区看起来像这样，更新一个共享变量的权威方法如下：

```
balance = balance + 1;
```

当然，其他的临界区也是可行的，比如向一个链表添加元素或者更多的复杂更新数据结构，但是我们仅仅现在保留这个简单的例子。如果要使用一个锁，我们添加一些代码在临界区周围，如下所示：

```
1 lock_t mutex;
2 ...
3 lock(&mutex);
4 balance = balance + 1;
5 unlock(&mutex);
```

一个锁仅仅就是一个变量，如果要使用当然要先定义某种类型的锁变量（比如上面提到的mutex）。这个锁变量在任何时候都维护着这个锁的状态。它要么是可用的（unlocked或者free），此时没有线程持有这个锁；要么是被持有着（locked或者hold），此时正好有一个线程持有着这个锁并且有可能正在临界区中内。我们也可以在这个数据类型上存储其他信息，比如哪个线程持有锁，或者一个维护者请求锁队列，但这样的信息是对用户是隐藏的。

首先```lock()```和```unlock()```的程序语义很简单。调用程序```lock()```试图获取锁；如果没有其他线程持有锁（比如它处于自由状态），该线程将获得锁并进入临界区；这个线程也被称作是锁的拥有者。 如果另一个线程在同一个锁变量上调用```lock()```（mutex这个例子），只要该锁被另一个线程持有，它就不会返回；在这种方式之下，当一个线程持有时其他线程会被阻止进入临界区。

一旦锁的持有者调用```unlock()```，锁就变得可用了（free）。如果没有其他线程在等待锁（即没有其他线程调用了```lock()```后被阻塞在那里），锁的状态简简单单的被切换成free。如果有等待的线程（阻塞在```lock()```处），其中一个线程将（最终）观察到（或被告知的）锁状态的这种变化，获得锁并进入临界区。

锁向程序员提供一些最低限度的调度控制。在一般情况下，我们可以认为，不管操作系统如何选择，线程是由程序员创建，但是是由操作系统调度的实体。锁将一些控制权交还给程序员；通过将一段代码段加上锁，程序员可以保证这段代码中不超过一个线程中是活跃的。因此这样可以帮助改变传统的操作系统调度的混乱状态到一个被更好的控制的状态。

## 28.2 Pthread Locks

POSIX库使用互斥量作为锁的名称，因为锁是用来提供线程之间的相斥性，如果一个线程处于临界区，它会阻止其他线程进入临界区直到它已完成因此，当你看到下面的POSIX线程的代码，你应该明白这是做与上文一样的事情（我们再次使用我们的包装是检查在锁和解锁上的错误）：

```
1 pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
2 
3 Pthread_mutex_lock(&lock); // wrapper for pthread_mutex_lock()
4 balance = balance + 1;
5 Pthread_mutex_unlock(&lock);
```

> THE CRUX：如何构建一把锁？
> 我们如何构建一个有效的锁？高效的锁以最小代价提供了互斥，同时也可以获得一些将要讨论的其他的属性。我们需要什么硬件支持和操作系统支持呢？

您可能还注意到这里说的POSIX版本传递一个变量给lock和unlock，因为我们可能会使用不同的锁来保护不同变量。这样做可以提高并发性：更倾向于使用不同的锁保护不同的数据和数据结构，从而使更多的线程获取到锁（一**更细粒度**的方法），而不是在任何时间任何临界区使用同一把锁（**粗粒**锁定策略）。

## 28.3 构建一把锁

现在，你应该从程序员的角度来看对锁的工作原理有一定的了解。但是，我们应该如何构建一把锁？需要什么样的硬件支持？什么操作系统支持？这正是在本章的其他部分我们将提出的一系列问题。

要建立一个工作锁，我们需要从我们的老朋友硬件以及我们的好伙计操作系统上获取一些帮助。多年来，许多不同的硬件原语的已被添加到各种计算机体系结构的指令集合中；虽然我们不会研究如何将这些指令如何实现的（计算机结构课的主题），但是我们将研究如何使用它们才能建立一个互斥原语就像一把锁。我们也将研究操作系统如何参与，使我们能够建立一个复杂的锁定库。

## 28.4 锁的评估

在构建任何锁之前，我们应该先了解一下我们的目标是什么，因此，我们问如何评价一个特定锁实现的效率。为了评估一个锁是否有效（并且效果很好），我们首先应该建立一些基本准则。

第一点是**提供互斥**。锁是否确实完成了基本任务，也就是能否防止多线程进入一个临界区？

第二点是**公平性**。每个线程是否在锁自由的时候以一个公平的机会抢占？看看这另一种方式是通过检查更加极端的情况：没有任何线程因为抢占锁饥饿，也就是从来没有得到它？

第三点是**性能**。特别使用锁增加的时间开销。有几种不同的情况下，这是值得考虑的。一个是没有竞争的情况；当一个正在运行的线程获取和释放锁，什么是这样做的开销？另一种是多个线程竞争的情况下在单CPU上的锁；在这种情况下，是否存在性能问题？最后，当有涉及到多个CPU如何锁定执行，并在每个线程争用锁？通过比较这些不同的方案，我们可以更好地了解使用对性能的影响各种锁定技术，如下所述。

## 28.5 控制中断

## 28.6 Test And Set (Atomic Exchange)

因为禁用中断在多处理器情形下不适用，系统设计者开始发明支持锁的硬件。最早的多处理器系统，如Burroughs B5000在1960年[M82]支持。今天所有的系统都提供这种支持，即使是单处理器系统。

最容易理解的硬件支持锁的部分是有名是**test-and-set指令**，也被称作是**原子交换**。为了了解它是如何工作的，让我们首先尝试不用它构建一个锁。这在个失败的尝试中，我们使用简单的标志变量来表明锁是否被持有。
```
typedef struct __lock_t { int flat;} lock_t;

void init(lock_t *mutex) {
    // 0 -> lock is available, 1 ->held
    mutex->flag = 0;
}

void lock(lock_t *mutex) {
    while (mutex->flat == 1) // TEST the flag;
        ; // spin-wait (do nothing)
    mutex->flag = 1;
}

void unlock(lock_t *mutex) {
    mutex->flag = 0;
}
```
**Figure 28.1: First Attemp: A Simple Flag**

在最早的尝试中（图28.1），这个想法十分简单：用一个简单的变量来表明某些线程是否持有一个锁。第一个进入临界区的线程会调用```lock()```，它会**测试**表示是否为1（这个示例中，它并不是），然后**设置**这个标志位1来表明线程**持有**着这个锁。当从临界区结束，该线程会调用```unlock()```并清空标志，从而表明锁不再被持有着。

如果正好有其他线程在调用```call()```当第一个线程在临界区中，它会简单的在while循环中**自旋等待**另一个线程调用```unlock()```并清空标志。一旦第一个线程这样做了，这个等待的线程会从while循环中退出，为自己设置标志位1并进入临界区。

不幸的事，这段代码有两个问题：第一个是正确性，第二个是性能。正确性的问题很简单看出来，一旦你习惯了用并发编程思想来思考。想象代码如图28.2发生交叉，假设一开始```flag=0```。

| Thread 1 | Thread 2 |
| -- | -- |
| call lock() |  |
| while (flag == 1) |  |
| interrupt: switch to Thread 2 |  |
|  | call lock() |
|  | while (flag == 1)|
|  | interrupt: switch to Thread 1 |
| flag =1; // set flag to 1(too!) |  |

**Fiture 28.2: Trace: No Mutual Exclusion**

正如你在这个交叉中可以看到的，随着适宜和不合时宜的中断，我们可以简单的产生这种情况，两个线程同时设置flag为1因此两个线程都可以进入临界区。这种行为被专业称作为“bad”，我们明显的没有提供最基本的需要：提供一个互斥的环境。


性能问题，我们将在后面更多的讨论，是基于这样的事实，一个线程等待获取一个已经持有的锁：它会不断地检查标志的值，也即是被称为自旋等待的技术。自旋等待浪费时间等待另一个线程释放锁。单处理器环境该浪费是非常高的，其他等待着甚至不能运行（至少，直到发生上下文切换）！因此，正如我们会更进一步提出的更复杂解决方案，我们也应该考虑如何避免这种浪费。

## 28.7 构建一个可以工作的自旋锁
```
int TestAndSet(int *old_ptr, int new) {
    int old = *old_ptr; // fetch old value at old_ptr
    *old_ptr = new; // store ’new’ into old_ptr
    return old; // return the old value
}
```

```
typedef struct __lock_t {
    int flag;
} lock_t;

void init(lock_t *lock) {
    // 0 indicates that lock is available, 1 that it is held
    lock->flag = 0;
}

void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1)
        ; // spin-wait (do nothing)
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```
**Figure 28.3: A Simple Spin Lock Using Test-and-set**


## 28.8 评价自旋锁

## 28.9 Compare-And-Swap

一些系统提供了另一个硬件原语操作，即compare-and-swap指令（在SPARC上的叫法），或者是compare-and-exchange（在x86上的叫法）。这个单指令的C语言伪代码如图28.4所示。

```
int CompareAndSwap(int *ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected)
        *ptr = new;
    return actual;
}

```
**Figure 28.4: Compare-and-swap**



CAS的基本想法是测试ptr指针指向内存位置上的值是否与expected相等，如果相等则更新这个内存位置上的值为新值new，否则则什么都不做。不论在哪种情况下，它都会返回这个内存位置上的值，它允许调用者通过返回值获知该操作是否成功。

通过CAS指令，我们可以构建一个与test-and-set非常类似的锁。比如，我们可以通过替换上文中的```lock()```如下：

```
void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1)
        ; // spin
}
```

代码的剩余部分与上文中test-and-set的例子一致。两段代码工作基本一致，首先简单检查flag是否为0，如果为0则原子的交换为1也即获得了这个锁。其他试图获取这个已经被持有的锁的线程会陷入自旋等待直到这个锁被最终释放。

如果你想知道真实，x86版本下的C代码CAS，下面代码可能会有用（来自[S05]）：

```
char CompareAndSwap(int *ptr, int old, int new) {
    unsigned char ret;

    // Note that sete sets a ’byte’ not the word
    __asm__ __volatile__ (
        " lock\n"
        " cmpxchgl %2,%1\n"
        " sete %0\n"
        : "=q" (ret), "=m" (*ptr)
        : "r" (new), "m" (*ptr), "a" (old)
        : "memory");
    return ret;
}
```

最后，你可能察觉到，CAS是一个比test-and-set更有力的指令。我们将在之后使用一些这种能力的，当我们简要地深入到wait-free synchronization中[H91]。然而，如果只是需要使用它构建一个简单的自旋锁，它的行为与我们上面所分析的自旋锁一致。

## 28.10 Load-Linked and Store Conditional

```
int LoadLinked(int *ptr) {
    return *ptr;
}

int StoreConditional(int *ptr, int value) {
    if (no one else has updated *ptr since the LoadLinked to this address) {
        *ptr = value;
        return 1; // success!
    } else {
        return 0; // failed to update
    }
}
```
**Figure 28.5: Load-linked And Store-conditional**


```
void lock(lock_t *lock) {
    while (1) {
        while (LoadLinked(&lock->flag) == 1)
            ; // spin until it’s zero
        if (StoreConditional(&lock->flag, 1) == 1)
            return; // if set-it-to-1 was a success: all done
        // otherwise: try it all over again
    }
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```
**Figure 28.6: Using LL/SC To Build A Lock**


## 28.11 Fetch-And-Add

最后一个硬件原语还是fetch-and-add指令，它在返回旧值的时候自动增加其地址上的值，C语言的伪代码如下：

```
int FetchAndAdd(int *ptr) {
    int old = *ptr;
    *ptr = old + 1;
    return old;
}
```


```
typedef struct __lock_t {
    int ticket;
    int turn;
} lock_t; 

void lock_init(lock_t *lock) {
    lock->ticket = 0;
    lock->turn = 0; 
}

void lock(lock_t *lock) {
    int myturn = FetchAndAdd(&lock->ticket);
    while (lock->turn != myturn)
        ; // spin
}

void unlock(lock_t *lock) {
    FetchAndAdd(&lock->turn);
}
```
**Figure 28.7: Ticket Locks**

在上面例子中，我们使用fetch-and-add构建了一个更有意思的**入场券锁**，它由Mellor-Crummey和Sott引入[MS91]。加锁与释放锁的代码如图28.7所示。

该解决方案使用入场券而不是一个值来量组合变构建一个锁。基本操作是非常简单的：当一个线程希望获取锁，它首先原子的对入场券的值进行fetch-and-add；该值目前被认为是该线程的“次序”（```myturn```）。然后全局共享的```lock->turn```然后被用于确定该线程的次序；当指定线程的（```myturn==turn```），则轮到指定线程进入临界区。释放锁则通过简单地增加turn，使得下一个等待的线程（如果有）现在可以进入临界区。



## 28.12 太多自旋：What Now?



## 28.13 一个简单办法：Just Yield,  Baby

```
void init() {
    flag = 0;
}

void lock() {
    while (TestAndSet(&flag, 1) == 1)
        yield(); // give up the CPU
}

void unlock() {
    flag = 0;
}
```
**Figure 28.8: Lock With Test-and-set And Yield**

## 28.14 使用队列：用睡眠取代自旋

与我们以前的办法相比，真正的问题是他们留下太多太多时间去尝试。调度确定哪个线程下一个运行; 如果调度作出了一个不好的选择，一个线程运行必须要么自旋等待锁（我们的第一种方法），或立即放弃CPU（我们的第二个方法）。无论使用哪种方式，都有潜在的资源浪费并且无法预防饥饿。

因此，我们必须明确地施加一些控制谁可以获取到锁，当目前持有者释放后。要做到这一点，我们需要多一点操作系统的支持，以及一个队列来跟踪哪些线程正在等待进入锁。

为简单起见，我们将使用的Solaris提供的两个调用：```park()```将调用线程睡眠，```unpark(threadID)```唤醒通过```threadID```所指定的特定线程。这两个调用可串联用于构建一个锁，这个锁可以保证，如果线程试图获得一个被其他线程持久的锁并唤醒它，它将会进入睡眠状态，当锁是free时将会被线程。让我们看28.9来理解这些原语的可能使用。


```
typedef struct __lock_t {
int flag;
int guard;
queue_t *q;
} lock_t;
 
void lock_init(lock_t *m) {
    m->flag = 0;
    m->guard = 0;
    queue_init(m->q);
}

void lock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ; //acquire guard lock by spinning
    if (m->flag == 0) {
        m->flag = 1; // lock is acquired
        m->guard = 0;
    } else {
        queue_add(m->q, gettid());
        m->guard = 0;
        park();
    }
}

void unlock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ; //acquire guard lock by spinning
    if (queue_empty(m->q))
        m->flag = 0; // let go of lock; no one wants it
    else
        unpark(queue_remove(m->q)); // hold lock (for next thread!)
    m->guard = 0;
}
```
**Figure 28.9: Lock With Queues, Test-and-set, Yield, And Wakeup**


这个例子中我们做了一些有趣的事情在。首先，我们结合之前的test-and-set的想法与一个明确的锁定等待者的队列中来提供一个更有效率的锁。其次，我们用一个队列，以帮助控制谁将得到接下来的锁，从而避免饥饿。

你可能会注意到guard是如何使用（图28.9），基本上就是一个围绕着flag的自旋锁。这种做法，这并不完全避免自旋等待;一个线程在获取锁或者释放锁中可能会被中断，因此导致了其他线程字段等待这个指令再次执行。然而消耗在资源等待上的时间非常有限（在lock与unlock代码中仅仅只有几条指令，而不是用于定义临界区），因此这个方法可能是合理的。

第二，你可能会在```lock()```中注意到，当一个线程不能获得时锁（它已经被持有），我们会小心的把该线程自身添加到队列中（通过调用```gettid()```调用来获得当前的线程ID），设置guard为0，并放弃CPU。读者可能会问：如果释放guard锁在```park()```之后而不是之前，会发生什么？提示：一些不好的事情。

您可能还注意到一个有趣的事实是，当一个线程从队列中被唤醒时，flag不会被设置回0。 为什么是这样？嗯它不是一个错误，而是一种必然！当一个线程被唤醒时，它刚从```park()```返回；然而，它在代码点时不持有guard，因此不能将该标志设置为1，连尝试的机会没有。因此，我们只是直接从线程释放锁到下一个请求它的线程；标志在中间时未设置为0。

最后，你可能会在解决方案中发现的被感知竞争条件，就在调用```park()```之前。在一个错的时间点时，一个线程将即将park，假定它应该睡眠直到锁不再被持有。在这个时候切换到另一个线程（比如一个线程持有着锁）可能导致麻烦，例如，如果该线程之后就释放锁，被第一个线程park的线程将永远睡眠（可能）。这个问题有时也被称为唤醒/等待竞争；为了避免它，我们需要做一些额外的工作。

为了避免它，Solaris通过增加第三个系统调用```setpark()```来解决这个问题。通过调用它，一个线程可以提示它*即将*park。如果它正好被中断，而另一个线程中调用在真正调用park前调用unpark，随后的park马上返回而不是睡眠。修改的代码量很小（在```lock()```内）：

```
1 queue_add(m->q, getid());
2 setpark(); // new code
3 m->guard = 0;
```

## 28.15 不同的操作系统，不同的支持
```
void mutex_lock (int *mutex) {
    int v;
    /* Bit 31 was clear, we got the mutex (this is the fastpath) */
    if (atomic_bit_test_set (mutex, 31) == 0)
        return;
    atomic_increment (mutex);
    while (1) {
        if (atomic_bit_test_set (mutex, 31) == 0) {
            atomic_decrement (mutex);
            return;
        }
        /* We have to wait now. First make sure the futex value
        we are monitoring is truly negative (i.e. locked). */
        v = *mutex;
        if (v >= 0)
            continue;
        futex_wait (mutex, v);
    }
}

void mutex_unlock (int *mutex) {
    /* Adding 0x80000000 to the counter results in 0 if and only if
    there are not other interested threads */
    if (atomic_add_zero (mutex, 0x80000000))
        return;
        
    /* There are other threads waiting for this mutex,
     wake one of them up. */
    futex_wake (mutex);
}
```
**Figure 28.10: Linux - based Futex Locks**
## 28.16 两阶段锁

## 28.17 总结
