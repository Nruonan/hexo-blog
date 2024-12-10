---
title: Java并发理论基础
date: 2024-08-09 14:58:17
tags:
---



# 理论基础

## 并发三要素

### 可见性

**定义： 一个线程对共享变量的修改， 另外一个线程能够立刻看到。**

```java
// 线程1执行
int i = 0;
i = 10;

// 线程2执行
j = i;
```

根据上述代码，当线程1执行 i = 10这句时；会先把i的初始值加载到 CPU1的高速缓存中，然后赋值为10，那么CPU1的高速缓存中i的值变为10，却没有立刻写入到主存中，此时线程2去主存读取i的值并加载到CPU2的缓存当中，那么j = 0。

这就是可见性问题，线程1对变量i修改了之后，线程2没有立即看到线程1修改的值。

### 原子性

**定义： 一个操作或者多个操作，要么全部都执行并且不会被其他因素打断，要么都不执行。**

```java
int i = 1;

// 线程1执行
i += 1;

// 线程2执行
i += 1;
```

这里需要注意：` i += 1` 需要3条CPU指令：

1. 将变量 i 从内存读取到 CPU寄存器；
2. 在CPU寄存器中执行 i + 1 操作；
3. 将最后的结果i写入内存

因此在线程切换的情况下：

- 线程 1 读取到 `i` 的值 1。
- 然后线程 1 切换到线程 2。
- 线程 2 读取 `i` 的值（此时仍然是 1），然后进行加 1 操作，计算结果是 2。
- 线程 2 将结果 2 写回内存。
- 接着，线程 1 再次被调度，继续执行，发现 `i` 仍为 1， +1操作，计算结果为2

### 有序性

**定义：即程序执行的顺序按照代码的先后顺序执行。**

```java
int i =1;
String a = "ab"
i = 2;
a = "bc"
```

上面的代码从代码顺序上看，语句1在语句2前面，但JVM在真正执行这段代码**不一定**保证语句1在语句2前，为什么？

这里可能发生**指令重排序**

在执行程序时为了提高性能，编译器和处理器常常会对指令做重排序。重排序分三种类型：

- 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
- 指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-Level Parallelism， ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
- 内存系统的重排序。由于处理器使用缓存和读 / 写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

## 创建线程方式

### 继承 Thread类

创建一个类继承Thread 类，并重写 run 方法

```java
public class Test1 extends Thread{

    @Override
    public void run() {
        for (int i = 0; i < 100; i++){
            System.out.println(Thread.currentThread().getName() + " " + i);
        }
    }
 }
```

### 实现 Runnable 接口

创建一个类实现 Runnable接口 ，并重写 run 方法

```java
public class Test implements Runnable{

    @Override
    public void run() {
       for(int i = 0; i < 100; i++){
   			System.out.println(Thread.currentThread().getName() + " " + i);
       }
    }
    
    public static void main(String[] args) {
        Test test = new Test();

        Thread thread1 = new Thread(test, "线程1");
        Thread thread2 = new Thread(test, "线程2");
        thread1.start(); thread2.start();
    }
}
```

### 实现Callable接口

实现 Callable 接口，重写 call 方法，这种方式可以通过 FutureTask 获取任务执行的返回值。

```java
public class Test1 implements Callable<String>{
    @Override
   	public String call() throws Exception {
       	return "testtest";
   	}
    
    public static void main(String[] args) {
        //创建异步任务
        FutureTask<String> task=new FutureTask<String>(new Test1());
        //启动线程
        new Thread(task).start();
        try {
            //等待执行完成，并获取返回结果
            String result=task.get();
            System.out.println(result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

**通过继承 Thread 的方法和实现 Runnable 接口的方式创建多线程，哪个好？**

实现 Runnable 接口好，原因有两个：

- 避免了 Java 单继承的局限性，Java 不支持多重继承
- 适合多个相同的程序代码去处理同一资源的情况，把线程、代码和数据有效的分离，更符合面向对象的设计思想

## 线程常用方法

### run方法

默认的run() 方法是不会做任何事情，为了让线程执行任务，我们需要重写 run()方法。

- `run()`：封装线程执行的代码，直接调用相当于调用普通方法。
- `start()`：启动线程，然后由 JVM 调用此线程的 `run()` 方法。

### Sleep方法

使当前正在执行的线程暂停指定的毫秒数， 进入休眠的状态

```java
try {//sleep会发生异常要显示处理
    Thread.sleep(20);//暂停20毫秒
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

### yield方法

yield() 方法是一个静态方法，用于暗示当前线程愿意放弃其当前的时间片，允许其他线程执行。

```java
public void run() {
    Thread.yield();
}
```

### setDaemon方法

守护线程是程序运行时在后台提供服务的线程，不属于程序中不可或缺的部分。

当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。

main() 属于非守护线程。

使用 setDaemon() 方法将一个线程设置为守护线程。

```java
public static void main(String[] args) {
    Thread thread = new Thread(new MyRunnable());
    thread.setDaemon(true);
    thread.start()
}
```

## 线程状态转换

![线程转换图](life.png)

### 新建 New

处于 NEW 状态的线程此时尚未启动。这里的尚未启动指的是还没调用 Thread 实例的`start()`方法。

```java
private void test() {
    Thread thread = new Thread(() -> {});
    System.out.println(thread.getState()); // 输出 NEW
}
```

### 可运行 RUNABLE

可能正在运行，也可能正在等待 CPU 时间片。

包含了操作系统线程状态中的 Running 和 Ready.

### 阻塞 BLOCKED

阻塞状态，处于该状态的线程正等待其他线程释放锁以此进入同步区

### 无限期等待 WAITING

等待状态。处于等待状态的线程变成 RUNNABLE 状态需要其他线程唤醒。

调用下面这 3 个方法会使线程进入等待状态：

- `Object.wait()`：使当前线程处于等待状态直到另一个线程唤醒它；
- `Thread.join()`：等待线程执行完毕，底层调用的是 Object 的 wait 方法；
- `LockSupport.park()`：除非获得调用许可，否则禁用当前线程进行线程调度。

### 限期等待 TIMED_WAITNG

无需等待其他线程显示地唤醒，在一定的时间之后会被系统自动唤醒

调用如下方法会使线程进入超时等待状态：

- `Thread.sleep(long millis)`：使当前线程睡眠指定时间；
- `Object.wait(long timeout)`：线程休眠指定时间，等待期间可以通过`notify()`/`notifyAll()`唤醒；
- `Thread.join(long millis)`：等待当前线程最多执行 millis 毫秒，如果 millis 为 0，则会一直执行；
- `LockSupport.parkNanos(long nanos)`： 除非获得调用许可，否则禁用当前线程进行线程调度指定时间；
- `LockSupport.parkUntil(long deadline)`：同上，也是禁止线程进行调度指定时间；

### 死亡 TERMINATED

终止状态，线程执行完毕

## 线程中断

一个线程执行完毕之后会自动结束，如果在运行过程中罚睡异常也会提前结束。



- 如果该线程正在执行低级别的可中断方法（如`Thread.sleep()`、`Thread.join()`或`Object.wait()`），则会解除阻塞并**抛出`InterruptedException`异常**。
- 否则`Thread.interrupt()`仅设置线程的中断状态，此时在run()调用 interrupted() 方法来判断线程是否处于中断状态

```java
public class Test implements Runnable{

    @Override
    public void run() {
       try{
           System.out.println("hello");
           Thread.sleep(2000);
           System.out.println(Thread.currentThread().getName());
       } catch (InterruptedException e) {
           throw new RuntimeException(e);
       }
    }

    public static void main(String[] args) {
        Thread thread = new Thread(new Test(), "a");
        thread.start();
        thread.interrupt();
        System.out.println("run");
    }
}

```

## 线程之间的协作

当多个线程可以一起工作去解决某个问题时，如果某些部分必须在其它部分之前完成，那么就需要对线程进行协调。

### join方法

等待这个线程执行完才会轮到后续线程得到 cpu 的执行权

```java
Test test = new Test();
        Test test1 = new Test();
        Thread thread = new Thread(test, "线程1");
        Thread thread1 = new Thread(test1, "线程2");

        thread.start();
        try {
            thread.join(100);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        thread1.start();
```



### wait方法 notify方法

调用 wait() 使得线程等待某个条件满足，线程在等待时会被挂起，当其他线程的运行使得这个条件满足时，其它线程会调用 notify() 或者 notifyAll() 来唤醒挂起的线程。

它们都属于 Object 的一部分，而不属于 Thread。

只能用在同步方法或者同步控制块中使用，否则会在运行时抛出 IllegalMonitorStateExeception。

使用 wait() 挂起期间，线程会释放锁。这是因为，如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 notify() 或者 notifyAll() 来唤醒挂起的线程，造成死锁。

```java
public class Test {

    public synchronized void before(){
        System.out.println("before");
        notifyAll();
    }
    public synchronized void after(){
        try {
            wait();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println("after");
    }

    public static void main(String[] args) {
        Test test = new Test();
        new Thread(test::after).start();
        new Thread(test::before).start();
    }
}

before
after
```

上述的代码，正常来说会在after()进行阻塞等待，通过多线程notifyAll()方法唤醒阻塞线程从而避免死锁。

**sleep和wait方法关系**

**共同点**：两者都可以暂停线程的执行。

**区别**：

- **`sleep()` 方法没有释放锁，而 `wait()` 方法释放了锁** 。
- `wait()` 通常被用于线程间交互/通信，`sleep()`通常被用于暂停执行。
- `wait()` 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 `notify()`或者 `notifyAll()` 方法。`sleep()`方法执行完成后，线程会自动苏醒，或者也可以使用 `wait(long timeout)` 超时后线程会自动苏醒。
- `sleep()` 是 `Thread` 类的静态本地方法，`wait()` 则是 `Object` 类的本地方法

### await signal

java.util.concurrent 类库中提供了 Condition 类来实现线程之间的协调，可以在 Condition 上调用 await() 方法使线程等待，其它线程调用 signal() 或 signalAll() 方法唤醒等待的线程。相比于 wait() 这种等待方式，await() 可以指定等待的条件，因此更加灵活。

使用 Lock 来获取一个 Condition 对象。

```java
public class Test {
    ReentrantLock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    public void before(){
        lock.lock();
        try{
            System.out.println("before");
            condition.signalAll();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public void after(){
        lock.lock();
        try{
            condition.await();
            System.out.println("after");

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }

    }

    public static void main(String[] args) {
        Test test = new Test();
        new Thread(test::after).start();
        new Thread(test::before).start();

    }
}
before
after
```

![](thread.png)
