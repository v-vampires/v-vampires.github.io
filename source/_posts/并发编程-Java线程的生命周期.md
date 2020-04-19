title: 并发编程-Java线程的生命周期
author: v-vampires
date: 2020-01-11 22:23:32
tags:
categories: 
- 并发编程
---
## 介绍
在Java领域，实现并发程序的主要手段就是多线程。而Java语言里面的线程本质是就是操作系统的线程，他们是一一对应的
## 生命周期
java线程的生命周期分为初始状态（NEW）、可运行/运行状态（RUNNABLE）、休眠状态（BLOCKED、WAITING、TIMED_WAITING）、终止状态（TERMINATED）
## 状态转换
状态转换图如下
![upload successful](/images/pasted-8.png)
### NEW到RUNNABLE
Java 刚创建出来的 Thread 对象就是NEW状态，调用线程对象的 start() 方法。就可以从NEW状态转换到RUNNABLE
### RUNNABLE与BLOCKED的状态转换
只有一种场景会触发这种转换，就是线程等待synchronized锁时，此时线程的状态由RUNNABLE转换到BLOCKED状态。而当等待的线程获得synchronized锁时，则由BLCOKED转换到RUNNABLE状态。  
注意：如果线程调用阻塞式API时，在操作系统层面线程是会转换到休眠状态，但是在JVM层面，java的线程是不会变换的，依然为RUNNABLE状态。JVM层面并不关心操作系统线程的状态，在JVM看来，等待CPU使用权（操作系统层面此时处于可执行状态）与等待I/O（操作系统层面此时是处于休眠状态）没有区别，都是在等待某个资源，所以都归入了RUNNABLE状态
### RUNNABLE与WAITING状态的转换
第一种场景：当线程获得synchronized锁后，调用无参的Object.wait()方法  

第二种场景：调用无参数的 Thread.join() 方法，join方法是一种线程同步方法。例如有一个线程thread A，当调用A.join()后，执行这条语句的线程会等待A线程执行完毕，而等待中的线程，会从RUNNABLE转换到WAITING，当线程A执行完，原来等待它的线程又从WAITING转换到RUNNABLE  

但三种场景：调用 LockSupport.park() 方法，当前线程会阻塞，状态会从 RUNNABLE 转换到 WAITING。调用 LockSupport.unpark(Thread thread) 可唤醒目标线程，状态又会从 WAITING 状态转换到 RUNNABLE

### RUNNABLE与TIMED_WAITING的状态转换
第一种场景：调用带超时参数的 Thread.sleep(long millis) 方法

第二种场景：调用带超时参数的 Object.wait(long timeout) 方法

第三种场景：调用带超时参数的 Thread.join(long millis) 方法

第四种场景：调用带超时参数的 LockSupport.parkNanos(Object blocker, long deadline) 方法

第五种场景：调用带超时参数的 LockSupport.parkUntil(long deadline) 方法
### 从 RUNNABLE 到 TERMINATED 状态
线程执行完 run() 方法后，会自动转换到 TERMINATED 状态，如果执行 run() 方法的时候异常抛出，也会导致线程终止