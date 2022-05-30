### 介绍
使用线程池可以减少线程的频繁创建和销毁，使用合适的线程池可以极大的提升服务性能。举例来说，我们把应用当做一个银行，一个线程就当做一个柜台，一次请求当做一个客户，在现实生活中，没有银行会在一个客户来办理业务的时候就增加一个柜台。大部分情况下是这样的：当前客户较少，可以为每一个客户安排一个柜台提供服务，如果所有柜台都在为客户办理业务，那么后来的客户可以在等候区等待，如果客户太多，等候区也容纳不了，可以增加一些临时柜台办理业务，而如果以上措施还不能满足客户业务办理的速度，只能暂停后续客户的业务办理请求或者让客户在大堂外等待有柜台空出或者等待区空出，如果柜台持续没有客户办理业务那么可以暂停一些柜台的开放。而以上这种情景也适用于我们工作开发中，在多线程情况下，可以使用线程池减少线程频繁的创建和销毁的开销，java中的线程池实现为```ThreadPoolExecutor```。
### 使用示例
举例创建一个线程池，核心线程数为1，最大线程数为2，任务队列长度为1，线程工厂为默认方法，拒绝策略为主线程执行，非核心线程最长空闲存活时间为0毫秒。
```
public class ThreadPoolTest {

    public static void main(String[] args)throws Exception {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1,2,0L, TimeUnit.MILLISECONDS,new ArrayBlockingQueue<>(1), Executors.defaultThreadFactory(),new ThreadPoolExecutor.CallerRunsPolicy());
        threadPoolExecutor.execute(()->{
            System.out.println("done");
        });
    }
}
```
### 源码分析
#### 重要参数和函数
```
    /**
     *ctl是一个AtomicInteger的参数，它分为两部分，高三位为线程池的状态，剩余29位是线程池的线程数
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //29
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //所以线程池最大线程数为536870911
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    //线程池状态，由小到大RUNNING<SHUTDOWN<STOP<TIDYING<TERMINATED，状态只能往上变
    //线程池可以根据最小为什么状态或者最大为什么状态作为条件
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 线程池状态，取高3位
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    //任务线程数量，取低29位
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    //计算ctl的值，高3位和低29位进行或运算
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    //线程池是否处于运行状态，因为小于SHUTDOWN的只有RUNNING
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
    //任务队列
    private final BlockingQueue<Runnable> workQueue;
    //工作线程集合
    private final HashSet<Worker> workers = new HashSet<Worker>();
    //线程池的最大到达大小
    private int largestPoolSize;
    //总完成任务数
    private long completedTaskCount;
    //线程工厂
    private volatile ThreadFactory threadFactory;
    //拒绝策略
    private volatile RejectedExecutionHandler handler;
    //线程最大空闲时间
    private volatile long keepAliveTime;
    //是否允许核心线程销毁
    private volatile boolean allowCoreThreadTimeOut;
    //核心线程数
    private volatile int corePoolSize;
    //最大线程数
    private volatile int maximumPoolSize;

```
#### 构造函数
```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        //设置核心线程数和最大线程数，核心线程数不能小于0，最大线程数须大于等于核心线程数，空闲存活时间不能小于0
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        //任务队列，线程构造工厂和拒绝策略不可为空
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        //将设置空闲存活时间转位纳秒
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
#### 线程提交
```
public void execute(Runnable command) {
        //如果线程为空，则抛空指针异常
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //线程执行线程池会有三种情况
        //1、如果存活的线程数小于核心线程数，则创建一个核心线程，对于核心线程，会尽快创建到设置的数量
        //2、如果已经创建足够的核心线程，那么会将提交的线程放到任务队列，放入的条件是线程池正在运行
        //2.1、获取线程池最新状态，，如果线程池不处于运行状态且任务移除成功，则对线程执行拒绝策略
        //2.2、如果线程池中的线程数为0，那么就创建一个非核心线程，这种属于极端场景，譬如线程池核心线程数为0或者核心线程被回收才会发生
        //3、如果往任务队列添加任务失败也就是任务队列已满，那么就创建非核心线程，如果非核心线程创建失败则执行拒绝策略
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
除这个方法之外，线程池还有另外一个方法将线程添加到线程池中，和```execute```不同的是，```submit```是有返回值的，该方法会将提交的命令包装成```RunnableFuture```然后调用```execute```方法：
```
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```
#### 添加任务
```
    private boolean addWorker(Runnable firstTask, boolean core) {
        //判断是否需要增加工作线程
        //1、如果线程池状态不为运行中，且不满足（状态为关闭，同时提交的任务为null，并且工作线程不为空），那么就不会继续往下走，也就是不会添加新的工作线程
        //2、如果工作线程数大于线程池的最大容量，也就是(1<<29)-1，或者如果需要创建的线程为核心线程但是当前核心线程数大于等于设置参数，或者如果需要创建的线程为非核心线程但是当前最大线程数大于等于设置参数，那么就不会继续往下走，也就是不会添加新的工作线程
        //3、如果工作线程数增加成功则跳出循环继续往下执行
        //4、如果工作线程数增加失败则失败则判断当前线程池状态，如果和前次状态相同则继续从第二步开始执行，如果不想听则跳出当前循环从第一步开始执行
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    //获取线程池最新状态，如果线程池运行中或者虽然线程池已关闭但是提交的任务为空才会创建新的工作线程
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果新建的工作线程已启动则抛异常，否则则将工作线程添加到线程集合里，并启动线程
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
#### 任务执行
```
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //如果执行任务不为空或者从任务队列获取到新的任务，那么就执行获取到的任务，所以核心逻辑需要看getTask方法，也就是任务获取
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 如果线程池状态不为关闭或运行状态或者（当前线程已中断并且状态不为关闭或运行状态），那么就中断当前线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    //执行前方法，默认为空，可由子类扩展
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
                        //执行后方法，默认为空，可由子类扩展
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
            //线程退出
            processWorkerExit(w, completedAbruptly);
        }
    }
```
#### 任务获取
```
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果线程池已停止或者任务队列为空，那么就将工作线程减一并返回null，在外层循环获取不到任务时，工作线程执行完成退出
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 判断是否可以移除工作线程，条件是是否允许核心线程销毁或者当前工作线程数大于核心线程数
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //如果工作线程数大于最大线程或者满足（可以移除工作线程且获取任务已超时），并且（工作线程大于1或者任务队列为空），那么就尝试将工作线程减一，它和decrementWorkerCount的区别是decrementWorkerCount会不断尝试直到成功，而compareAndDecrementWorkerCount只尝试一次
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //如果可以移除工作线程，那么获取任务就已设置的等待时间限时等待，否则就一直等待，如果获取不到任务则设置超时标志为true，也就是下次循环再获取不到任务就将移除该线程
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
#### 线程退出
```
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        //判断工作线程是否意外退出，如果意外退出则需要移除线程，意外退出是任务线程中断异常，或者钩子方法执行异常等情况下才会出现
        if (completedAbruptly) 
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //统计完成任务数并将工作线程移出队列
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
        //尝试中断线程池
        tryTerminate();

        int c = ctl.get();
        //线程池的状态是运行中或者关闭
        if (runStateLessThan(c, STOP)) {
            //任务线程不是意外退出
            if (!completedAbruptly) {
                //线程池最少线程数量，如果不允许核心线程超时那么就是核心线程数，否则为0
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                //如果线程池最少线程数为0，但是任务队列还有任务，那么线程数最少为0
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                //如果线程池线程大于最少线程则不处理
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //增加一个线程
            addWorker(null, false);
        }
    }

    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            //如果线程池状态为关闭且任务队列为空，或者状态为停止，就去中断线程池
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            //如果此时工作线程不为0，也就是当前线程不是最后一个执行完成的线程，就中断一个空闲的任务线程，确保关闭信号传播
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //使用cas将ctl的状态改为TIDYING，线程数改为0
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //执行钩子方法
                        terminated();
                    } finally {
                        //最终将ctl的状态改为TERMINATED，线程数改为0
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```