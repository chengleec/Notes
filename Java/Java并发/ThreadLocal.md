### 简述

ThreadLocal 提供了线程的局部变量，这样就不会和其他线程的局部变量发生冲突，实现了线程的数据隔离，避免了线程安全问题。

### 原理

每个**线程**中都维护了 ThreadLocalMap 这么一个 Map，key 是 ThreadLocal 对象本身，value 则是要存储的对象。Rehash 时使用的是开放地址法。

可以创建多个 TheadLocal 对象来存储多个值。

```java
ThreadLocal<Integer> tl1 = new ThreadLocal<>();
ThreadLocal<Integer> tl2 = new ThreadLocal<>();
```

### 内存泄漏

内存泄漏指未能释放不再使用的内存。

#### 原因

ThreadLocalMap 使用 ThreadLocal 的弱引用作为 key，而 value 是强引用。如果一个 ThreadLocal 没有外部强引用来引用它，那么系统 GC 的时候，这个 ThreadLocal 势必会被回收，这样一来，ThreadLocalMap 中就会出现 key 为 null 的 Entry，就没有办法访问这些 key 为 null 的 Entry 的 value，如果当前线程再迟迟不结束的话 (比如线程池)，这些 key 为 null 的 Entry 的 value 就会一直存在一条强引用链：`Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄漏。

其实，ThreadLocalMap 的设计中已经考虑到这种情况，也加上了一些防护措施：在 ThreadLocal 的`get()`,`set()`,`remove()`的时候都会清除线程 ThreadLocalMap 里所有 key 为 null 的 value。

但是这些被动的预防措施并不能保证不会内存泄漏：

* 使用 static 的 ThreadLocal，延长了 ThreadLocal 的生命周期，可能导致的内存泄漏。
* 分配使用了 ThreadLocal 又不再调用`get()`,`set()`,`remove()`方法，那么就会导致内存泄漏。

#### 解决方法

每次使用完 ThreadLocal，都调用 `remove()` 方法清除数据。

### 方法

#### set() 方法

```java
public void set(T value) {   
    //获取当前的线程对象
	Thread t = Thread.currentThread();
    //获取该线程对象的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //如果map不为空，执行set操作，以当前ThreadLocal对象为key，存储对象为value
    if (map != null) map.set(this, value);
    //如果map为空，则为该线程创建ThreadLocalMap
    else createMap(t, value);    
}
```

#### get() 方法

```java
public T get(){
    Threadt = Thread.currentThread();
    ThreadLocal.ThreadLocalMapmap = this.getMap(t);
    if (map != null) {
        ThreadLocal.ThreadLocalMap.Entry e=map.getEntry(this);
        if(e!=null){
            T result=e.value;
            return result;
        }
    }
    return this.setInitialValue();
}
```

 #### remove() 方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null) {
        m.remove(this);
    }
}
```



 