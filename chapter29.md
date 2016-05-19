# 29 基于锁的并发数据结构

在介绍锁之外前，我们将首先介绍如何使用锁来实现常见的数据结构。通过向数据结构中添加锁，来保证**线程安全**。当然，添加锁究竟如何同时保证正确性和性能是我们面临的挑战：

当然，我们将很难向所有的数据结构或方法添加并发性，因为这是已经研究多年的主题，有关它的数以千计的研究论文被发表。因此，我们希望能够提供足够的介绍需要的思考类型，并提供一些好的资料供你作进一步的学习。我们发现Moir和Shavit的调查是一个非常好的信息[MS04]来源。

## 29.1 并发计数器

一个最简单的数据结构是一个计数器。它是一种具有简单的接口常用的结构。我们定义简单的非同步计数器如图29.1。

```
typedef struct __counter_t {
    int value;
} counter_t;

void init(counter_t *c) {
    c->value = 0;
}

void increment(counter_t *c) {
    c->value++;
}

void decrement(counter_t *c) {
    c->value--;
}

int get(counter_t *c) {
    return c->value;
}
```
**Figure 29.1: A Counter Without Locks**

### 简单但不可伸缩

正如你所看到的，非同步计数器是一个微不足道的数据结构，只需要少量代码实现。现在，我们有我们的下一个挑战：我们如何才能使此代码**线程安全**的？图29.2所示我们是如何做到这一点。

```
typedef struct __counter_t {
    int value;
    pthread_mutex_t lock;
} counter_t;

void init(counter_t *c) {
    c->value = 0;
    Pthread_mutex_init(&c->lock, NULL);
}

void increment(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value++;
    thread_mutex_unlock(&c->lock);
}

void decrement(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value--;
    Pthread_mutex_unlock(&c->lock);
}

int get(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    int rc = c->value;
    Pthread_mutex_unlock(&c->lock);
    return rc;
}
```
**Figure 29.2: A Counter With Locks**

这同步计数器工作简单、正常。事实上，它遵循最简单和最基本并发数据结构共同的设计模式：它只是增加了一个锁，这是调用该操作的数据结构的函数时获得的，并且在调用返回时被释放。通过这种方式，它是一个与**监视器**类似的数据结构[BH73]，其中锁随着调用函数和函数返回被自动获取释放收购。


## 29.2 并发链表

## 29.3 并发队列

## 29.4 并发哈希表

## 29.5 总结

