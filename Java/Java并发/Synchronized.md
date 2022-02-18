#### Synchronized 简述

Synchronized 关键字解决的是多个线程之间访问资源的同步性，Synchronized 关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。

锁主要存在四种状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态，他们会随着竞争的激烈而逐渐升级。注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

#### Synchronized 使用方法

* 修饰实例方法：对当前实例对象加锁，进入同步代码前要获得当前对象实例的锁。
* 修饰静态方法：对当前类的类对象加锁，进入同步代码前要获得当前类对象的锁。
* 修饰代码块：对 Synchronized 括号内的对象加锁。

#### Synchronized 底层原理

<img src="/Users/licheng/Documents/Typora/Picture/image-20200422165101931.png" alt="image-20200422165101931" style="zoom:50%;" />

Synchronized 是基于进入和退出 Monitor 对象来实现方法同步和代码块同步的。

同步方法是通过设置 ACC_SYNCHRONIZED 标志来实现，当线程执行有 ACC_SYNCHRONI 标志的方法，需要获得 Monitor 锁。

代码块的同步是通过 `monitorenter` 和 `monitorexit` 实现的。

当一个线程执行到 `monitorenter` 指令时，会尝试获取对象对应的 Monitor 的所有权。如果计数器为 0，表示可以获取，将计数器 + 1。获取到所有权后会将对象的 Mark Word 会被替换为 Monitor 指针。如果计数器不为 0，当前线程会进入阻塞状态。执行到 `monitorexit` 指令时，会将对象的 Mark Word 重置，并将计数器减 1，如果计数器为 0 ，表明锁被释放。然后会唤醒 EntryList 中等待的线程来竞争锁。

#### 等待/通知机制实现原理

每个锁对象都有两个队列，一个是等待队列，一个是阻塞队列。当一个线程不满足运行条件时会调用 `wait()` 方法进入等待队列，等待其他线程调用 `notify()`，它才会进入阻塞队列，等待被 CPU 调度。

使用方法：`wait() `、`wait(long timeout)`、` notify()`和 `notifyAll()`。

#### Synchronized 锁升级过程

在 Java 早期版本中，Synchronized 属于重量级锁，因为 Synchronized 锁的实现依赖于 Monitor，而 Monitor 是依赖于底层的操作系统来实现的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，时间成本相对较高。

 JDK 1.6 对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。

<img src="/Users/licheng/Documents/Typora/Picture/image-20200422165353103.png" alt="image-20200422165353103" style="zoom:67%;" />

* 首先判断是不是无锁状态，如果是无锁状态，会偏向第一次访问它的线程，线程利用 CAS 将对象的 hashCode 替换为当前线程 id，如果成功则表示当前线程获得偏向锁，置偏向标志位为 1。
* 第二次访问时如果还是可偏向的状态，检测 Mark Word 里面是不是当前线程的 id，如果是，那么可以直接使用。
* 如果不是，则尝试使用 CAS 获取偏向锁，如果成功就将线程 id 替换为当前线程的 id。如果失败，说明发生竞争，撤销偏向锁，升级为轻量级锁。
* 在当前线程的栈帧中创建 Lock Record 对象。让 Lock Record 中的 Object Reference 指向锁对象，并尝试用 CAS 将锁对象的 Mark Word 替换为 Lock Record 对象的地址，并将锁标志位置为 00。如果成功，当前线程获得锁。
* 如果失败，表示存在竞争，当前线程便尝试使用自旋来获取锁。
* 如果自旋成功则依然处于轻量级锁状态。
* 如果自旋失败，则升级为重量级锁。

#### Synchronized 和 ReenTrantLock 的对比

* Synchronized 由 JVM 实现，而 ReentrantLock 由 JDK 实现.
* ReentrantLock 可以指定是公平锁还是非公平锁。而 Synchronized 只能是非公平锁。(公平锁就是先等待的线程先获得锁)
* ReentrantLock 线程对象可以注册在指定的 Condition 中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。
* ReentrantLock 提供了一种能够中断等待锁的线程的机制，正在等待的线程可以选择放弃等待，改为处理其他事情。
* 两者都是可重入锁（自己可以再次获取自己的内部锁）

#### Synchronized 关键字和 volatile 关键字对比

* volatile 关键字只能用于变量，而 Synchronized 关键字可以修饰变量、方法以及代码块。
* volatile 关键字能保证数据的可见性，但不能保证数据的原子性。Synchronized关键字两者都能保证。
* volatile 关键字不会造成线程的阻塞，而 Synchronized 关键字可能会造成线程的阻塞。

> volatile 本质是在告诉 jvm 当前变量在寄存器中的值是不确定的，需要从主存中读取；而 Synchronized 是锁住当前变量，只有当前线程可以访问该变量。
