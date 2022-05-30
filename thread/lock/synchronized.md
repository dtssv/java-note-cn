### 介绍
```synchronized```是java原生锁实现，只需要在需要加锁的方法或者代码块增加```synchronized```关键字，即可加锁完成，当并发情况下，只有获取锁的线程可以执行代码，其他线程需要等待锁被释放后获取锁才能执行相关代码。```synchronized```是可重入锁，同时它的实现是非公平锁。 ```synchronized```修饰的对象根据状态分为无锁、偏向锁、轻量级锁和重量级锁四种状态，锁的状态存储在对象头的markword，下面是两张图分别展示在32位虚拟机和64位虚拟机下对象不同状态的markword内容。
#### 32位
![32位](https://github.com/dtssv/java-note-cn/blob/8e4e6ee3e17b7f06968e7ed69c4ba25fd719ec1b/static/img/32位.png "32位")
#### 64位
![64位](../../static/img/64位.png "64位")
### 使用
```synchronized```根据加锁的方式可以分为方法锁、对象锁和类锁，它们的用法分别如下：
```
public class LockTest {
    //静态对象
    static Object lockS = new Object();

    //普通对象
    Object lockN = new Object();

    public static void main(String[] args)throws Exception {


    }
    //静态方法类锁
    public synchronized static void methodS() throws Exception{
        Thread.sleep(10000L);
        System.out.println();
        System.out.println(Thread.currentThread().getName() + " sm  done");
    }
    //普通方法锁
    public synchronized void methodN() throws Exception{
        Thread.sleep(10000L);
        System.out.println();
        System.out.println(Thread.currentThread().getName() + " nm done");
    }
    //静态对象类锁
    public void testS(){
        new Thread(()->{
           synchronized (lockS){
               try {
                   Thread.sleep(10000L);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
               System.out.println(Thread.currentThread().getName() + " so done");
           }
        }).start();
    }
    //普通对象对象锁
    public void testN(){
        new Thread(()->{
            synchronized (lockN){
                try {
                    Thread.sleep(10000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " no done");
            }
        }).start();
    }
}
```
#### 锁竞争
| 线程A|线程B|是否竞争|
|---|---|---|
|testN|testN|是|
|testN|methodN|否|
|testN|testS|否|
|methodN|methodN|是|
|methodN|methodS|否|
|testS|methodS|否|
#### 锁获取流程

### 原理
首先执行如下命令```javap -c -p LockTest```查看类编译后生成的字节码如下：
```
public class com.zyf.test.layout.LockTest {
  static java.lang.Object lockS;

  java.lang.Object lockN;

  public com.zyf.test.layout.LockTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: new           #2                  // class java/lang/Object
       8: dup
       9: invokespecial #1                  // Method java/lang/Object."<init>":()V
      12: putfield      #3                  // Field lockN:Ljava/lang/Object;
      15: return

  public static void main(java.lang.String[]) throws java.lang.Exception;
    Code:
       0: new           #4                  // class com/zyf/test/layout/LockTest
       3: dup
       4: invokespecial #5                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: invokevirtual #6                  // Method testS:()V
      12: new           #7                  // class java/lang/Thread
      15: dup
      16: invokedynamic #8,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable;
      21: invokespecial #9                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
      24: invokevirtual #10                 // Method java/lang/Thread.start:()V
      27: return

  public static synchronized void methodS() throws java.lang.Exception;
    Code:
       0: ldc2_w        #11                 // long 10000l
       3: invokestatic  #13                 // Method java/lang/Thread.sleep:(J)V
       6: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
       9: invokevirtual #15                 // Method java/io/PrintStream.println:()V
      12: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
      15: new           #16                 // class java/lang/StringBuilder
      18: dup
      19: invokespecial #17                 // Method java/lang/StringBuilder."<init>":()V
      22: invokestatic  #18                 // Method java/lang/Thread.currentThread:()Ljava/lang/Thread;
      25: invokevirtual #19                 // Method java/lang/Thread.getName:()Ljava/lang/String;
      28: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      31: ldc           #21                 // String  sm  done
      33: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      36: invokevirtual #22                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      39: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      42: return

  public synchronized void methodN() throws java.lang.Exception;
    Code:
       0: ldc2_w        #11                 // long 10000l
       3: invokestatic  #13                 // Method java/lang/Thread.sleep:(J)V
       6: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
       9: invokevirtual #15                 // Method java/io/PrintStream.println:()V
      12: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
      15: new           #16                 // class java/lang/StringBuilder
      18: dup
      19: invokespecial #17                 // Method java/lang/StringBuilder."<init>":()V
      22: invokestatic  #18                 // Method java/lang/Thread.currentThread:()Ljava/lang/Thread;
      25: invokevirtual #19                 // Method java/lang/Thread.getName:()Ljava/lang/String;
      28: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      31: ldc           #24                 // String  nm done
      33: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      36: invokevirtual #22                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      39: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      42: return

  public void testS();
    Code:
       0: new           #7                  // class java/lang/Thread
       3: dup
       4: invokedynamic #25,  0             // InvokeDynamic #1:run:()Ljava/lang/Runnable;
       9: invokespecial #9                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
      12: invokevirtual #10                 // Method java/lang/Thread.start:()V
      15: return

  public void testN();
    Code:
       0: new           #7                  // class java/lang/Thread
       3: dup
       4: aload_0
       5: invokedynamic #26,  0             // InvokeDynamic #2:run:(Lcom/zyf/test/layout/LockTest;)Ljava/lang/Runnable;
      10: invokespecial #9                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
      13: invokevirtual #10                 // Method java/lang/Thread.start:()V
      16: return

  private void lambda$testN$2();
    Code:
       0: aload_0
       1: getfield      #3                  // Field lockN:Ljava/lang/Object;
       4: dup
       5: astore_1
       6: monitorenter
       7: ldc2_w        #11                 // long 10000l
      10: invokestatic  #13                 // Method java/lang/Thread.sleep:(J)V
      13: goto          21
      16: astore_2
      17: aload_2
      18: invokevirtual #28                 // Method java/lang/InterruptedException.printStackTrace:()V
      21: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
      24: new           #16                 // class java/lang/StringBuilder
      27: dup
      28: invokespecial #17                 // Method java/lang/StringBuilder."<init>":()V
      31: invokestatic  #18                 // Method java/lang/Thread.currentThread:()Ljava/lang/Thread;
      34: invokevirtual #19                 // Method java/lang/Thread.getName:()Ljava/lang/String;
      37: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      40: ldc           #29                 // String  no done
      42: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      45: invokevirtual #22                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      48: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      51: aload_1
      52: monitorexit
      53: goto          61
      56: astore_3
      57: aload_1
      58: monitorexit
      59: aload_3
      60: athrow
      61: return
    Exception table:
       from    to  target type
           7    13    16   Class java/lang/InterruptedException
           7    53    56   any
          56    59    56   any

  private static void lambda$testS$1();
    Code:
       0: getstatic     #30                 // Field lockS:Ljava/lang/Object;
       3: dup
       4: astore_0
       5: monitorenter
       6: ldc2_w        #11                 // long 10000l
       9: invokestatic  #13                 // Method java/lang/Thread.sleep:(J)V
      12: goto          20
      15: astore_1
      16: aload_1
      17: invokevirtual #28                 // Method java/lang/InterruptedException.printStackTrace:()V
      20: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
      23: new           #16                 // class java/lang/StringBuilder
      26: dup
      27: invokespecial #17                 // Method java/lang/StringBuilder."<init>":()V
      30: invokestatic  #18                 // Method java/lang/Thread.currentThread:()Ljava/lang/Thread;
      33: invokevirtual #19                 // Method java/lang/Thread.getName:()Ljava/lang/String;
      36: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      39: ldc           #31                 // String  so done
      41: invokevirtual #20                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      44: invokevirtual #22                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      47: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      50: aload_0
      51: monitorexit
      52: goto          60
      55: astore_2
      56: aload_0
      57: monitorexit
      58: aload_2
      59: athrow
      60: return
    Exception table:
       from    to  target type
           6    12    15   Class java/lang/InterruptedException
           6    52    55   any
          55    58    55   any

  private static void lambda$main$0();
    Code:
       0: invokestatic  #32                 // Method methodS:()V
       3: goto          11
       6: astore_0
       7: aload_0
       8: invokevirtual #34                 // Method java/lang/Exception.printStackTrace:()V
      11: return
    Exception table:
       from    to  target type
           0     3     6   Class java/lang/Exception

  static {};
    Code:
       0: new           #2                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: putstatic     #30                 // Field lockS:Ljava/lang/Object;
      10: return
}

```
#### 对象锁
从字节码可以看到在方法```lambda$testN$2```的第6行```6: monitorenter```在执行同步代码块之前会先获取锁的监视器，而在执行完毕之后也就是第52行```52: monitorexit```会释放锁的监视器，如果执行异常则也会释放锁的监视器，也就是第58行```58: monitorexit```。以上代码翻译成显示锁则相当于：
```
        LOCK.lock();
        try {
            Thread.sleep(10000L);
            System.out.println(Thread.currentThread().getName() + " no done");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            LOCK.unlock();
        }
```
#### 方法锁
对于通过在方法加锁的方式实现起来和对象锁略有区别，对```synchronized```修饰的方法会增加```ACC_SYNCHRONIZED```标识这是一个同步方法，调用方法的线程需要预先获得锁的监视器。
#### 锁优化
JVM也在不断的对内置锁进行优化，已期达到和显示锁相同的效率，优化手段主要有以下几种：
##### 自旋锁
自旋锁是为了防止多处理器并发而引入的一种锁，自旋锁可以避免频繁从用户态切换到内核态去争抢锁，如果当前锁已被其他线程获取，那么该线程将循环等待以获取锁，java默认自旋10次，但是之后会根据之前自旋的结果调整自旋次数，如果之前自旋获取锁成功那么下次会尝试自旋更多次，如果之前自旋获取 锁失败，那么下次就会减少自旋次数或者不自旋。
###### 自旋锁优点
减少线程的阻塞，在竞争不激烈且锁占用时间很短的情况下，可以极大的提升性能。
###### 自旋锁缺点
在线程竞争激烈或者占用时间很长的情况下，自旋会使cpu做无用功，反而影响性能。
##### 锁消除
如果一个同步方法或者同步代码块不会产生线程的锁竞争，那么在其上的```synchronized```就会消除，举例如下：
```
    public void testN(){
        Object lockN = new Object();
        synchronized (lockN){
            try {
                Thread.sleep(10000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("done");
        }
    }
```
在这个情况下同步代码块会消除，因为```lockN```对象是线程栈上分配，不会产生多线程锁竞争。当然，这是一个极端举例，常用的情况譬如```StringBuffer.append()```方法是同步方法，如果声明的```StringBuffer```对象是在方法内那么同步是可以消除的。
##### 锁粗化
锁粗化是指如果多个代码块都要获取同一个锁，那么JVM会将这些获取锁的操作合并为一个，这样会减少重复获取锁的操作，举例如下：
```
    //合并前
    public void testN(){
        synchronized (lockN){
            System.out.println("done-1");
        }
        synchronized (lockN){
            System.out.println("done-2");
        }
    }
    //合并后
    public void testN(){
        synchronized (lockN){
            System.out.println("done-1");
            System.out.println("done-2");
        }
    }  
```
### MarkWord分析
我们使用jol来对对象头进行分析，需要增加maven依赖：
```
        <dependency>
            <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-core</artifactId>
            <version>${jol.version}</version>
        </dependency>
```
这里推荐使用最新版本，新版本展示效果更佳直观。
代码运行前会先等待10秒，```Thread.sleep(10000L);```，这是因为jvm启动时默认把偏向锁的启用延迟了4秒，因为启动时需要加载大量资源，此时启用偏向意义不大。
#### 无锁
初始状态，如果之前未获取过轻量级锁或者重量级锁，那么处于可偏向状态，如果获取过轻量级锁或重量级锁，那么处于不可偏向状态。
##### 演示代码
```
public class SyncTest {
    Object lock = new Object();
    ClassLayout lockLayout = ClassLayout.parseInstance(lock);
    public static void main(String[] args)throws Exception{
        Thread.sleep(10000L);
        SyncTest syncTest = new SyncTest();
        System.out.println(syncTest.lockLayout.toPrintable());
    }
}
```
##### 打印结果
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000000000000d (biasable; age: 1)
  8   4        (object header: class)    0x00001000
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
我们将markword转换成2进制结果如下```0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 1101```，根据markword在64位虚拟机下的结构可知锁标志位为```01```，是否偏向锁```1```，分代年龄```0001```，hashCode为空。
#### 偏向锁
锁对象会将获取锁的第一个线程的线程ID存在对象头，如果后续还是该线程获取锁，那么不用通过自旋或者获取锁的监视器而直接获取锁，如果锁对象偏向其他线程则会尝试通过cas替换为当前线程ID，成功则获取偏向锁，失败则意味着产生锁竞争，需要撤销偏向锁。
##### 演示代码
```
public class SyncTest {
    Object lock = new Object();
    ClassLayout lockLayout = ClassLayout.parseInstance(lock);
    public static void main(String[] args)throws Exception{
        Thread.sleep(10000L);
        SyncTest syncTest = new SyncTest();
        System.out.println(syncTest.lockLayout.toPrintable());
        new Thread(()->{
            synchronized (syncTest.lock){
                System.out.println(syncTest.lockLayout.toPrintable());
            }
        }).start();
    }
}

```
##### 打印结果
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000000000000d (biasable; age: 1)
  8   4        (object header: class)    0x00001000
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f82362d080d (biased: 0x0000001fe08d8b42; epoch: 0; age: 1)
  8   4        (object header: class)    0x00001000
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
在加锁前对象的markword和无锁状态一致，加锁后的markword转换为二进制```0000 0000 0000 0000 0000 0000 0111 1111 1000 0010 0011 0110 0010 1101 0000 1000 0000 1101```，有此可知锁标志位为```01```，是否偏向锁```1```，分代年龄```0001```，未使用位```0```，Epoch位```00```，偏向线程id```1111111100000100011011000101101000010```，转换为16进制为```0x1fe08d8b42```。

我们知道对象的hashCode一级生产就不会再变更了，对象的hashCode存储在对象的markword里，那么明显和偏向锁的结构有冲突，那如果我们在上边代码先调用对象的hashCode方法```syncTest.lock.hashCode();```和获取偏向锁之后调用hashCode的方法输出结果分别如下（省略无锁状态的输出）：
###### 获取锁之前
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000030abf0a10 (thin lock: 0x000000030abf0a10)
  8   4        (object header: class)    0x00001000
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
此时对象markword转换为二进制```0000 0000 0000 0000 0000 0000 0000 0000 0000 0011 0000 1010 1011 1111 0000 1010 0001 0000```，根据markword在64位虚拟机下的结构可知锁标志位为```00```，也就是轻量级锁，指向栈中锁记录的指针为```1100 0010 1010 1111 1100 0010 1000 0100```，转换为16进制为```0xc2afc284```。
###### 获取锁之后
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fdbac81e80d (biased: 0x0000001ff6eb207a; epoch: 0; age: 1)
  8   4        (object header: class)    0x00001000
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fdbb000fe02 (fat lock: 0x00007fdbb000fe02)
  8   4        (object header: class)    0x00001000
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
由结果可知，在hashCode方法执行之前，线程获取的锁还是偏向锁，但是在获取锁之后执行hashCode方法，锁膨胀为重量级锁，将markword转换为二进制为```0000 0000 0000 0000 0111 1111 1101 1011 1011 0000 0000 0000 1111 1110 0000 0010```，如图可知，锁标志位为```10```，指向重量级锁的指针为```0001 1111 1111 0110 1110 1100 0000 0000 0011 1111 1000 0000```，转换为16进制为```0x1ff6ec003f80```。  
由实现可知，锁对象一旦获取了hashCode，则线程就不能获取偏向锁，在调用hashCode之后获取锁得到的是轻量级锁，在获取偏向锁之后调用hashCode，偏向锁膨胀为重量级锁。
#### 轻量级锁
重量级锁需要转入内核态获取锁的监视器，用户态切换到内核态是一个比较重的操作，所以JVM对此进行了优化，会先尝试自旋已等待获取锁，如果自旋成功则获取锁，否则则膨胀为重量级锁。
##### 演示代码
```
public class SyncTest {
    Object lock = new Object();
    ClassLayout lockLayout = ClassLayout.parseInstance(lock);
    public static void main(String[] args)throws Exception{
        Thread.sleep(10000L);
        SyncTest syncTest = new SyncTest();
        System.out.println(syncTest.lockLayout.toPrintable());
        new Thread(()->{
            synchronized (syncTest.lock){
                System.out.println(syncTest.lockLayout.toPrintable());
            }
        }).start();
        Thread.sleep(10L);
        new Thread(()->{
            synchronized (syncTest.lock){
                System.out.println(syncTest.lockLayout.toPrintable());
            }
        }).start();
    }
    Thread.sleep(1000L);
    System.out.println(syncTest.lockLayout.toPrintable());
}
```
##### 打印结果
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000020a5eee6005 (biased: 0x000000008297bb98; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000007e9b9ff2c0 (thin lock: 0x0000007e9b9ff2c0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
虽然这个示例不能保证每次偏向锁释放不偏向其他线程，但是多运行几次还是可以得到轻量级锁竞争的情况，分析markword：
1. 在线程加锁前markword为和无锁状态的差别在于分代年龄，其他一致，可知，当前为无锁可偏向状态。
2. 第一个线程获取锁后的markword为```0000 0000 0000 0000 0000 0010 0000 1010 0101 1110 1110 1110 0110 0000 0000 0101```，可知当前锁处于偏向状态，偏向线程为```0x8297bb98```。
3. 当产生锁竞争膨胀为轻量级锁后的markword为```0000 0000 0000 0000 0000 0111 1110 1001 1011 1001 1111 1111 0010 1100 0000```，可知当前锁处于轻量级锁状态，指向栈中锁记录的指针为```0xfd373fe5```
4. 当所有锁释放后的markword为和无锁状态的差别在于分代年龄和偏向标志其他一致，可知，当前为无锁不可偏向状态。
#### 重量级锁
重量级锁需要线程切换到内核态获取锁的监视器。
##### 演示代码
```
public class SyncTest {

    Object lock = new Object();
    ClassLayout lockLayout = ClassLayout.parseInstance(lock);
    public static void main(String[] args)throws Exception{
        Thread.sleep(10000L);
        SyncTest syncTest = new SyncTest();
        System.out.println(syncTest.lockLayout.toPrintable());
        new Thread(()->{
            synchronized (syncTest.lock){
                System.out.println(syncTest.lockLayout.toPrintable());
                try {
                    Thread.sleep(100L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        new Thread(()->{
            synchronized (syncTest.lock){
                System.out.println(syncTest.lockLayout.toPrintable());
                try {
                    Thread.sleep(100L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```
##### 打印结果
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000024ef5b45005 (biased: 0x0000000093bd6d14; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000024ef10f7e2a (fat lock: 0x0000024ef10f7e2a)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```
启动两个线程争抢锁，获取到锁的线程休眠100毫秒确保另外一个线程自旋获取锁失败膨胀为重量级锁，分析markword：
1. 加锁前为无锁可偏向状态。
2. 第一个线程获取锁为偏向锁，偏向线程为```93bd6d14```，获取锁后休眠100毫秒。
3. 第二个线程获取锁，首先尝试自旋获取锁，在尝试一定次数后获取锁失败，膨胀为重量级锁，markword为```0000 0000 0000 0000 0000 0010 0100 1110 1111 0001 0000 1111 0111 1110 0010 1010```，可知当前处于重量级锁定状态，执行锁监视器的指针为```93bc43df```。