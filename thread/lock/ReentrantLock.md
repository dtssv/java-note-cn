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
如果上一步获取锁失败，则调用```acquire```方法：  
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
