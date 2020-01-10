title: 并发编程-AQS详解
author: v-vampires
date: 2020-01-07 23:14:36
tags:
---
## 介绍
AQS是AbstractQueuedSynchronizer的简称，为那些依赖一个共享资源（volatile类型的原子变量state）的自定义同步器（ReentrantLock/Semaphore/CountDownLatch）提供了一个基础。自定义同步器在是实现时只需要实现共享资源的获取与释放，至于线程等待队列的维护（如获取资源失败入队/资源释放唤醒线程出队），AQS已经给实现好了。
## 资源共享方式
AQS定义了两种资源共享的方式：Exclusive(独占，只有一个线程能获取资源并执行，如ReentrantLock)、Share（共享，多个线程可同时获取到资源并执行，如Semaphore/CountDownLatch）。  
不同的自定义同步器使用共享资源的方式也不同。
## 源码实现
AQS内部维护了一个队列（双向链表），也就是锁上面有一个队列，当线程获取锁失败后，会被添加到队列的尾部，只有头节点才有锁的获取权。在AQS内部最重要的代码就是，当线程获取锁失败，然后将当前线程封装成Node，添加到队列的尾部，然后挂起线程（注意：挂起就是使线程进入非可执行的状态，在这个状态下，CPU不会分给线程时间片）。代码如下
### acquireQueued方法
```
//此时，node已经被加入到队列的尾部
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //1. 获取前置节点
            final Node p = node.predecessor();
            //2. 如果前置是头节点，并且当前节点获取到锁，则设置当前节点为head，原来的头节点出队列（p.next=null）
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //3. 获取锁失败，如果应该挂起线程，则挂起线程,注意如果该方法短路，则会进行自旋，直到第二步返回，或者线程挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
### shouldParkAfterFailedAcquire方法
```
//pred是node的前驱节点
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * 如果前驱结点完成任务后能够唤醒自己，那么当前节点就可以安心挂起了
         */
        return true;
    if (ws > 0) {//表示前驱节点被取消或者中断了了
        do {
            node.prev = pred = pred.prev;//删掉链表中的取消或者中断的节点
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         * 找到了合适的前驱节点，将其状态设置为SINGAL，表示前驱节点可以唤醒自己，再次循环进来时，就可以安心挂起了
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
当线程第一次执行到该方法时，pred是原来的tail节点，此时pred的waitStatus为0或者1，线程需要修改waistatus的值或者清除失效节点，返回false，因此parkAndCheckInterrupt()则会被短路，线程不会被挂起，进行第二次循环，而如果此时当前线程节点的前置节点为head节点，线程再一次尝试去获取一次资源，如果获取失败，则第二次进入shouldParkAfterFailedAcquire，此时pred的waitStatus为Node.SIGNAL，则当前线程进行挂起。

因次可以看到，AQS通过shouldParkAfterFailedAcquire(p, node)方法保证在挂起线程之前至少1次的"自旋"并在"自旋"前后尝试去获取资源。只所以使用自旋+挂起的方式，是因为挂起和唤醒一个线程将会带来上下文切换的开销，但是如果长时间的自旋也会浪费CPU。

### release方法
```
    public final boolean release(int arg) {
        //释放资源
        if (tryRelease(arg)) {
            Node h = head;
            //唤醒后继节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
### unparkSuccessor方法
```
    //唤醒此节点的后继节点
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * 如果要唤醒的后继节点为空，或者被取消，应踢出队列，再从后向前找到最靠前的合法的节点
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
当后继节点被唤醒后，则继续执行acquireQueued方法，再次循环并获取资源，如果获取成功则设置该节点为头节点，让原来的头节点出队列；如果获取不成功（刚巧，有一个其他线程进来获取到了资源，这也是非公平锁产生的主要由来）则继续挂起