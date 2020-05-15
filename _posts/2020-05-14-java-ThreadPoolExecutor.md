---
layout: post
title: "java线程池ThreadPoolExecutor学习"
subtitle: 'ThreadPoolExecutor'
author: "W"
header-style: text
tags:
  - Java

---

### 创建线程池

* Executors 具体参考官方文档 ``` 实际开发不建议使用  ```
* 自定义线程池  ```ThreadPoolExecutor ```

	__本次主要是针对ThreadPoolExecutor做分析__

### ThreadPoolExecutor初始化参数

![image-20200514201708672](/img/java/thread/threadPoolExecutor.png)

 * corePoolSize ```核心线程数 当创建线程池时线程池并不会一开始就初始化指定的corePoolSize线程 是当有任务执行时初始化核心线程 如果有需要初始化核心线程数 可以通过方法 prestartCoreThread()、prestartAllCoreThreads() 来实现```

 * maximumPoolSize ```线程池最大线程数 当workQueue没有可用位置的时候 线程池会进行新增非核心线程来处理任务 当指定的workQueue为无界队列时此参数无效```

 * keppAliveTime ```非核心线程生存时间```

 * unit ```生存单位```

 * workQueue ```阻塞队列```

 * threadFactory ```线程生产工厂```

 * rejectHandler ```拒绝策略 默认Abort(抛出异常)```



### ThreadPoolExecutor类具体分析

首先先看类中的关于线程池状态和线程数的表示

![sp20200514_202350_630](/img/java/thread/ThreadPoolClassState.png)

* ctl 综合表示线程池状态和线程数的一个原子类变量 ``` 高3位表示线程池状态, 后29位表示线程个数 具体参数可以自己创建demo 然后debug跟一下源码```
* RUNNING``` 运行状态```
* SHUTDOWN ```关闭状态```
* STOP ```停止状态```
* TIDYING``` 整理状态```
* TERMINATED ```停止状态```
* runStateOf() ``` 获取当前线程池状态```
* workCountOf() ``` 线程池中的线程数```

线程池状态图
![image-20200514201708672](/img/java/thread/threadpoolstate.png)
```当线程池处于running状态时表示可以处理外部传过来的任务,当处理不过来时会放入workQueue;如果调用了shutdown() 表示线程池拒绝接受新的任务,但仍然会处理已经存在于workQueue中的任务;调用了shutdownNow() 线程池会将在workQueue中的任务逐一移除并返回 并转换到Tidying状态 执行tryTerminate()清尾工作最终转换为stop状态```

ThreadPoolExecutor的execute方法

```java
public void execute(Runnable command) { 
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //判断当前线程数是否小于核心线程
        if (workerCountOf(c) < corePoolSize) {
        	//新增核心线程任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //判断线程池是否为Running状态 && 追加任务到workQueue的末尾
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //如果线程池状态不为Running状态 则移除当前任务并且拒绝此次添加动作
            if (! isRunning(recheck) && remove(command))
                reject(command);
                //新增非核心线程来处理任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //新增任务失败则进行拒绝
        else if (!addWorker(command, false))
            reject(command);
    } 
```

从代码上可以看出来线程池添加任务具体的代码就在addWorker这个方法中，接下来进入addWorker代码解析

```java
	 private boolean addWorker(Runnable firstTask, boolean core) {
         //外层循环标记
        retry:
         //外层死循环
        for (;;) {
            //获取线程池综合信息
            int c = ctl.get();
            //线程池当前状态
            int rs = runStateOf(c);
			
            //1判断当前线程池是否为大于SHUTDOWN(非running)状态
            //2当前线程池状态为SHUTDOWN(不接收新任务)并且传入的执行任务为null并且阻塞队列不为空
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
			//内层循环
            for (;;) {
                //获取当前线程池有效线程数
                int wc = workerCountOf(c);
                //1 wc >= CAPACITY 当前线程数是否大于初始值
                //2 当前线程数是否大于等于 核心线程数或者线程池最大线程数(主要看方法入参是否为核心线程)
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //cas操作改变ctl的值 成功则跳出外循环进行下一步操作
                //方法是将workcount的值+1
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //再次获取线程池状态 因为再循环阶段可能会改变线程池状态
                c = ctl.get();  // Re-read ctl
                //如果线程池状态不是一开始获取的将再次执行外循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
		
         //执行到这说明cas操作已经成功
        boolean workerStarted = false;//worker是否启动标识
        boolean workerAdded = false;//worker是否添加成功
        Worker w = null;
        try {
            //任务执行具体类(Work类分析时会讲)
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                //加锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                  
                    //获取线程池状态
                    int rs = runStateOf(ctl.get());
					
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //判断线程是否存活
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //添加w任务到set集合
                        workers.add(w);
                        int s = workers.size();
                        //更新largestPoolSize
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //修改添加标识为true
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //如果添加成功启动线程
                //修改启动标识
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //启动失败执行失败方法 
            //方法具体做法就是将失败前添加到works的任务remove
            //中断Worker关联的线程
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

到这里添加任务的步骤解析已经完成，下面就是具体执行的代码分析。

首先在代码中线程池会将任务创建一个Worker对象，这是一个内部类，具体的代码执行就会在这里。

```java
	private final class Worker extends AbstractQueuedSynchronizer implements Runnable
    {
       
        private static final long serialVersionUID = 6138294804551838833L;

        final Thread thread;
        Runnable firstTask;
        volatile long completedTasks;

       
        Worker(Runnable firstTask) {
            setState(-1); 
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        public void run() {
            runWorker(this);
        }

     

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

可以看出Worker类本身就是一个可运行的线程类而且继承了AQS 实现了自己的锁机制。

在addWorker()中有判断workerAdded 状态的代码，会执行线程start(),而那个线程就是worker本身，所以代码会执行Worker类中的run方法，也就会去执行runWorker(this)，所以我们在跟进这个方法。

```java
final void runWorker(Worker w) {
    	//获取当前线程
        Thread wt = Thread.currentThread();
    	//获取runnable
        Runnable task = w.firstTask;
    	
        w.firstTask = null;
    	//更改当前线程的state为0 具体会去执行Worker类中tryRelease()
   		//允许中断
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 任务不为null
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

```java
  private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            //获取线程池综合信息
            int c = ctl.get();
            //获取线程池状态
            int rs = runStateOf(c);

            //1 Check if queue empty only if necessary.
            //2 线程池状大于等于SHUTDOWN并且 线程池状态大于等于STOP 或者 阻塞队列为空
            //执行cas操作使线程数-1 返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
			
            //获取线程池当前线程数
            int wc = workerCountOf(c);

            // Are workers subject to culling?
            //timed 是来确认是否关闭线程
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
			// wc > maximumPoolSize || (timed && timedOut) 当前有效线程数是否大于最大线程数 
            //  有效线程数>1 或者工作队列是空的
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //timed 为true 则执行workQueue.poll() 指定等待时常
               	//否则执行take() 阻塞当前线程
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                //判断r的值 如果为null 说明等待超时
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

相关的还有shutdown()和shutdownNow() 具体请查看源码进行查看。

还有线程复用的问题，如果细心应该会看出来，其实就是启动的coreSize数量的线程阻塞去获取阻塞队列里面的任务，如果阻塞队列没有任务就会一直在阻塞，本质其实是生产-消费。