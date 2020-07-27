### 介绍
```ReentrantLock```顾名思义是一个可重入锁，可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。  
### 使用举例
举例一个最简单的使用场景，线程1先获取锁，休眠五秒然后释放锁，释放之后线程2获取锁。
```
public class ReentrantLockTest {
    public static final ReentrantLock LOCK = new ReentrantLock();
    public static void main(String[] args) {
        new Thread(()->test(),"First").start();
        new Thread(()->test(),"Second").start();
    }
    public static void test(){
        try {
            LOCK.lock();
            System.out.println(Thread.currentThread().getName() + "获取锁");
            Thread.sleep(5000l);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            LOCK.unlock();
            System.out.println(Thread.currentThread().getName() + "释放锁");
        }
    }
}
```
### 源码分析
首先看它的构造方法：  
```
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

```
```ReentrantLock```有两个构造方法，无参构造方法默认使用非公平锁，有参构造方法可以选择使用公平锁或者非公平锁，一般情况下都使用非公平锁，非公平锁效率更高，下面我们分别分析获取锁的逻辑。  
#### NonfairSync  非公平锁
非公平锁获取锁的方法：  
```
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
```
首先它会先尝试获取锁，如果获取成功就讲当前锁的占有着设为自己，否则排队等待。获取锁的方法是将标志位从0改为1，实现如下：  
```
    protected final boolean compareAndSetState(int expect, int update) {
        // 通过cas修改标志位的值，如果修改成功即为锁获取成功
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
如果上一步获取锁失败，则调用```acquire()```方法：  
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
```tryAcquire()```是一个模板方法，当前为非公平锁时的实现方式如下：  
```
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
```
所以还是要关注```nonfairTryAcquire()```，这个方法分两部分，如果当前没有其他线程持有锁，则直接尝试获取锁，如果获取成功返回True，或者是该线程持有锁，则将锁的状态为加1，返回获取成功，其他情况返回失败：  
```
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
上边是尝试获取锁，如果锁获取失败，则将当期获取锁请求放入等待队列```addWaiter()```:  
```
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
然后阻塞获取锁```acquireQueued()```，流程如下，获取当前节点的前一个节点，如果获取到的节点是head节点就尝试获取锁，如果成功就返回，如果不成功就调用```shouldParkAfterFailedAcquire()```和```parkAndCheckInterrupt()```，如果锁获取异常则调用```cancelAcquire()```:  
```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
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
```
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            //当信号值是-1时直接返回true
            return true;
        if (ws > 0) {
            //取节点的上级节点并判断信号值
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //将信号值改为-1
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);//挂起线程，直到许可可用或者被中断，内部调用UNSAFE.park()
        return Thread.interrupted();//返回线程中断状态并清除中断状态
    }
```
如果需要响应中断则最后会调用```selfInterrupt()```中断当前线程：  
```
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```
#### FairSync  公平锁
公平锁和非公平锁总体实现大致一样，区别是在获取锁方面略有不同，可以看到非公平锁当```state=0```时会直接尝试获取锁，当是公平锁会先调用```hasQueuedPredecessors()```：  
```
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
    }
```
```
    public final boolean hasQueuedPredecessors() {
        // 判断链表前边有没有元素，因为链表先进先出，所以此处保证了公平
        //链表前边没有元素的逻辑是   head==tail（也就是链表只有一个元素） 
        //或者head.next.thread == currentThread 
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
