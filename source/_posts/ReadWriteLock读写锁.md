---
layout: java锁
title: ReadWriteLock读写锁
date: 2020-06-18 16:10:20
tags: java锁
categories: juc
---

# ReadWriteLock读写锁

为了提高性能，Java 提供了读写锁，在读的地方使用读锁，在写的地方使用写锁，灵活控制，如果没有写锁的情况下，读是无阻塞的,在一定程度上提高了程序的执行效率。读写锁分为读锁和写锁，多个读锁不互斥，读锁与写锁互斥，这是由 jvm 自己控制的，你只要上好相应的锁即可。

## **读锁**

如果你的代码只读数据，可以很多人同时读，但不能同时写，那就上读锁

## **写锁**

如果你的代码修改数据，只能有一个人在写，且不能同时读取，那就上写锁。总之，读的时候上读锁，写的时候上写锁！

Java 中 读 写 锁 有 个 接 口 java.util.concurrent.locks.ReadWriteLock ， 也 有 具 体 的 实 现ReentrantReadWriteLock。

