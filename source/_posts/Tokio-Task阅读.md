---
title: Tokio-Task阅读
date: 2024-12-10 11:19:09
tags:
---
## 任务模块概述
任务模块负责管理运行时生成的任务，并为其他模块提供安全的 API 接口。每个任务通常存储在 `OwnedTasks` 或 `LocalOwnedTasks` 中，并通过引用计数（ref-count）来跟踪任务的活动引用。
### 任务引用类型
在运行时，任务可以通过不同的引用类型进行管理，主要有以下几种：

- **`OwnedTask`**：任务存储在 `OwnedTasks` 或 `LocalOwnedTasks` 中。
- **`JoinHandle`**：任务的句柄，允许访问任务的输出。
- **`Waker`**：每个任务可以有多个 `Waker` 引用，用于唤醒任务继续执行。
- **`Notified`**：用于追踪任务是否被通知。
- **`Unowned`**：用于任务不在运行时管理下的引用，通常用于阻塞任务或测试。

`Unowned` 类型会占用两个引用计数，而其他引用类型只占用一个，Unowned通过`std::mem::forget`函数保证了两个引用计数

### 任务状态

任务的状态通过一个原子 `usize` 类型的字段来表示，包含以下重要的标志位（bitfields）：

- **`RUNNING`**：表示任务是否正在运行或被取消，是任务的锁定标志。
- **`COMPLETE`**：表示任务是否完成并且资源已被释放。
- **`NOTIFIED`**：表示任务是否已经被通知。
- **`CANCELLED`**：标记任务是否应尽早取消。
- **`JOIN_INTEREST`**：指示是否有 `JoinHandle` 对应的任务。
- **`JOIN_WAKER`**：控制 `JoinHandle` 访问任务的 `waker` 字段。

### 任务字段访问规则

任务的各个字段在不同情况下有不同的访问权限和规则：

- **`state` 字段**：通过原子操作进行访问。
- **`OwnedTask`**：`OwnedTask` 类型独占访问 `owned` 字段。
- **`Notified`**：`Notified` 类型独占访问 `queue_next` 字段。
- **`JoinHandle`**：任务完成后，`JoinHandle` 独占访问 `stage` 字段。
- **`waker` 字段**：多个线程可以并发访问，但需要通过 `JOIN_WAKER` 位来控制访问权限，确保线程安全。

### 任务生命周期和访问控制
任务的生命周期由以下规则管理：
1. **`JOIN_WAKER`** 位控制对 `JoinHandle` 和运行时对 `waker` 字段的访问。
2. 任务完成后，`JoinHandle` 可以触发 waker，唤醒等待的线程。
3. 任务状态只能由当前线程设置和修改，以保证多线程访问时的一致性。

### 关于任务的所有权
``` rust
pub(crate) struct Task<S: 'static> {
    raw: RawTask,
    _p: PhantomData<S>,
}
pub(crate) struct Notified<S: 'static>(Task<S>);
pub(crate) struct UnownedTask<S: 'static> {
    raw: RawTask,
    _p: PhantomData<S>,
}
pub(crate) struct RawTask {
    ptr: NonNull<Header>,
}
```
因为Task持有的RawTask是一个NonNull类型的指针，也就是说它并没有实际上Header的所有权，因此我们要保证UnownedTask还在的时候，rawTask不能被释放，因此我们要避免因为作用域的原因自动调用与UnownedTask绑定的Task或者Notified的drop函数发生内存释放，因此我们采用`std::mem::forget()`函数使得rust不再管Task和Notified的drop，让UnownedTask接管内存的释放。

### 任务数据结构Cell<T, S>的构成
#### Header
- **state**: AtomicUsize，任务的状态和引用计数
- **queue_next**: 指向下一个 Header（任务头部）
- **vtable**:
  - **poll**: 任务轮询函数
  - **schedule**: 任务调度函数
  - **dealloc**: 内存释放函数
  - **try_read_output**: 读取任务输出
  - **drop_join_handle**: 丢弃 `JoinHandle`
  - **drop_abort_handle**: 丢弃中止句柄
  - **shutdown**: 调度器关闭
  - **trailer offset**: 尾部偏移
  - **scheduler offset**: 调度器偏移
  - **id offset**: ID 偏移
- **owned_id**: `Option<NonZeroU64>`

#### Core<T, S>
- **scheduler**: 类型为 `S` 的调度器
- **task_id**: 任务 ID
- **stage**: `CoreStage<T>`，表示 Future 或者 Output
  - **Running(T)**: 运行中
  - **Finished(Result<T::Output>)**: 已完成
  - **Consumed**: 已消费
#### Trailer
- **owned**: `linked_list::Pointers<Header>`
- **waker**: `Option<Waker>`，唤醒器
- **hooks**: `TaskHarnessScheduleHooks`，调度时的回调函数


### 任务的新建
首先调用Task的new()函数，进而在Task的new()中调用了rawTask的new()函数，这个函数通过Box::into_raw()分配一段空间并返回&mut类型指针，然后通过cast修改指针类型，进而使得rawTask是NonNull\<header\>的类型
``` rust
fn new_task<T, S>(
        task: T,
        scheduler: S,
        id: Id,
    ) -> (Task<S>, Notified<S>, JoinHandle<T::Output>)
    where
        S: Schedule,
        T: Future + 'static,
        T::Output: 'static,
    {
        let raw = RawTask::new::<T, S>(task, scheduler, id);
        let task = Task {
            raw,
            _p: PhantomData,
        };
        let notified = Notified(Task {
            raw,
            _p: PhantomData,
        });
        let join = JoinHandle::new(raw);

        (task, notified, join)
    }
    pub(super) fn new<T, S>(task: T, scheduler: S, id: Id) -> RawTask
    where
        T: Future,
        S: Schedule,
    {
        let ptr = Box::into_raw(Cell::<_, S>::new(task, scheduler, State::new(), id));
        let ptr = unsafe { NonNull::new_unchecked(ptr.cast()) };

        RawTask { ptr }
    }
```