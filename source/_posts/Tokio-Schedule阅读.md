---
title: Tokio-Schedule阅读
date: 2024-12-13 22:41:47
tags:
categories: Tokio
---

## Multi-threads

因为默认的配置是多线程的，所以先阅读的多线程的部分

### mod.rs

这里主要是关于 Schedule 的 Handle 以及调度 worker 运行时的 Context，他们都通过 enum 进行了一层抽象，就是里面都是差不多的内容，不过外面包了一层枚举类型，你可以通过这个类型来找到当前的 Runtime 是多线程还是单线程的，接下来是 Handle 和 Context 具体的内容

```rust
pub(crate) struct Handle {
    /// Task spawner
    pub(super) shared: worker::Shared,

    /// Resource driver handles
    pub(crate) driver: driver::Handle,

    /// Blocking pool spawner
    pub(crate) blocking_spawner: blocking::Spawner,

    /// Current random number generator seed
    pub(crate) seed_generator: RngSeedGenerator,

    /// User-supplied hooks to invoke for things
    pub(crate) task_hooks: TaskHooks,
}
pub(crate) struct Context {
    /// Worker
    worker: Arc<Worker>,

    /// Core data
    core: RefCell<Option<Box<Core>>>,

    /// Tasks to wake after resource drivers are polled. This is mostly to
    /// handle yielded tasks.
    pub(crate) defer: Defer,
}
```

### 多线程

#### 多线程调度器的创建

```rust
pub(super) fn create(
    size: usize,
    park: Parker,
    driver_handle: driver::Handle,
    blocking_spawner: blocking::Spawner,
    seed_generator: RngSeedGenerator,
    config: Config,
) -> (Arc<Handle>, Launch) {
    let mut cores = Vec::with_capacity(size);
    let mut remotes = Vec::with_capacity(size);
    let mut worker_metrics = Vec::with_capacity(size);

    // Create the local queues
    for _ in 0..size {
        let (steal, run_queue) = queue::local();

        let park = park.clone();
        let unpark = park.unpark();
        let metrics = WorkerMetrics::from_config(&config);
        let stats = Stats::new(&metrics);

        cores.push(Box::new(Core {
            tick: 0,
            lifo_slot: None,
            lifo_enabled: !config.disable_lifo_slot,
            run_queue,
            is_searching: false,
            is_shutdown: false,
            is_traced: false,
            park: Some(park),
            global_queue_interval: stats.tuned_global_queue_interval(&config),
            stats,
            rand: FastRand::from_seed(config.seed_generator.next_seed()),
        }));

        remotes.push(Remote { steal, unpark });
        worker_metrics.push(metrics);
    }

    let (idle, idle_synced) = Idle::new(size);
    let (inject, inject_synced) = inject::Shared::new();

    let remotes_len = remotes.len();
    let handle = Arc::new(Handle {
        task_hooks: TaskHooks {
            task_spawn_callback: config.before_spawn.clone(),
            task_terminate_callback: config.after_termination.clone(),
        },
        shared: Shared {
            remotes: remotes.into_boxed_slice(),
            inject,
            idle,
            owned: OwnedTasks::new(size),
            synced: Mutex::new(Synced {
                idle: idle_synced,
                inject: inject_synced,
            }),
            shutdown_cores: Mutex::new(vec![]),
            trace_status: TraceStatus::new(remotes_len),
            config,
            scheduler_metrics: SchedulerMetrics::new(),
            worker_metrics: worker_metrics.into_boxed_slice(),
            _counters: Counters,
        },
        driver: driver_handle,
        blocking_spawner,
        seed_generator,
    });

    let mut launch = Launch(vec![]);

    for (index, core) in cores.drain(..).enumerate() {
        launch.0.push(Arc::new(Worker {
            handle: handle.clone(),
            index,
            core: AtomicCell::new(Some(core)),
        }));
    }

    (handle, launch)
}
```

这是多线程下调度器的创建，大概就是初始化了一下调度器下 worker 的队列以及 shared、remote 等内容，最后返回调度器的 Handle 和 worker 的一个 vec，也就是 launch

#### 多线程下的 worker 是怎样跑起来的

先调用了 worker.rs 下的 run,初始化这个 worker 的上下文，以及是运行在哪个线程下的，应该在主函数里是在`thread::spawn(run(worker))`,然后这个 run 初始化好上下文之后，调用上下文的 run 真正开始跑起来这个 worker，这个context就是每次检查一下当前worker是否被shutdown，没有的话就找个任务来跑。
```rust
fn run(worker: Arc<Worker>) {
    let core = match worker.core.take() {
        Some(core) => core,
        None => return,
    };
    worker.handle.shared.worker_metrics[worker.index].set_thread_id(thread::current().id());
    let handle = scheduler::Handle::MultiThread(worker.handle.clone());
    crate::runtime::context::enter_runtime(&handle, true, |_| {
        // Set the worker context.
        let cx = scheduler::Context::MultiThread(Context {
            worker,
            core: RefCell::new(None),
            defer: Defer::new(),
        });

        context::set_scheduler(&cx, || {
            let cx = cx.expect_multi_thread();
            assert!(cx.run(core).is_err());
            // Check if there are any deferred tasks to notify. This can happen when
            // the worker core is lost due to `block_in_place()` being called from
            // within the task.
            cx.defer.wake();
        });
    });
}
//主要就是设置了上下文，然后调用上下文的run
fn run(&self, mut core: Box<Core>) -> RunResult {
    while !core.is_shutdown {
        // First, check work available to the current worker.
        if let Some(task) = core.next_task(&self.worker) {
            core = self.run_task(task, core)?;
            continue;
        }
        if let Some(task) = core.steal_work(&self.worker) {
            // Found work, switch back to processing
            core.stats.start_processing_scheduled_tasks();
            core = self.run_task(task, core)?;
        } else {
            // Wait for work
            core = if !self.defer.is_empty() {
                self.park_timeout(core, Some(Duration::from_millis(0)))
            } else {
                self.park(core)
            };
            core.stats.start_processing_scheduled_tasks();
        }
    }
}
```
