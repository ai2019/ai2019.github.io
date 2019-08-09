---
layout: post
title:  "Java自定义线程池"
date:   2019-03-24 12:00:00
categories: Java
tags: thread-pool 线程池
excerpt: 在Java中实现自定义线程池
mathjax: true
---

* content
{:toc}
实现多线程时，不可避免地会用到线程池，虽然JDK提供了4种默认的线程池，但是在实际项目中还是需要自定义线程池。

### 1. ThreadPoolExecutor

Java中通过ThreadPoolExecutor实现自定义线程池，根据实际需求，设置不同的参数，配置不同类型的线程池：

- 核心线程数，指定线程池的核心线程数量
- 最大线程数，指定线程池中最大的线程数量
- 空闲过期时间，指定允许的线程空闲时间
- 空闲过期时间的单位，指定线程过期时间的单位
- 任务队列，指定线程池的阻塞队列
- 线程工厂，指定产生线程池中线程的工厂
- 拒绝策略，指定线程池不能接受提交的任务时的拒绝策略



> 注意：默认情况下，过期策略只适用于超出核心线程数的线程，核心线程在超期后也是不会被结束的。曾经遇到一位同学，错以为默认情况下核心线程也会过期。当然，如果想让核心线程也过期的话，可以通过allowCoreThreadTimeOut(true)允许核心线程也过期。



通过execute()或者submit()方法提交任务，最终都是执行的execute()方法，execute的执行逻辑如下：

1. 当线程池中没有空闲的线程，且线程的数量没有超过核心线程数时，线程池新起一个线程并执行提交的任务；
2. 当线程池中没有空闲的线程，且线程的数量超过核心线程数时，将任务追加到任务队列中进行排队，待线程池中有空闲的线程时，从任务队列中取出任务并执行；
3. 当线程池中没有空闲的线程，线程的数量超过核心线程数，且任务队列已满，但未超过最大线程数时，新起一个线程并执行提交的任务；
4. 当线程池中没有空闲的线程，线程的数量超过最大线程数，且任务队列已满时，线程池拒绝执行提交的任务，并指定给定的拒绝策略。

可以通过execute()的源码理解上面的逻辑：

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

特别注意源码中的`ctl`，该属性是AtomicInteger类型的，包含了两部分信息：

- 线程池的状态，用高3个比特表示
- 线程池中线程的数量，用剩余的29个比特表示

通过`ctl`可以同时获取线程的状态或线程的数量。

### 2. JDK提供的4种类型线程池

为了方便使用线程池，JDK在Executors中提供了4种类型的线程池：

#### 2.1 单线程的线程池

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

该线程池提供了1个线程，任务队列是LinkedBlockingQueue，注意任务较多时可能导致OOM。

#### 2.2 固定线程数的线程池

```java
ExecutorService executor = Executors.newFixedThreadPool(3);
```

该线程池提供了固定数量的线程池，任务队列是LinkedBlockingQueue，注意任务较多时可能导致OOM。

#### 2.3 单线程的线程池

```java
ExecutorService executor = Executors.newCachedThreadPool();
```

由于任务队列是SynchronousQueue，该线程池在接收到任务后，立刻新建一个线程执行任务，注意任务较多时导致的线程资源耗尽的问题。

#### 2.4 调度线程池

```java
ExecutorService executor = Executors.newScheduledThreadPool(3);
```

该线程池提供了延迟执行的功能，后面可以详细研究。



### 3. 自定义线程池

虽然JDK提供了4种类型的线程池，但是在实际业务种，JDK提供的线程池并不能满足需求，通常需要自定义线程池，例如，核心线程数和最大线程数的大小、自定义线程的名称等需求。

#### 3.1 任务队列类型

任务队列的类型是阻塞队列，实现方式有两种：

- ```java
  LinkedBlockingQueue
  ```

- ```java
  ArrayBlockingQueue
  ```

LinkedBlockingQueue没有大小限制，也就是说，线程池种的线程数量只会是核心线程的数量，来不及执行的任务会被添加到阻塞队列中。因此，注意OMM问题。

ArrayBlockingQueue底层是数组，可以限制大小。

#### 3.2 线程工厂

首先，看下默认的线程工厂：

```java
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```

默认线程工厂的线程组和线程名称前缀不便于业务区分，因此可以自定义，通过日志分析不同的业务日志：

```java
public class OrderThreadFactory implements ThreadFactory {

    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    OrderThreadFactory() {
        group = new ThreadGroup("order-thread-group");
        namePrefix = "pool-" +
                poolNumber.getAndIncrement() +
                "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                namePrefix + threadNumber.getAndIncrement(),
                0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

此外，可以定义线程的优先级。

### 3.3 拒绝策略

（1）抛出拒绝异常策略

```java
    public static class AbortPolicy implements RejectedExecutionHandler {
        
        public AbortPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

（2）调用者自己执行策略

```java
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        
        public CallerRunsPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

（3）直接丢弃任务策略

```java
    public static class DiscardPolicy implements RejectedExecutionHandler {

        public DiscardPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

（4）丢弃最早任务策略

```java
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {

        public DiscardOldestPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

