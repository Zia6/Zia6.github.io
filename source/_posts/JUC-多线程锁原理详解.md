---
title: JUC-多线程锁原理详解
date: 2025-03-18 10:20:03
tags:
catagories: JUC
---

# Java 多线程锁原理详解 —— 从字节码到 JVM 实现全解

Java 并发编程离不开对“锁”机制的理解与掌握。无论是最基础的 `synchronized`，还是功能丰富的 `ReentrantLock`，亦或是高效提升读操作性能的 `ReadWriteLock`，它们背后蕴含的实现原理和适用场景各有千秋。

本文将从为什么需要锁讲起，结合 **字节码**、**JVM 实现**，一步步剖析各类锁的底层机制，帮助你彻底掌握多线程锁的原理。

---

## 目录

- [1. 为什么需要锁？](#1-为什么需要锁)
- [2. 可重入锁是什么？](#2-可重入锁是什么)
- [3. synchronized 详解](#3-synchronized-详解)
  - [3.1 基本用法](#31-基本用法)
  - [3.2 字节码与 JVM 实现](#32-字节码与-jvm-实现)
  - [3.3 优缺点总结](#33-优缺点总结)
- [4. wait/notify 机制详解](#4-waitnotify-机制详解)
- [5. ReentrantLock 详解](#5-reentrantlock-详解)
  - [5.1 基本用法](#51-基本用法)
  - [5.2 公平锁与非公平锁](#52-公平锁与非公平锁)
  - [5.3 tryLock 与中断](#53-trylock-与中断)
  - [5.4 Condition 精准唤醒](#54-condition-精准唤醒)
  - [5.5 Condition vs wait/notify 对比](#55-condition-vs-waitnotify-对比)
- [6. 读写锁 ReadWriteLock](#6-读写锁-readwritelock)
  - [6.1 原理与用法](#61-原理与用法)
  - [6.2 适用场景](#62-适用场景)
- [7. StampedLock 简介](#7-stampedlock-简介)
- [8. LockSupport 简介](#8-locksupport-简介)
- [9. 多线程锁大对比表](#9-多线程锁大对比表)
- [10. 实战案例选型指导](#10-实战案例选型指导)
- [11. 总结](#11-总结)

---

## 1. 为什么需要锁？

在多线程并发环境下，多个线程可能会同时访问同一个共享变量，导致数据不一致的问题。

**示例问题**：

```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

多个线程同时调用 `increment()`，`count++` 不是原子操作，可能导致最终结果不正确。

**需要锁的原因**：

- **原子性**：保证多个操作作为一个整体，不被打断。
- **可见性**：确保一个线程修改的共享变量对其他线程可见。
- **有序性**：防止指令重排导致逻辑出错。

---

## 2. 可重入锁概念与意义

**什么是可重入锁？**

可重入锁（Reentrant Lock）指的是 **同一个线程在持有锁的情况下，可以再次获取该锁而不会死锁**。

### 2.1 举例说明

```java
public class ReentrantExample {
    public synchronized void methodA() {
        methodB();
    }

    public synchronized void methodB() {
        System.out.println("Inside methodB");
    }
}
```

`methodA()` 持有锁，调用 `methodB()` 时，依然可以获取同一把锁，不会死锁。

### 2.2 synchronized 和 ReentrantLock 都是可重入锁

- synchronized 由 JVM 维护锁计数
- ReentrantLock 内部通过 **state 计数器** 实现

---

## 3. synchronized 详解

### 3.1 基本用法

`synchronized` 是 Java 最基础的内置锁，语法简洁，适用于大部分简单互斥场景。

**用法：**

1. 修饰实例方法：锁对象实例 `this`
2. 修饰静态方法：锁 Class 对象
3. 修饰代码块：自定义锁对象

```java
public class SyncExample {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public void blockIncrement() {
        synchronized (this) {
            count++;
        }
    }
}
```

### 3.2 字节码与 JVM 实现

`synchronized` 由 JVM 层实现，核心指令：

- **monitorenter**：进入同步块，获取 Monitor 锁
- **monitorexit**：退出同步块，释放锁

反编译示例：

```bash
0: aload_0
1: dup
2: monitorenter
3: aload_0
4: dup
5: getfield #2 // count
8: iconst_1
9: iadd
10: putfield #2 // count++
13: monitorexit
```

JVM 底层使用 **对象头 Mark Word** 存储锁状态，锁会经历以下升级过程：

| 锁类型 | 特点 |
|---|---|
| 无锁 | 无竞争，无需加锁 |
| 偏向锁 | 单线程访问，减少 CAS 开销 |
| 轻量级锁 | 竞争轻微，自旋避免阻塞 |
| 重量级锁 | 竞争激烈，阻塞挂起线程 |

### 3.3 优缺点总结

| 优点 | 缺点 |
|----|----|
| 简单易用，JVM 层实现，异常自动释放锁 | 无法中断，无法公平，无精准唤醒 |

**适用场景**：简单互斥、无高并发需求的线程安全保护。

---

## 4. wait/notify 机制详解

`synchronized` 常搭配 **Object.wait()** / **Object.notify()** / **Object.notifyAll()** 实现线程通信。

### 4.1 基本原理

- `wait()`：释放锁，当前线程进入等待队列，等待被唤醒。
- `notify()`：随机唤醒等待队列中的一个线程。
- `notifyAll()`：唤醒所有等待线程。

必须在同步块内部调用，否则抛出 `IllegalMonitorStateException`。

### 4.2 示例

```java
public class WaitNotifyExample {
    private final Object lock = new Object();

    public void waitMethod() throws InterruptedException {
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName() + " waiting");
            lock.wait();
            System.out.println(Thread.currentThread().getName() + " resumed");
        }
    }

    public void notifyMethod() {
        synchronized (lock) {
            lock.notify();
        }
    }
}
```

### 4.3 特点

| 特性 | 描述 |
|---|---|
| 依赖锁 | 必须搭配 synchronized 使用 |
| 唤醒粒度 | 随机唤醒一个或全部，不精准 |
| 性能 | JVM 层实现，性能尚可 |

### 4.4 常见问题

- 必须在持有锁的同步块中调用，否则抛异常
- 可能产生“虚假唤醒”，推荐搭配 `while` 循环二次判断条件

## 5. ReentrantLock 详解

`ReentrantLock` 是 JDK 提供的可重入显式锁，功能上比 synchronized 更加灵活，尤其支持公平性、中断控制和多条件队列。

### 5.1 基本用法

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}
```

特点：

- 必须手动释放锁，推荐配合 try-finally 保证锁释放。

---

### 5.2 公平锁与非公平锁

```java
// 公平锁
ReentrantLock fairLock = new ReentrantLock(true);

// 非公平锁（默认）
ReentrantLock unfairLock = new ReentrantLock(false);
```

- **公平锁**：线程按照请求顺序排队，避免饥饿。
- **非公平锁**：允许当前线程插队，吞吐量更高。

实际开发中，**非公平锁更常用**，性能优于公平锁。

---

### 5.3 tryLock 与中断

**tryLock()**：尝试获取锁，获取不到立即返回，不阻塞。

```java
if (lock.tryLock()) {
    try {
        // 获得锁
    } finally {
        lock.unlock();
    }
} else {
    // 没获取到锁
}
```

**tryLock(long timeout, TimeUnit unit)**：等待指定时间，超时放弃。

```java
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    // 获得锁
}
```

**lockInterruptibly()**：可响应中断，避免死锁场景。

```java
try {
    lock.lockInterruptibly();
    // 临界区
} catch (InterruptedException e) {
    // 被中断
}
```

---

### 5.4 Condition 精准唤醒

ReentrantLock 支持多个 **Condition 条件队列**，实现更精确的线程通信。

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();

// 线程等待
lock.lock();
try {
    condition.await();
} finally {
    lock.unlock();
}

// 其他线程唤醒
lock.lock();
try {
    condition.signal(); // 唤醒一个线程
    // condition.signalAll(); // 唤醒所有等待线程
} finally {
    lock.unlock();
}
```

**优势**：

- 一个锁可以绑定多个 Condition 队列，实现**多条件精准唤醒**。
- 适用于复杂生产者-消费者模型、任务调度。

---

### 5.5 Condition vs wait/notify 对比

| 特性                         | Condition                              | wait/notify                      |
|----------------------------|----------------------------------------|----------------------------------|
| 所属锁                     | ReentrantLock                          | synchronized                     |
| 精准唤醒粒度               | 支持多个条件队列，精准唤醒               | 唤醒单个或所有线程，不可区分     |
| 是否需要手动释放锁          | 是，需 lock.lock()/unlock()            | 自动释放 synchronized 锁        |
| 是否支持中断               | 支持                                   | 支持                             |
| 是否支持公平性             | 可配合公平锁使用                       | 不支持                           |

**总结建议**：

- 简单同步场景：`synchronized` + `wait/notify` 即可。
- 多条件唤醒、复杂线程协作推荐 **ReentrantLock + Condition**，灵活性更高。

---
## 6. 读写锁 ReadWriteLock

### 6.1 原理与用法

`ReadWriteLock` 允许多个读线程并发读取，但写线程独占。典型实现是 `ReentrantReadWriteLock`。

**基本用法**：

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// 读线程
readLock.lock();
try {
    // 读取共享资源
} finally {
    readLock.unlock();
}

// 写线程
writeLock.lock();
try {
    // 修改共享资源
} finally {
    writeLock.unlock();
}
```

**特点**：

- **读锁可重入**，多个线程可同时持有。
- **写锁互斥**，写操作独占，防止数据冲突。
- **读写互斥**，写操作会阻塞所有读线程。

---

### 6.2 适用场景

适用于 **读多写少** 场景，如：

- 配置信息加载
- 缓存读取、偶尔更新
- 数据表查询为主，偶尔写入

**注意**：

- 如果写操作频繁，读写锁性能不如普通互斥锁。
- 读锁不可升级为写锁，容易引发死锁。

---

## 7. StampedLock 简介

`StampedLock` 是 JDK 8 引入的新型读写锁，支持 **悲观读**、**写锁** 和 **乐观读** 三种模式，性能优于 `ReadWriteLock`。

### 7.1 三种模式

1. **悲观读锁**：功能类似 ReadWriteLock 的读锁，支持多线程并发读取，写时互斥。
2. **写锁**：独占，和 ReentrantWriteLock 类似。
3. **乐观读锁**：无阻塞，线程获取版本戳后直接读取，适用于短时间读取且不介意数据略微不一致的场景。

### 7.2 使用示例

```java
StampedLock lock = new StampedLock();

// 乐观读
long stamp = lock.tryOptimisticRead();
int data = sharedData;
if (!lock.validate(stamp)) { // 数据修改则重新获取
    stamp = lock.readLock();
    try {
        data = sharedData;
    } finally {
        lock.unlockRead(stamp);
    }
}

// 写锁
long writeStamp = lock.writeLock();
try {
    sharedData++;
} finally {
    lock.unlockWrite(writeStamp);
}
```

### 7.3 适用场景

- 高并发读、多线程访问无强一致性要求
- 替代 ReentrantReadWriteLock 性能瓶颈

**注意**：不支持可重入、不可升级锁，使用需小心避免死锁。

---

## 8. LockSupport 简介

`LockSupport` 是 JUC 提供的底层线程阻塞/唤醒工具类，核心方法：

- `park()` —— 阻塞当前线程
- `unpark(Thread)` —— 唤醒指定线程

### 8.1 关键特性

- 无需持有锁即可阻塞线程
- 支持先 unpark 再 park，不丢失信号
- 不会抛出 InterruptedException，但会响应中断

### 8.2 使用示例

```java
Thread t = new Thread(() -> {
    System.out.println("Thread parking...");
    LockSupport.park();
    System.out.println("Thread resumed.");
});

t.start();
Thread.sleep(1000);
LockSupport.unpark(t);
```

### 8.3 典型应用

- AQS 队列实现核心
- 自定义锁、线程调度器底层工具

---
## 9. 多线程锁大对比表

| 特性                   | synchronized            | ReentrantLock           | ReadWriteLock             | StampedLock               | LockSupport           |
|----------------------|-------------------------|-------------------------|---------------------------|---------------------------|-----------------------|
| 是否可重入              | 是                      | 是                      | 读锁可重入                 | 否                        | N/A                   |
| 公平性支持              | 否                      | 支持                     | 支持                      | 否                        | N/A                   |
| 可中断性                | 否                      | 支持 lockInterruptibly()| 支持                      | 支持                      | 支持                  |
| 超时获取锁              | 否                      | 支持 tryLock()           | 支持                      | 支持                      | N/A                   |
| 精准唤醒                | 否                      | Condition 支持           | 支持                      | 不支持多条件队列          | 支持 unpark() 唤醒    |
| 实现层                  | JVM 层实现               | Java 层实现，基于 AQS    | Java 层实现，AQS 变种      | Java 层实现               | JVM + Unsafe 原语    |
| 性能                   | JDK1.6 后较好           | 灵活控制，高性能         | 适合读多写少               | 极高读性能，适合乐观读     | 高效低级别工具        |
| 适用场景               | 简单同步，锁粒度粗        | 中断/公平性/多条件需求    | 读多写少，互斥性要求高      | 读非常频繁、无需重入       | 自定义锁/线程调度核心    |

---

## 10. 实战案例选型指导

### 10.1 synchronized 推荐场景

- 简单同步逻辑，线程安全保证即可
- 无需考虑公平性、中断
- 例如：计数器累加器、单线程同步方法

### 10.2 ReentrantLock 推荐场景

- 需要公平锁策略（如限流队列）
- 需要响应中断避免死锁
- 需要多条件精准唤醒（多生产者多消费者模型）
- 例如：高并发抢票系统、线程池任务调度

### 10.3 ReadWriteLock 推荐场景

- 数据读取远多于写入
- 写线程操作较少、更新数据一致性要求强
- 例如：缓存系统、配置读取、规则引擎

### 10.4 StampedLock 推荐场景

- 高并发读，追求极致性能
- 对一致性容忍度高（支持乐观读）
- 例如：大规模只读数据分析、内存数据库

### 10.5 LockSupport 推荐场景

- 实现底层并发框架或自定义锁
- AQS 等队列同步器核心组件
- 例如：Semaphore、CountDownLatch、CyclicBarrier 实现

---

## 11. 总结

Java 并发锁体系丰富，理解它们的底层实现与适用场景，能帮助我们写出性能优异且线程安全的多线程程序。

- 简单场景优先 synchronized
- 有复杂控制需求选择 ReentrantLock
- 读多写少优先 ReadWriteLock 或 StampedLock
- 并发框架开发掌握 LockSupport
## 9. 总结

Java 多线程锁体系从 synchronized、ReentrantLock 到读写锁，逐步引入公平性、中断感知、读写分离、精准唤醒等特性，适应复杂并发场景需求。

掌握它们底层原理，有助于写出高性能且线程安全的并发程序。