### 概述

AQS（队列同步器，AbstractQueuedSynchronizer)，是用来构建锁或其他同步组件的核心基础框架。由一个整型变量 state 表示同步状态，一个队列实现等待队列，多个队列实现条件等待队列。

AQS的设计是基于**模板方法**模式的。使用时需要继承 AQS 并重写相应的方法。

```java
protected boolean tryAcquire(int arg) {throw new UnsupportedOperationException();}
protected boolean tryRelease(int arg) {throw new UnsupportedOperationException();}
protected int tryAcquireShared(int arg) {throw new UnsupportedOperationException();}
protected int tryReleaseShared(int arg) {throw new UnsupportedOperationException(); }
protected int isHeldExclusively(int arg) {throw new UnsupportedOperationException();}
```

waitStatus 表示当前线程的等待状态，等待队列中需要设置相应的状态来实现相应的处理逻辑。

```java
//表示线程因为中断或者等待超时，需要从等待队列中取消等待
static final int CANCELLED = 1; 
//表示当前线程有义务唤醒等待队列中后面的线程
static final int SIGNAL = -1; 
//表示当前线程正在条件队列中等待
static final int CONDITION = -2; 
//当前线程处在SHARED情况下，该字段才会使用
static final int PROPAGATE = -3;
```

### 实现

#### Reentrantlock

##### 加锁

* 调用 `tryAcquire(1) ` 方法，用 `getState()` 获取 state 的值，如果 state 的值为 0，那么使用 CAS 将 state 的值设为 1，设置成功则将该线程成为锁的所有者。

* 如果 state 的值不为 0，查看占用锁的线程是不是自己，如果是的话就将 state+1。

* 如果 state 不为 0 且锁的持有者又不是自己，那就将当前线程构造成一个 Node 结点，加入到等待队列中。
* 如果等待队列中前驱节点为头结点，则再次调用`tryAcquire(1)`方法尝试获取锁，如果获取成功，将当前节点设置为同步队列的头结点，当前线程执行。
* 否则将前驱节点的状态设置为 SIGNAL，然后 park 当前线程，等待被前驱节点唤醒。

##### 释放锁

* 首先调用 `tryRelease() `判断此次释放锁后 state 的值是否为 0，如果是 0，将锁的持有者设置成 null；如果不是 0，代表锁发生了锁重入，锁还没有释放完，就将 state - 1。

* 然后检查等待队列，如果 head 不为 null 并且 waitStatus 不为 0，就调用 unparkSuccessor 唤醒其后继节点。

* 等待队列中的节点被唤醒之后获取锁，将自己设为头结点，并删除前驱节点。

#### ReentrantReadWriteLock

当读操作远远高于写操作时，这时候使用 ReentrantReadWriteLock 让读操作可以并发，提高性能。

实现原理是将 state 分成两部分：

`SHARED_UNIT`：读锁状态码，由 state 的高 16 位表示。

`EXCLUSIVE_MASK`：写锁状态码，由 state 的低 16 位表示。

#### StampedLock

该类自 JDK 1.8 加入，是为了进一步优化读性能，它的特点是在使用读锁、写锁时都必须配合戳使用。

```java
//加读锁
long stamp = lock.readLock();
lock.unlockRead(stamp);
//加写锁
long stamp = lock.writeLock();
lock.unlockWrite(stamp);
```

#### Semaphore

自 JDK 1.5 之后加入，用来限制能同时访问共享资源的线程上限。

```java
public class SemaphoreDemo{
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    semaphore.acquire();
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release();
                }
            }).start();
        }
    }
}
```

#### CountdownLatch

自 JDK 1.5 之后加入，利用它可以实现类似计数器的功能。

比如有一个任务 A，它要等待其他 4 个任务执行完毕之后才能执行，此时就可以利用 CountDownLatch 来实现这种功能了。

其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一。

```java
public class CountdownLatchDemo{
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(3);
        for (int i = 0; i < 3; i++) {
            int num = i;
            new Thread(()->{
                try {
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    countDownLatch.countDown();
                }
            }).start();
        }
        countDownLatch.await();
    }
}
```



