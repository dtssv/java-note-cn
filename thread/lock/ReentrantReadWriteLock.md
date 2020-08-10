### 介绍
```ReentrantReadWriteLock```顾名思义是一个可重入的读写锁，它有两个锁，一个读锁，一个写锁，它也类似于可重入锁，实现方式分为公平锁和非公平锁，默认情况下是非公平锁，效率更高。读写锁的状态位分为两部门，高16位为读锁，低16位为写锁，其中写锁为独占锁，读锁为共享锁，写锁加锁逻辑和可重入锁大致相似，读锁则比较复杂。
#### 使用举例
举例一个最简单的使用场景，读线程1、2先获取读锁，休眠五秒然后释放锁，释放之后写线程1获取写锁，写锁释放后读线程3获取读锁，线程3释放后写线程2获取写锁，然后释放由读线程4获取读锁，由打印日志可知多个读线程可以同时获取读锁，读锁会阻塞写锁的获取，当某一个线程持有写锁时，其他线程无法获取读锁和写锁，总结如下：  
读读不互斥  
读写互斥  
写写互斥。
```
public class ReentrantReadWriteLockTest {
    public static final ReentrantReadWriteLock LOCK = new ReentrantReadWriteLock();
    public static void main(String[] args) {
        new Thread(()->test1(),"read1").start();
        new Thread(()->test1(),"read2").start();
        new Thread(()->test(),"write1").start();
        new Thread(()->test1(),"read3").start();
        new Thread(()->test(),"write2").start();
        new Thread(()->test1(),"read4").start();
    }

    public static void test(){
        try {
            System.out.println(Thread.currentThread().getName() + "进入");
            LOCK.writeLock().lock();
            System.out.println(Thread.currentThread().getName() + "获取锁");
            Thread.sleep(5000l);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            LOCK.writeLock().unlock();
            System.out.println(Thread.currentThread().getName() + "释放锁");
        }
    }
    public static void test1(){
        try {
            System.out.println(Thread.currentThread().getName() + "进入");
            LOCK.readLock().lock();
            System.out.println(Thread.currentThread().getName() + "获取锁");
            Thread.sleep(5000l);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            LOCK.readLock().unlock();
            System.out.println(Thread.currentThread().getName() + "释放锁");
        }
    }

}
```
#### 写锁
写锁的实现和可重入锁的实现大致类似，区别在状态值得计算判断相关，现在只看两块差一点：  
##### 非公平锁
获取锁：  
```
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. 如果读锁或者写锁的获取量不是0且所属线程不是当前线程则失败
             * 2. 如果count超过最大值则失败. (只有当count不为0时才会发生.)
             * 3. 否则，获取锁成功的话就设置为独占线程
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            //writerShouldBlock()非公平锁情况下一直是false
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```
释放锁：  
```
        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```
##### 公平锁
公平锁和非公平锁的区别在这里```writerShouldBlock()```，非公平锁模式下此处返回值为false，其他逻辑在可重入锁已经分析完毕，此处不过多分析：  
```
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        //此处和可重入锁公平锁逻辑一致
        public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
#### 读锁
##### 非公平锁
读锁的非公平锁实现如下：  
```
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
获取锁：  
```
        protected final int tryAcquireShared(int unused) {
            /*
             * 1. 如果有其他线程占有写锁则获取失败.
             * 2. 读锁获取是否需要阻塞，读锁数量小于最大值，cas读锁+1，如果满足条件则读锁获取成功.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```
