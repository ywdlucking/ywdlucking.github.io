---
layout:     post
title:      "并发之ReentrantLock"
subtitle:   " \"JAVA并发包浏览\""
date:       2019-05-31 12:00:00
author:     "Dong"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 并发
---

## 描叙
ReentrantLock是Java并发包中提供的一个可重入的互斥锁。ReentrantLock和synchronized在基本用法，行为语义上都是类似的，同样都具有可重入性。只不过相比原生的Synchronized，ReentrantLock增加了一些高级的扩展功能，比如它可以实现公平锁，同时也可以绑定多个Conditon。

## 可重入性/公平锁/非公平锁
- 可重入性

　　所谓的可重入性，就是可以支持一个线程对锁的重复获取，原生的synchronized就具有可重入性，一个用synchronized修饰的递归方法，当线程在执行期间，它是可以反复获取到锁的，而不会出现自己把自己锁死的情况。ReentrantLock也是如此，在调用lock()方法时，已经获取到锁的线程，能够再次调用lock()方法获取锁而不被阻塞。那么有可重入锁，就有不可重入锁，我们在之前文章中自定义的一个Mutex锁就是个不可重入锁，不过使用场景极少而已。

- 公平锁/非公平锁

　　所谓公平锁,顾名思义，意指锁的获取策略相对公平，当多个线程在获取同一个锁时，必须按照锁的申请时间来依次获得锁，排排队，不能插队；非公平锁则不同，当锁被释放时，等待中的线程均有机会获得锁。synchronized是非公平锁，ReentrantLock默认也是非公平的，但是可以通过带boolean参数的构造方法指定使用公平锁，但非公平锁的性能一般要优于公平锁。

synchronized是Java原生的互斥同步锁，使用方便，对于synchronized修饰的方法或同步块，无需再显式释放锁。synchronized底层是通过monitorenter和monitorexit两个字节码指令来实现加锁解锁操作的。而ReentrantLock做为API层面的互斥锁，需要显式地去加锁解锁。　



## 代码分析
ReentrantLock是基于AQS的，AQS是Java并发包中众多同步组件的构建基础，它通过一个int类型的状态变量state和一个FIFO队列来完成共享资源的获取，线程的排队等待等。AQS是个底层框架，采用模板方法模式，它定义了通用的较为复杂的逻辑骨架，比如线程的排队，阻塞，唤醒等，将这些复杂但实质通用的部分抽取出来，这些都是需要构建同步组件的使用者无需关心的，使用者仅需重写一些简单的指定的方法即可（其实就是对于共享变量state的一些简单的获取释放的操作）。
- 无参构造器（默认为非公平锁）
```java
public ReentrantLock() {
    sync = new NonfairSync();//默认是非公平的
}
```
- 带布尔值的构造器（是否公平）
```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();//fair为true，公平锁；反之，非公平锁
 }
```
**接下来，关于如何实现重入性，如何实现公平性，就得去看这几个静态内部类了**
- NonFairSync（非公平可重入锁）
```java

final void lock() {
            if (compareAndSetState(0, 1))////CAS设置state状态，若原值是0，将其置为1
                setExclusiveOwnerThread(Thread.currentThread());//将当前线程标记为已持有锁
            else
                acquire(1);//若设置失败，调用AQS的acquire方法，acquire又会调用我们下面重写的tryAcquire方法
        }

protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);//调用Sync中的方法
        }
```
**流程**
1. 先获取state值，若为0，意味着此时没有线程获取到资源，CAS将其设置为1，设置成功则代表获取到排他锁了；

2. 若state大于0，肯定有线程已经抢占到资源了，此时再去判断是否就是自己抢占的，是的话，state累加，返回true，重入成功，state的值即是线程重入的次数；

3. 其他情况，则获取锁失败。


- FairSync

```java
final void lock() {
            acquire(1);
        }

protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {///判断在时间顺序上，是否有申请锁排在自己之前的线程，若没有，才能去获取
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
    }

public final boolean hasQueuedPredecessors() {
        Node t = tail; // 尾结点
        Node h = head;//头结点
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());//判断是否有排在自己之前的线程
    }
```

## 总结
ReentrantLock是一种可重入的，可实现公平性的互斥锁，它的设计基于AQS框架，可重入和公平性的实现逻辑都不难理解，每重入一次，state就加1，当然在释放的时候，也得一层一层释放。至于公平性，在尝试获取锁的时候多了一个判断：是否有比自己申请早的线程在同步队列中等待，若有，去等待；若没有，才允许去抢占