---
title: synchronized详解
date: 2024-08-11 11:10:02
tags:
---

# Synchronized的使用

注意点：

- 一把锁只能同时被一个线程获取，没有获得锁的线程只能等待；
- 每个实例都对应有自己的一把锁(this),不同实例之间互不影响；例外：锁对象是***.class**以及synchronized修饰的是**static方法**的时候，**所有对象公用同一把锁**
- synchronized修饰的方法，无论方法**正常执行完毕**还是**抛出异常**，都会释放锁

### 对象锁

包括方法锁（默认锁为this）和同步代码块锁（自己指定锁对象）

### 类锁

指synchronize修饰静态的方法或指定锁对象为Class对象

# 原理分析

## 加锁和释放锁的原理

```java
public void method1(){
	synchronized (object) {

    }
    sout("hello")
}
```

经过jvm解析得到是

![image-20241211112413784](image-20241211112413784-17338874548381.png)

`Monitorenter`和`Monitorexit`指令，会让对象在执行，使其锁计数器加1或者减1。每一个对象在同一时间只与一个monitor(锁)相关联，而一个monitor在同一时间只能被一个线程获得，一个对象在尝试获得与这个对象相关联的Monitor锁的所有权的时候，

`monitorenter`指令会发生如下3中情况之一：

- monitor计数器为0，意味着目前还没有被获得，那这个线程会把自身的id，交给锁，然后获得锁，把锁计数器+1，一旦+1，别的线程再想获取，就需要等待
- 如果这个monitor已经拿到了这个锁的所有权，又重入了这把锁，那锁计数器就会累加，变成2，并且随着重入的次数，会一直累加
- 这把锁已经被别的线程获取了，等待锁释放

`monitorexit指令`：释放对于monitor的所有权，释放过程很简单，就是讲monitor的计数器减1，如果减完以后，计数器不是0，则代表刚才是重入进来的，当前线程还继续持有这把锁的所有权，如果计数器变成0，则代表当前线程不再拥有该monitor的所有权，即释放锁。

## 可重入原理：加锁次数计数器

**可重入锁**：又名递归锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。
