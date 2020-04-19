title: 并发编程-Fork/Join框架
author: v-vampires
date: 2020-02-04 17:19:07
tags:
---
## 背景
并发编程时，我们为了处理一个大任务，通常会将一个大任务分成几个子任务然后分别去处理。还有一种情况是，将一个大任务分解成多个相似的子任务,再将子任务分解成相似的子子任务，直到子任务可以直接计算。这种思想就是分治思想，常见的应用场景如归并排序、快速排序、计算框架MapReduce等。这种任务模型就是我们常说的分治任务模型。
## 分治任务模型
分治任务模型分为两个阶段，一个阶段是任务分解，就是将大任务分解成子任务，再讲子任务分解成子子任务，直到任务可以直接计算；另一个阶段是计算结果合并，逐层合并子任务的结果，直至获取到最终的结果，下图是一个简化的分治任务模型图

![upload successful](/images/pasted-11.png)
## Fork/Join的使用
Fork/Join是一个并行的计算框架，用来支持分治任务模型。Fork对应任务模型的任务分解，Join对应的是计算结果合并。Fork/Join框架包含两部分，一部分是分治任务的线程池ForkJoinPool，一部分是分治任务ForkJoinTask，这两部分的关系类似于ThreadPoolExecutor于Runnable的关系，都可以理解为提交任务到线程池  

ForkJoinTask是一个抽象类，核心的方法是fork()于join()方法，其中fork方法会异步执行一个子任务，而join方法则会阻塞当前线程来等待子任务的执行结果。ForkJoinTask有两个子类RecursiveAction 和 RecursiveTask，这两个子类都定义了抽象的compute()方法，区别是RecursiveAction的compute方法没有返回值，而RecursiveTask则有返回值。

代码示例：
```
    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool(4);
        //创建任务
        FibonacciTask task = new FibonacciTask(10);
        //启动任务
        Integer result = pool.invoke(task);
        System.out.println(result);
    }

    static class FibonacciTask extends RecursiveTask<Integer>{

        private final int n;

        public FibonacciTask(int n) {
            this.n = n;
        }

        @Override
        protected Integer compute() {
            if(n <=1) return n;
            FibonacciTask f1 = new FibonacciTask(n-1);
            //异步执行子任务f1
            f1.fork();
            FibonacciTask f2 = new FibonacciTask(n-2);
            //f2再当前线程中执行
            Integer r2 = f2.compute();
            Integer r1 = f1.join();
            return r2 + r1;
        }
    }
```
特别需要注意的点：  
1. 一定不能写成如下方式
```
// 分别对子任务调用fork():
subtask1.fork();
subtask2.fork();
```
因为这样，两个子任务都是在子线程中执行，而当前线程会空闲，而子任务可能又fork两个子子线程，子线程也会变成空闲，这样就会浪费很多线程
2. 需要先f2.compute(),再f1.join()；如果反过来则会因为f1.join()导致线程阻塞，f1完成之后才执行f2导致并发度降低