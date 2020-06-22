---
layout: java锁
title: ReentrantLock
date: 2020-06-18 15:53:17
tags: java锁
categories: juc
---

# **ReentrantLock**

ReentantLock 继承接口 Lock 并实现了接口中定义的方法，他是一种可重入锁，除了能完成 synchronized 所能完成的所有工作外，还提供了诸如可响应中断锁、可轮询锁请求、定时锁等避免多线程死锁的方法。

## **Lock** **接口的主要方法**

1. void lock(): 执行此方法时, 如果锁处于空闲状态, 当前线程将获取到锁. 相反, 如果锁已经被其他线程持有, 将禁用当前线程, 直到当前线程获取到锁.

2. boolean tryLock()：如果锁可用, 则获取锁, 并立即返回 true, 否则返回 false. 该方法和lock()的区别在于, tryLock()只是"试图"获取锁, 如果锁不可用, 不会导致当前线程被禁用, 当前线程仍然继续往下执行代码. 而 lock()方法则是一定要获取到锁, 如果锁不可用, 就一直等待, 在未获得锁之前,当前线程并不继续向下执行. 

3. void unlock()：执行此方法时, 当前线程将释放持有的锁. 锁只能由持有者释放, 如果线程并不持有锁, 却执行该方法, 可能导致异常的发生.

4. Condition newCondition()：条件对象，获取等待通知组件。该组件和当前的锁绑定，当前线程只有获取了锁，才能调用该组件的 await()方法，而调用后，当前线程将缩放锁。

5. getHoldCount() ：查询当前线程保持此锁的次数，也就是执行此线程执行 lock 方法的次数。

6. getQueueLength（）：返回正等待获取此锁的线程估计数，比如启动 10 个线程，1 个线程获得锁，此时返回的是 9

7. getWaitQueueLength：（Condition condition）返回等待与此锁相关的给定条件的线程估计数。比如 10 个线程，用同一个 condition 对象，并且此时这 10 个线程都执行了condition 对象的 await 方法，那么此时执行此方法返回 10

8. hasWaiters(Condition condition)：查询是否有线程等待与此锁有关的给定条件

(condition)，对于指定 contidion 对象，有多少线程执行了 condition.await 方法

9. hasQueuedThread(Thread thread)：查询给定线程是否等待获取此锁

10. hasQueuedThreads()：是否有线程等待此锁

11. isFair()：该锁是否公平锁

12. isHeldByCurrentThread()： 当前线程是否保持锁锁定，线程的执行 lock 方法的前后分别是 false 和 true

13. isLock()：此锁是否有任意线程占用

14. lockInterruptibly（）：如果当前线程未被中断，获取锁

15. tryLock（）：尝试获得锁，仅在调用时锁未被线程占用，获得锁

16. tryLock(long timeout TimeUnit unit)：如果锁在给定等待时间内没有被另一个线程保持，则获取该锁。

## **非公平锁**

JVM 按随机、就近原则分配锁的机制则称为不公平锁，ReentrantLock 在构造函数中提供了是否公平锁的初始化方式，默认为非公平锁。非公平锁实际执行的效率要远远超出公平锁，除非程序有特殊需要，否则最常用非公平锁的分配机制。

## **公平锁**

公平锁指的是锁的分配机制是公平的，通常先对锁提出获取请求的线程会先被分配到锁，ReentrantLock 在构造函数中提供了是否公平锁的初始化方式来定义公平锁。

## **ReentrantLock** **与** **synchronized**

1. ReentrantLock 通过方法 lock()与 unlock()来进行加锁与解锁操作，与 synchronized 会 被 JVM 自动解锁机制不同，ReentrantLock 加锁后需要手动进行解锁。为了避免程序出现异常而无法正常解锁的情况，使用 ReentrantLock 必须在 finally 控制块中进行解锁操作。

2. ReentrantLock 相比 synchronized 的优势是可中断、公平锁、多个锁。这种情况下需要使用 ReentrantLock。

### **ReentrantLock** **实现**

```java
public class MyService {
private Lock lock = new ReentrantLock();
//Lock lock=new ReentrantLock(true);//公平锁
//Lock lock=new ReentrantLock(false);//非公平锁
private Condition condition=lock.newCondition();//创建 Condition
public void testMethod() {
try {
lock.lock();//lock 加锁
//1：wait 方法等待：
//System.out.println("开始 wait");
condition.await();
//通过创建 Condition 对象来使线程 wait，必须先执行 lock.lock 方法获得锁
//:2：signal 方法唤醒
condition.signal();//condition 对象的 signal 方法可以唤醒 wait 线程
for (int i = 0; i < 5; i++) {
System.out.println("ThreadName=" + Thread.currentThread().getName()+ (" " + (i + 1)));
}
} catch (InterruptedException e) {
e.printStackTrace();
}
finally
121623125152125125
{
lock.unlock();
} } }
```

### **Condition** **类和** **Object** **类锁方法区别区别**

1. Condition 类的 awiat 方法和 Object 类的 wait 方法等效

2. Condition 类的 signal 方法和 Object 类的 notify 方法等效

3. Condition 类的 signalAll 方法和 Object 类的 notifyAll 方法等效

4. ReentrantLock 类可以唤醒指定条件的线程，而 object 的唤醒是随机的

### **tryLock** **和** **lock** **和** **lockInterruptibly** **的区别**

1. tryLock 能获得锁就返回 true，不能就立即返回 false，tryLock(long timeout,TimeUnit unit)，可以增加时间限制，如果超过该时间段还没获得锁，返回 false

2. lock 能获得锁就返回 true，不能的话一直等待获得锁

3. lock 和 lockInterruptibly，如果两个线程分别执行这两个方法，但此时中断这两个线程，lock 不会抛出异常，而 lockInterruptibly 会抛出异常。