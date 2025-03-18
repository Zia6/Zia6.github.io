---
title: JUC-CompleteFuture总结
date: 2025-03-18 09:42:02
tags:
catagories: JUC
---
# CompletableFuture 完全指南

> 从 Runnable 到 CompletableFuture，全面理解 Java 并发异步编排。

## 前言
在日常开发中，我们经常会遇到需要开启子线程执行任务的场景，比如网络请求、数据库操作、复杂计算等等。Java 为我们提供了多种方式来实现异步操作，从最基础的 Runnable、Callable 到 Future，但这些传统方式在实际使用中存在很多局限，尤其是在面对复杂的异步回调、异常处理、任务组合时。
本文将带你从最基础的线程模型讲起，一步步剖析为什么 CompletableFuture 会诞生，它到底帮我们解决了哪些问题，以及如何用它优雅地实现高效的异步编排。

---
## 目录

- [2. Runnable、Callable 与 Future：基础异步模型](#2-runnablecallable-与-future基础异步模型)
  - [2.1 Runnable 简介](#21-runnable-简介)
  - [2.2 Callable 与 Future 简介](#22-callable-与-future-简介)
  - [2.3 Runnable & Callable & Future 的局限](#23-runnable--callable--future-的局限)
- [3. CompletableFuture 为什么出现？](#3-completablefuture-为什么出现)
- [4. CompletableFuture 核心用法](#4-completablefuture-核心用法)
  - [4.1 创建异步任务](#41-创建异步任务)
  - [4.2 任务链式编排](#42-任务链式编排)
  - [4.3 多任务组合](#43-多任务组合)
  - [4.4 异常处理](#44-异常处理)
- [5. CompletableFuture 底层实现简单了解](#5-completablefuture-底层实现简单了解)
- [6. CompletableFuture 实战案例](#6-completablefuture-实战案例)
  - [6.1 电商多平台比价](#61-电商多平台比价)
  - [6.2 多服务数据聚合](#62-多服务数据聚合)
- [7. 常见坑与面试题总结](#7-常见坑与面试题总结)
- [8. 总结](#8-总结)




## 2. Runnable、Callable 与 Future：基础异步模型

### 2.1 Runnable 简介

在 Java 并发编程的早期，最基础的异步执行方式是通过实现 `Runnable` 接口：

```java
Thread thread = new Thread(() -> {
    // 子线程任务
    System.out.println("Hello from Runnable");
});
thread.start();
```

`Runnable` 接口非常简单，只有一个 `run()` 方法，特点是 **没有返回值**，也**无法抛出受检异常**。

适用于简单的异步任务，但有两个缺点：

- 无法获取任务执行结果
- 异常处理不便

---

### 2.2 Callable 与 Future 简介

为了解决 `Runnable` 不能返回结果的问题，Java 在 **JDK 1.5** 引入了 `Callable` 接口，并配套设计了 `Future` 接口。

**Callable 的特点**：

- 方法是 `call()`，可以有返回值。
- 可以抛出异常。

**Future 的作用**：

- 接收异步任务的结果。
- 提供查询、取消任务、检查是否完成的方法。

典型用法如下：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

Callable<Integer> task = () -> {
    // 计算任务
    return 42;
};

Future<Integer> future = executor.submit(task);

try {
    Integer result = future.get(); // 阻塞等待结果
    System.out.println("Result: " + result);
} catch (Exception e) {
    e.printStackTrace();
}
```

**Future 常用方法**：

| 方法 | 作用 |
|---|---|
| `get()` | 获取结果，阻塞当前线程 |
| `get(timeout, unit)` | 带超时等待 |
| `isDone()` | 判断任务是否完成 |
| `cancel()` | 取消任务 |
| `isCancelled()` | 是否被取消 |

---

### 2.3 Runnable & Callable & Future 的局限

虽然 `Callable + Future` 组合可以解决返回值、异常的问题，但在实际开发中仍然有以下缺点：

1. **get() 方法阻塞**  
   调用 `future.get()` 会阻塞主线程直到子任务完成，不够灵活。

2. **任务编排困难**  
   多个异步任务如果存在依赖，需要手动管理，**缺少链式调用支持**。

3. **异常处理繁琐**  
   每次调用 `get()` 都需要 try-catch 显式处理异常。

4. **组合任务困难**  
   无法方便地合并多个异步任务结果，缺乏统一 API。

5. **回调机制缺失**  
   无法在异步任务完成后自动触发下一步操作，只能手动轮询或阻塞等待。

---

## 3. CompletableFuture 为什么出现？

`Future` 解决了异步返回值的问题，但面对复杂业务场景，它的缺陷逐渐显现：

- **无法实现异步任务链式编排**
- **不支持非阻塞地获取结果**
- **缺乏优雅的异常处理机制**
- **多任务并发执行后结果整合困难**

为了解决这些问题，Java 在 **JDK 1.8** 引入了 **CompletableFuture**，它不仅实现了 `Future` 接口，还提供了丰富的功能：

- 支持链式调用，轻松实现任务依赖关系。
- 支持非阻塞获取结果。
- 提供优雅的异常处理方法。
- 内置多任务组合、合并结果功能。

接下来我们详细了解 `CompletableFuture` 的用法及其背后的设计理念。

---

## 4. CompletableFuture 核心用法

### 4.1 创建异步任务

`CompletableFuture` 提供两种创建异步任务的方法：

- `runAsync()` ：无返回值，类似 `Runnable`
- `supplyAsync()` ：有返回值，类似 `Callable`

```java
// 无返回值
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
    System.out.println("Run async task");
});

// 有返回值
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 123);
```

你也可以指定自定义线程池：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> 10, executor);
```

---

### 4.2 任务链式编排

`CompletableFuture` 支持链式调用，轻松实现异步任务依赖关系：

- `thenApply()` ：对结果进行转换
- `thenAccept()` ：消费结果，无返回
- `thenRun()` ：不依赖结果，仅执行后续动作

```java
CompletableFuture.supplyAsync(() -> 5)
    .thenApply(result -> result * 2)
    .thenAccept(result -> System.out.println("Result: " + result))
    .thenRun(() -> System.out.println("Task finished"));
```

#### 组合依赖

- `thenCompose()` ：任务串联，前后依赖

```java
CompletableFuture.supplyAsync(() -> "Hello")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " World"))
    .thenAccept(System.out::println);
```

- `thenCombine()` ：两个独立任务结果合并

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 20);

future1.thenCombine(future2, Integer::sum)
       .thenAccept(result -> System.out.println("Sum: " + result));
```

---

### 4.3 多任务组合

`CompletableFuture` 提供两个重要方法来处理多任务：

- `allOf()` ：等待所有任务完成
- `anyOf()` ：任意一个完成即可继续

```java
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2);
all.thenRun(() -> System.out.println("All tasks done"));

CompletableFuture<Object> any = CompletableFuture.anyOf(future1, future2);
any.thenAccept(result -> System.out.println("First finished: " + result));
```

---

### 4.4 异常处理

异常在异步编排中非常重要，`CompletableFuture` 提供了两种优雅的异常处理方式：

- `exceptionally()` ：发生异常时提供默认值

```java
CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Error");
    return 1;
}).exceptionally(ex -> {
    System.out.println("Exception: " + ex.getMessage());
    return -1;
}).thenAccept(System.out::println);
```

- `handle()` ：可以同时处理正常结果和异常

```java
CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Fail");
    return 100;
}).handle((result, ex) -> {
    if (ex != null) {
        System.out.println("Handle exception: " + ex.getMessage());
        return -1;
    } else {
        return result;
    }
}).thenAccept(System.out::println);
```

---

## 5. CompletableFuture 底层实现简单了解

`CompletableFuture` 底层依托于 **ForkJoinPool** 线程池实现异步任务调度。默认情况下，使用的是 `ForkJoinPool.commonPool()`，它内部采用 **工作窃取算法** 提高线程利用率。

核心设计要点：

- 每个异步任务会被包装成一个 `ForkJoinTask`，提交到线程池中执行。
- 任务之间的回调关系，依靠 **Completion 队列** 实现，保证每个阶段完成后能够自动触发下一个动作。

核心数据结构：

- **result**：保存任务的返回值或异常
- **stack**：保存回调链

你也可以通过指定自定义线程池，避免共用 `ForkJoinPool` 带来的影响，特别是在 Web 容器中推荐自定义线程池。

---

## 6. CompletableFuture 实战案例

### 6.1 电商多平台比价

需求：查询不同电商平台同一商品价格，汇总最低价。

```java
List<String> platforms = Arrays.asList("JD", "Taobao", "PDD");

List<CompletableFuture<Double>> priceFutures = platforms.stream()
    .map(platform -> CompletableFuture.supplyAsync(() -> queryPrice(platform)))
    .collect(Collectors.toList());

CompletableFuture.allOf(priceFutures.toArray(new CompletableFuture[0]))
    .thenRun(() -> {
        List<Double> prices = priceFutures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        System.out.println("All prices: " + prices);
    });

private static double queryPrice(String platform) {
    // 模拟调用
    try { Thread.sleep(1000); } catch (InterruptedException e) {}
    return Math.random() * 100;
}
```

---

### 6.2 多服务数据聚合

假设需要聚合用户信息、账户信息、积分信息：

```java
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> queryUser());
CompletableFuture<Account> accountFuture = CompletableFuture.supplyAsync(() -> queryAccount());
CompletableFuture<Score> scoreFuture = CompletableFuture.supplyAsync(() -> queryScore());

CompletableFuture.allOf(userFuture, accountFuture, scoreFuture)
    .thenRun(() -> {
        User user = userFuture.join();
        Account account = accountFuture.join();
        Score score = scoreFuture.join();
        // 合并结果
        System.out.println(user + " " + account + " " + score);
    });
```

---

## 7. 常见坑与面试题总结

### 7.1 get() 和 join() 有什么区别？

- `get()` 会抛出 checked 异常，需要显式处理。
- `join()` 会将异常包装成 `CompletionException`，无需显示捕获，推荐链式调用时使用。

### 7.2 默认线程池安全吗？

默认使用 `ForkJoinPool.commonPool()`，在高并发场景或 Web 应用中可能会导致线程被占满，**推荐自定义线程池**。

### 7.3 如何优雅处理异常？

优先使用 `handle()`，同时处理正常和异常；或者 `exceptionally()` 提供默认值，避免影响主线程。

### 7.4 面试高频问题

- CompletableFuture 的优点有哪些？
- thenCompose 和 thenCombine 区别？
- allOf 和 anyOf 使用场景？
- 异常处理机制有哪些？

---

## 8. 总结

从 Runnable、Callable + Future 到 CompletableFuture，Java 并发异步工具的发展，正是为了解决异步流程中**编排复杂、异常处理、结果整合**等问题。

掌握好 `CompletableFuture`，可以帮助我们写出高效、优雅、健壮的异步代码，真正解决开发中的异步回调地狱、线程阻塞问题。
