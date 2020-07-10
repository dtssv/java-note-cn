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