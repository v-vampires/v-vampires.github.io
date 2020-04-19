title: 并发编程-CompletableFuture的使用
author: v-vampires
categories:
  - 并发编程
date: 2020-01-16 20:29:58
tags:
---
## 介绍
jdk1.8提供了CompletableFuture来支持异步编程，它是一个功能强大的异步编程工具类。CompletableFuture主要实现了Future和CompletionStage接口，Future接口主要是用来实现获取异步任务的执行结果和取消任务等操作。而CompletionStage接口则主要用来描述任务的时序关系的，如果我们把一项大的工作分成几个小的任务，那这几个小的任务之间是有一定的时序关系的，例如串行关系，并行关系，汇聚关系等。如下图例子所示

![upload successful](/images/pasted-10.png)

## 执行模式
CompletableFuture的大部分方法都有三种执行模式
```
//由当前线程，或者调用线程执行
xxx(fn)
//使用默认的forkJoinPool执行
xxxAsync(fn)
//使用指定的线程池执行
xxxAsync(fn,executor)
```
## 创建任务
```
CompletableFuture<Void> runAsync(Runnable runnable)
CompletableFuture<U> supplyAsync(Supplier<U> supplier)
```
这两个接口的主要区别是Runnable接口没有返回值，Supplier接口有返回值
## 任务关系
### 串行关系
```
CompletableFuture<U> thenApply(fn)
CompletableFuture<Void> thenAccept(consumer)
CompletableFuture<Void> thenRun(runnable)
```
fn是有入参，也有返回值  
consumer是只有入参，没有返回值  
runnable是既没有入参，也没有返回值  
示例代码
```
public static void serial() {
    CompletableFuture<String> f =
            CompletableFuture.supplyAsync(() -> "hello")//1
                    .thenApply(s -> s + " world!")//2
                    .thenApply(String::toUpperCase);//3
    System.out.println(f.join());
}
```
首先是通过supplyAsync开启一个异步线程执行任务，再执行两个串行操作。这三步是串行执行的，先执行1，再执行2，再执行3
### 并行关系
```
/**
 * 并行执行
 */
public static void parallel() {
    CompletableFuture<Void> f1 = CompletableFuture.runAsync(() -> {
        System.out.println("T1:do something start");
        sleep(10, TimeUnit.SECONDS);
        System.out.println("T1:do something end");
    });
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        System.out.println("T2:do something start");
        System.out.println("T2:do something end");
        return "test";
    });
    //等待f1，f2都执行完毕
    CompletableFuture.allOf(f1, f2).join();
}
```
此示例是f1和f2是并行执行
### AND汇聚关系
```
CompletableFuture<V> thenCombine(other, fn)
CompletableFuture<Void> thenAcceptBoth(other,consumer)
CompletableFuture<Void> runAfterBoth(other, runnable)
```
示例代码
```
public static void test2() {
    CompletableFuture<Void> f1 = CompletableFuture.runAsync(() -> {
        System.out.println(Thread.currentThread().getName() + ":洗水壶");
        sleep(1, TimeUnit.SECONDS);
        System.out.println(Thread.currentThread().getName() + ":烧开水");
        sleep(10, TimeUnit.SECONDS);
    });
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        System.out.println(Thread.currentThread().getName() + ":洗茶壶");
        sleep(1, TimeUnit.SECONDS);
        System.out.println(Thread.currentThread().getName() + ":洗茶杯");
        sleep(2, TimeUnit.SECONDS);
        System.out.println(Thread.currentThread().getName() + ":拿茶叶");
        sleep(1, TimeUnit.SECONDS);
        return "龙井";
    });

    CompletableFuture<String> f3 = f1.thenCombine(f2, (aVoid, tf) -> {
        System.out.println(Thread.currentThread().getName() + ":拿到茶叶:" + tf);
        System.out.println(Thread.currentThread().getName() + ":泡茶...");
        return Thread.currentThread().getName() + ":上茶:" + tf;
    });

    //等待任务3执行结果
    System.out.println(f3.join());
}
```
### or汇聚关系
```
CompletableFuture<U> applyToEither(other, fn)
CompletableFuture<Void> acceptEither(other, consumer)
CompletableFuture<Void> runAfterEither(other, runnable) 
```