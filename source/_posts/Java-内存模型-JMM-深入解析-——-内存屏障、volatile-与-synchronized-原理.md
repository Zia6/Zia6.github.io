---
title: Java 内存模型 (JMM) 深入解析 —— 内存屏障、volatile 与 synchronized 原理
date: 2025-03-18 14:31:13
tags:
catagories: JUC

---

# Java 内存模型 (JMM) 深入解析 —— 内存屏障、volatile 与 synchronized 原理

> 从 Java 内存模型出发，详细剖析 volatile 和 synchronized 如何通过内存屏障保障可见性与有序性，彻底掌握 CPU 与编译器优化下的并发底层原理。

---

## 目录

- [1. 什么是 JMM？](#1-什么是-jmm)
- [2. 重排序问题简述](#2-重排序问题简述)
- [3. 内存屏障概念](#3-内存屏障概念)
  - [3.1 内存屏障分类](#31-内存屏障分类)
- [4. volatile 与内存屏障](#4-volatile-与内存屏障)
  - [4.1 内存语义](#41-内存语义)
  - [4.2 JVM 对 volatile 插入的屏障](#42-jvm-对-volatile-插入的屏障)
  - [4.3 示例分析](#43-示例分析)
- [5. synchronized 与内存屏障](#5-synchronized-与内存屏障)
  - [5.1 内存语义](#51-内存语义)
  - [5.2 字节码 monitorenter/monitorexit 对应屏障](#52-字节码-monitorenter-monitorexit-对应屏障)
  - [5.3 示例分析](#53-示例分析)
- [6. 内存屏障总结](#6-内存屏障总结)
- [7. volatile 与 synchronized 屏障对比](#7-volatile-与-synchronized-屏障对比)
- [8. happens-before 规则详解](#8-happens-before-规则详解)

---

## 1. 什么是 JMM？

Java 内存模型（JMM, Java Memory Model）定义了 **多线程之间可见性、有序性、原子性** 的规则，主要解决 CPU 缓存一致性、编译器优化与线程通信问题。

JMM 并不直接对应物理硬件，而是一组抽象规范，其核心：

- **主内存**：共享变量存储位置。
- **工作内存**：每个线程独享的变量副本。
- **内存交互操作**：load、store、read、write 指令。

---

## 2. 重排序问题简述

现代 CPU 和编译器为了提升性能，常做如下优化：

- **指令重排序**：代码指令执行顺序不同于源代码顺序。
- **缓存一致性问题**：每个 CPU 核心有 L1/L2 缓存，写入主内存不及时，导致线程间数据不一致。

JMM 保证：

1. 单线程内 **as-if-serial** 语义，重排序不影响单线程结果。
2. 多线程通过 **happens-before** 关系保证关键指令顺序。

---

## 3. 内存屏障概念

内存屏障（Memory Barrier / Fence）是 CPU 提供的指令，用来防止特定的指令重排序，或强制刷新缓存，保证可见性和有序性。

### 3.1 内存屏障分类

| 屏障类型            | 作用                                                                           |
|-----------------|-------------------------------------------------------------------------------|
| LoadLoadBarrier  | 屏障前的所有 **读** 操作完成，才能执行后续读                                  |
| StoreStoreBarrier| 屏障前的所有 **写** 操作完成，才能执行后续写                                  |
| LoadStoreBarrier | 屏障前的所有 **读** 操作完成，才能执行后续写                                  |
| StoreLoadBarrier | 屏障前所有 **写** 操作完成，并刷新到主存，后续读操作需重新加载主存            |

---

## 4. volatile 与内存屏障

`volatile` 是 JMM 关键字，用于保证变量的 **可见性** 和 **有序性**，但不保证原子性。

### 4.1 内存语义

1. **可见性**：对一个 volatile 变量写入，所有线程立刻可见。
2. **禁止指令重排序**：volatile 前后的指令不会被重排。

### 4.2 JVM 对 volatile 插入的屏障

JVM 在 volatile 读/写指令插入特定屏障：

| 操作        | 内存屏障                                           |
|------------|--------------------------------------------------|
| volatile 写 | **StoreStoreBarrier + StoreLoadBarrier** ：确保写操作对其他线程立即可见 |
| volatile 读 | **LoadLoadBarrier + LoadStoreBarrier** ：确保读操作不会被提前             |

HotSpot 中对应：

- volatile 写 → `lock addl` 指令 + StoreLoad 屏障（x86 上 lock 指令本身具备内存屏障效果）。
- volatile 读 → `mov` 普通加载，配合 CPU 硬件层面缓存一致性协议（如 MESI 协议）。

### 4.3 示例分析

```java
class VolatileExample {
    private volatile boolean flag = false;

    public void writer() {
        flag = true; // 写：StoreStore + StoreLoad 屏障
    }

    public void reader() {
        if (flag) {  // 读：LoadLoad + LoadStore 屏障
            // 确保看到最新 flag
        }
    }
}
```

**保证**：

1. `writer()` 方法对 flag 的写操作对其他线程立刻可见。
2. `reader()` 方法读取 flag 时，flag 前后的指令不会重排序。


## 5. synchronized 与内存屏障

`synchronized` 由 JVM 层实现，依靠 **Monitor 锁** 机制，同时隐式使用内存屏障，保证可见性与有序性。

### 5.1 内存语义

1. **可见性**：线程获取锁时，必须刷新工作内存中共享变量，获取主内存中最新值。
2. **有序性**：解锁前，必须将对共享变量的修改刷新到主内存，且解锁过程与之前操作不会发生重排序。

### 5.2 字节码 monitorenter/monitorexit 对应屏障

`synchronized` 编译后会生成：

- `monitorenter` → 获取锁
- `monitorexit` → 释放锁

HotSpot 虚拟机在这两个指令前后隐式插入屏障：

| 操作        | 内存屏障                                           |
|------------|--------------------------------------------------|
| monitorenter | **StoreStoreBarrier + LoadLoadBarrier + LoadStoreBarrier**：解锁前刷新主存，禁止重排 |
| monitorexit  | **StoreStoreBarrier** ：释放锁后，写入主存可见 |

具体行为：

- **monitorenter 前**：先清空工作内存，保证后续读取最新主内存值。
- **monitorexit 后**：将工作内存中共享变量的修改刷新到主存。

### 5.3 示例分析

```java
public class SyncExample {
    private int count = 0;

    public synchronized void increment() {
        count++; // monitorenter + 屏障
    }
}
```

执行过程：

1. **monitorenter** 之前，清空工作内存中 count，保证读取最新主存值。
2. count++ 操作完成后，在 **monitorexit** 之前，强制将 count 写回主存。
3. 其他线程获取锁后，读取到的是最新 count 值。

---

## 6. 内存屏障总结

| 关键字         | 内存屏障插入点                                   | 屏障类型                                           | 保证效果                                     |
|--------------|-----------------------------------------------|--------------------------------------------------|--------------------------------------------|
| volatile 写   | 写操作前后                                      | StoreStoreBarrier + StoreLoadBarrier             | 写入对其他线程立即可见，禁止重排序           |
| volatile 读   | 读操作前后                                      | LoadLoadBarrier + LoadStoreBarrier               | 保证读取到主内存最新值，禁止读重排序        |
| synchronized | monitorenter & monitorexit 处                   | StoreStoreBarrier + LoadLoadBarrier + LoadStoreBarrier | 保证进入锁后读取最新值，退出锁前刷新主存    |


**volatile 和 synchronized 虽实现机制不同，但最终都借助内存屏障确保多线程的可见性和有序性，关键区别在于是否保证原子性。**


## 7. volatile 与 synchronized 屏障对比

| 特性           | volatile                                    | synchronized                                 |
|--------------|---------------------------------------------|---------------------------------------------|
| 可见性        | 保证所有线程立即可见                          | 保证线程释放锁前刷新主存，获取锁时读取最新值     |
| 有序性        | 防止指令重排序，特别是读写相关指令              | 锁释放前后指令严格禁止重排                     |
| 原子性        | 不保证                                       | 保证（临界区操作整体原子）                    |
| 内存屏障位置   | 读写操作前后均插入屏障                          | monitorenter、monitorexit 处插入屏障         |
| 屏障种类      | 读：LoadLoad + LoadStore<br>写：StoreStore + StoreLoad | StoreStore + LoadLoad + LoadStore          |
| 适用场景      | 简单状态标志、状态通知                         | 复杂原子性操作、需要锁保护的临界区               |

### 📌 核心区别总结

1. **volatile 更轻量**，适用于变量状态标志切换，读写频繁但无复合操作。
2. **synchronized 更强大**，不仅保证可见性、有序性，还提供原子性，适用于完整临界区保护。
3. 屏障位置不同：volatile 细粒度作用在变量读写，synchronized 作用于方法/代码块整体。
4. **底层实现区别**：volatile 依赖 CPU 硬件层 Cache 一致性 + 屏障，synchronized 借助 JVM Monitor 结构控制锁获取与释放。


## 8. happens-before 规则详解

Java 内存模型通过 **happens-before 规则**，规定了操作之间的可见性与有序性保障。

简单来说：

> **前一个操作的结果对后续操作可见，并且前一个操作的执行顺序在后一个操作之前。**

### 8.1 happens-before 主要规则

| 规则编号 | 规则内容                                                       | 说明                                                         |
|-------|----------------------------------------------------------|------------------------------------------------------------|
| 1     | **程序顺序规则**                                               | 单线程内，按照代码顺序执行                                   |
| 2     | **锁规则**                                                   | 一个线程解锁，接下来获取该锁的线程能看到前线程释放锁时的结果 |
| 3     | **volatile 变量规则**                                         | 对 volatile 变量的写，先行发生于后续对同一个变量的读         |
| 4     | **传递性规则**                                                | A happens-before B，B happens-before C，则 A happens-before C |
| 5     | **线程启动规则**                                              | Thread.start() 先行发生于该线程执行的每一步                  |
| 6     | **线程终结规则**                                              | Thread.join() 先行发生于主线程继续                          |
| 7     | **线程中断规则**                                              | 调用线程 interrupt() 先行发生于目标线程检测到中断            |
| 8     | **对象终结规则**                                              | 对象的构造完成，其他线程能看到这个对象引用                    |

---

### 8.2 结合 volatile 和 synchronized 理解

#### synchronized 与 happens-before

- **释放锁 → 获取锁**，构成 happens-before 关系。
- 保证同步块内修改的变量对之后持有锁的线程可见。

#### volatile 与 happens-before

- **写 volatile → 读 volatile**，构成 happens-before 关系。
- 保证写线程对 volatile 变量的更新对之后读线程立即可见。

---

### 8.3 示例

```java
volatile boolean flag = false;
int data = 0;

Thread 1:                   Thread 2:
data = 42;                 if (flag) {
flag = true;                   System.out.println(data);
}                          }
```

**解析**：

- Thread 1 中 `flag = true` happens-before Thread 2 中 `if (flag)` 读取。
- 所以，Thread 2 若读到 flag 为 true，必然也能看到 data = 42。

---

## 📌 总结

happens-before 规则是 JMM 保证并发可见性、有序性的核心理论支撑，

- synchronized 和 volatile 的内存屏障实现，都是为了实现 **特定的 happens-before 关系**。
- 理解 happens-before 有助于正确使用锁、volatile 和设计线程安全程序。
