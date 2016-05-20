# 31 信号量

## 31.1 信号量：定义

```
#include <semaphore.h>
sem_t s;
sem_init(&s, 0, 1);
```
**Figure 31.1: Initializing A Semaphore**

```
int sem_wait(sem_t *s) {
    decrement the value of semaphore s by one
    wait if value of semaphore s is negative
}

int sem_post(sem_t *s) {
    increment the value of semaphore s by one
    if there are one or more threads waiting, wake one
}
```
**Figure 31.2: Semaphore: Definitions Of Wait And Post**

```
sem_t m;
sem_init(&m, 0, X); // initialize semaphore to X; what should X be?

sem_wait(&m);
// critical section here
sem_post(&m);
```
**Figure 31.3: A Binary Semaphore (That Is, A Lock**

## 31.2 二值信号量（锁）

## 31.3 信号量作为条件变量


```
sem_t s;

void *
child(void *arg) {
    printf("child\n");
    sem_post(&s); // signal here: child is done
    return NULL;
}

int
main(int argc, char *argv[]) {
    sem_init(&s, 0, X); // what should X be?
    printf("parent: begin\n");
    pthread_t c;
    Pthread_create(c, NULL, child, NULL);
    sem_wait(&s); // wait here for child
    printf("parent: end\n");
    return 0;
}
```
**Figure 31.6: A Parent Waiting For Its Child**

## 31.4 生产者消费者（有界缓存）问题
```
int buffer[MAX];
int fill = 0;
int use = 0;

void put(int value) {
    buffer[fill] = value; // line f1
    fill = (fill + 1) % MAX; // line f2
}

int get() {
    int tmp = buffer[use]; // line g1
    use = (use + 1) % MAX; // line g2
    return tmp;
}
```
**Figure 31.9: The Put And Get Routines**

```
sem_t empty;
sem_t full;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty); // line P1
        put(i); // line P2
        sem_post(&full); // line P3
    }
}

void *consumer(void *arg) {
    int i, tmp = 0;
    while (tmp != -1) {
        sem_wait(&full); // line C1
        tmp = get(); // line C2
        sem_post(&empty); // line C3
        printf("%d\n", tmp);
    }
}

int main(int argc, char *argv[]) {
    // ...
    sem_init(&empty, 0, MAX); // MAX buffers are empty to begi
    sem_init(&full, 0, 0); // ... and 0 are full
    // ...
}
```
**Figure 31.10: Adding The Full And Empty Conditions**

```
sem_t empty;
sem_t full;
sem_t mutex;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&mutex); // line p0 (NEW LINE)
        sem_wait(&empty); // line p1
        put(i); // line p2
        sem_post(&full); // line p3
        sem_post(&mutex); // line p4 (NEW LINE)
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&mutex); // line c0 (NEW LINE)
        sem_wait(&full); // line c1
        int tmp = get(); // line c2
        sem_post(&empty); // line c3
        sem_post(&mutex); // line c4 (NEW LINE)
        printf("%d\n", tmp);
    }
}

int main(int argc, char *argv[]) {
    // ...
    sem_init(&empty, 0, MAX); // MAX buffers are empty to begin with...
    sem_init(&full, 0, 0); // ... and 0 are full
    sem_init(&mutex, 0, 1); // mutex=1 because it is a lock (NEW LINE)
    // ...
}
```
**Figure 31.11: Adding Mutual Exclusion (Incorrectly)**


```
sem_t empty;
sem_t full;
sem_t mutex;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty); // line p1
        sem_wait(&mutex); // line p1.5 (MOVED MUTEX HERE...)
        put(i); // line p2
        sem_post(&mutex); // line p2.5 (... AND HERE)
        sem_post(&full); // line p3
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&full); // line c1
        sem_wait(&mutex); // line c1.5 (MOVED MUTEX HERE...)
        int tmp = get(); // line c2
        sem_post(&mutex); // line c2.5 (... AND HERE)
        sem_post(&empty); // line c3
        printf("%d\n", tmp);
    }
}

int main(int argc, char *argv[]) {
    // ...
    sem_init(&empty, 0, MAX); // MAX buffers are empty to begin with...
    sem_init(&full, 0, 0); // ... and 0 are full
    sem_init(&mutex, 0, 1); // mutex=1 because it is a lock
    // ...
}
```
**Figure 31.12: Adding Mutual Exclusion (Correctly)**

```
typedef struct _rwlock_t {
    sem_t lock; // binary semaphore (basic lock)
    sem_t writelock; // used to allow ONE writer or MANY readers
    int readers; // count of readers reading in critical section
} rwlock_t;

void rwlock_init(rwlock_t *rw) {
    rw->readers = 0;
    sem_init(&rw->lock, 0, 1);
    sem_init(&rw->writelock, 0, 1);
}

void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers++;
    if (rw->readers == 1)
        sem_wait(&rw->writelock); // first reader acquires writelock
    sem_post(&rw->lock);
}

void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers--;
    if (rw->readers == 0)
        sem_post(&rw->writelock); // last reader releases writelock
    sem_post(&rw->lock);
}

void rwlock_acquire_writelock(rwlock_t *rw) {
    sem_wait(&rw->writelock);
}

void rwlock_release_writelock(rwlock_t *rw) {
    sem_post(&rw->writelock);
}
```
**Figure 31.13: A Simple Reader-Writer Lock**

## 31.5 Reader-Writer Locks

## 31.6 The Dining Philosophers

```
void getforks() {
    sem_wait(forks[left(p)]);
    sem_wait(forks[right(p)]);
}

void putforks() {
    sem_post(forks[left(p)]);
    sem_post(forks[right(p)]);
}
```
**Figure 31.15: The getforks() And putforks() Routines**

```
typedef struct __Zem_t {
    int value;
    pthread_cond_t cond;
    pthread_mutex_t lock;
} Zem_t;

// only one thread can call this
void Zem_init(Zem_t *s, int value) {
    s->value = value;
    Cond_init(&s->cond);
    Mutex_init(&s->lock);
}

void Zem_wait(Zem_t *s) {
    Mutex_lock(&s->lock);
    while (s->value <= 0)
        Cond_wait(&s->cond, &s->lock);
    s->value--;
    Mutex_unlock(&s->lock);
}

void Zem_post(Zem_t *s) {
    Mutex_lock(&s->lock);
    s->value++;
    Cond_signal(&s->cond);
    Mutex_unlock(&s->lock);
}
```
**Figure 31.16: Implementing Zemaphores With Locks And CVs**

## 31.7 How To Implement Semaphores

## 31.8 Summary

