> 转载：[并发编程 —— 源码分析公平锁和非公平锁](https://juejin.im/post/6844903600422912007)

## 1. 前言

ReentrantLock 提供了公平锁和非公平锁，只需要在构造方法中使用一个 `boolean` 参数即可。默认非公平锁。

今天从源码层面看看区别和具体实现。

## 2. 类 UML 图

![2020-08-21-s3r3a3](https://image.ldbmcs.com/2020-08-21-s3r3a3.jpg)

`ReentrantLock` 内部有一个抽象类 `Sync`，继承了 AQS。

而公平锁的实现就是 `FairSync`，非公平锁的实现就是 `NodFairSync`。

两把锁的区别在于`lock` 方法的实现。

## 3. 公平锁 lock 方法实现

```java
final void lock() {
    acquire(1);
}
```

调用的是 AQS 的`acquire`方法，熟悉 AQS 的同学都知道，AQS 会回调子类的 `tryAcquire` 方法，看看公平锁的`tryAcquire`实现。

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

说下逻辑：

1. 获取 state 变量，如果是 0，说明锁可以获取。
2. **判断 AQS 队列中是否有等待的线程**，如果没有，就使用 CAS 尝试获取。获取成功后，将 CLH 的持有线程修改为当前线程。
3. 重入锁逻辑。
4. 如果失败，返回 false， AQS 会将这个线程放进队列，并挂起。

注意上面的第二步：**判断 AQS 队列中是否有等待的线程**。

> 这就是公平的体现。

再看看非公平锁的区别。

## 4. 非公平锁 lock 方法实现

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

lock 方法就不同了，很冲动的一个方法，直接使用 CAS 获取锁，如果成功了，就设置锁的持有线程为自己。很快速。所以默认使用了非公平锁。

如果失败了，就调用 AQS 的 `acquire` 方法。当然，我们看的还是`tryAcquire`方法，在上面的代码中，`tryAcquire`方法调用了父类`Sync` 的 `nonfairTryAcquire`，为什么在父类中呢？

在`ReentrantLock` 的 `tryLock`方法中，也调用了该方法。因为这个方法是快速返回的。该方法不会让等待时间久的线程获取锁。符合 `tryLock` 的设计。

方法实现如下：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法相比较公平锁的 `tryAcquire`方法，少了一步**判断 AQS 队列中是否有等待的线程**的操作。

他要做的就是直接抢锁，不让给队列里那些等待时间长的。

抢不到再进入队列。等待他的前置节点唤醒他。这个过程是公平的。

## 5. 总结

`ReentrantLock`中的公平锁和非公平锁的区别就在于：调用 `lock` 方法获取锁的时候 `要不要判断 AQS 队列中是否有等待的线程`，公平锁为了让每一个线程都均衡的使用锁，就需要判断，如果有，让给他，非公平锁很霸道，不让不让就不让。

但如果失败了，进入队列了，进会按照 AQS 的逻辑来，整体顺序就是公平的。

还有个注意的地方就是：`ReentrantLock` 的 `tryLock（无超时机制）` 方法使用的非公平策略。符合他的设计。

而 `tryLock(long timeout, TimeUnit unit)` 方法则会根据 Sync 的具体实现来调用。不会冲动的调用 `nonfairTryAcquire` 方法。