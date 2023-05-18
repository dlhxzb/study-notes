https://night-cruise.github.io/async-rust/%E5%BC%82%E6%AD%A5%E8%BF%90%E8%A1%8C%E6%97%B6.html

## Future运行机制，自上而下
* Executor：就是runtime，接收task，取出future，创建LocalWaker，调用Future::poll()
* Waker/Context：Waker由Executor构建，封装在Context 中，传递给 Future::poll()。
  如果返回Pending，它还必须以某种方式存储 waker（Future内），并在future应该再次poll时调用 Waker::wake()。
  Waker里面有`data`（即task）,`vtable`，用来调用wake方法：(vtable.wake)(data)
* task：包含future与发送到Executor的sender，要实现wake/wake_by_ref，在waker.wake()时调用，把自己投送进sender
* Future：里面有个Waker记录task。实现poll推进业务逻辑

io -> Waker::wake() -> Executor -> Future::poll() 


```rust
/*
[dependencies]
futures = "0.3"
*/

use futures::future::BoxFuture;
use futures::task::{waker_ref, ArcWake};
use futures::FutureExt;
use std::future::Future;
use std::pin::Pin;
use std::sync::mpsc::{sync_channel, Receiver, SyncSender};
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::Duration;

pub fn new_executor_and_spawner() -> (Executor, Spawner) {
    // 任务通道允许的最大缓冲数(任务队列的最大长度)
    // 当前的实现仅仅是为了简单，在实际的执行中，并不会这么使用
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor { ready_queue }, Spawner { task_sender })
}

/// 一个Future，它可以调度自己(将自己放入任务通道中)，然后等待执行器去`poll`
struct Task {
    /// 进行中的Future，在未来的某个时间点会被完成
    ///
    /// 按理来说`Mutex`在这里是多余的，因为我们只有一个线程来执行任务。但是由于
    /// Rust并不聪明，它无法知道`Future`只会在一个线程内被修改，并不会被跨线程修改。因此
    /// 我们需要使用`Mutex`来满足这个笨笨的编译器对线程安全的执着。
    ///
    /// 如果是生产级的执行器实现，不会使用`Mutex`，因为会带来性能上的开销，取而代之的是使用`UnsafeCell`
    future: Mutex<Option<BoxFuture<'static, ()>>>,

    /// 可以将该任务自身放回到任务通道中，等待执行器的poll
    task_sender: SyncSender<Arc<Task>>,
}

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // 通过发送任务到任务管道的方式来实现`wake`，这样`wake`后，任务就能被执行器`poll`
        let cloned = arc_self.clone();
        arc_self.task_sender.send(cloned).expect("任务队列已满");
    }
}

/// `Spawner`负责创建新的`Future`然后将它发送到任务通道中
#[derive(Clone)]
pub struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

impl Spawner {
    pub fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        // let future = std::pin::pin!(future);
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone(),
        });
        self.task_sender.send(task).expect("Channel已关闭");
    }
}

/// 任务执行器，负责从通道中接收任务然后执行
pub struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

impl Executor {
    pub(crate) fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            // 获取一个future，若它还没有完成(仍然是Some，不是None)，则对它进行一次poll并尝试完成它
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                // 基于任务自身创建一个 `LocalWaker`
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&waker);
                // `BoxFuture<T>`是`Pin<Box<dyn Future<Output = T> + Send + 'static>>`的类型别名
                // 通过调用`as_mut`方法，可以将上面的类型转换成`Pin<&mut dyn Future + Send + 'static>`
                if future.as_mut().poll(context).is_pending() {
                    // Future还没执行完，因此将它放回任务中，等待下次被poll
                    *future_slot = Some(future);
                }
            }
        }
    }
}

pub struct StringProcessFuture {
    task: Arc<Mutex<StringProcessTask>>,
}

pub struct StringProcessTask {
    s: String,
    processor: fn(&mut String),
    complete_checker: fn(&str) -> bool,
    waker: Option<Waker>,
}

impl StringProcessTask {
    pub fn check_task(&self) -> bool {
        (self.complete_checker)(&self.s)
    }

    pub fn process(&mut self) {
        while !self.check_task() {
            (self.processor)(&mut self.s);
            println!("cur len: {}", self.s.len());
            thread::sleep(Duration::new(1, 0));
        }
    }
}

impl Future for StringProcessFuture {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut task = self.task.lock().unwrap();
        if task.check_task() {
            Poll::Ready(task.s.clone())
        } else {
            task.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

impl StringProcessFuture {
    pub fn new(processor: fn(&mut String), complete_checker: fn(&str) -> bool) -> Self {
        let task = Arc::new(Mutex::new(StringProcessTask {
            s: String::new(),
            processor,
            complete_checker,
            waker: None,
        }));

        // 创建新线程
        let shared_task = task.clone();
        thread::spawn(move || {
            // Do real "IO" cost task
            let mut shared_state = shared_task.lock().unwrap();
            shared_state.process();
            // 通知执行`poll`对应的`Future`
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        StringProcessFuture { task }
    }
}

fn string_process_demo() {
    let (executor, spawner) = new_executor_and_spawner();

    // 生成一个任务
    spawner.spawn(async {
        println!("before StringProcessFuture!");
        // 创建定时器Future，并等待它完成
        let res = StringProcessFuture::new(
            |s: &mut String| {
                s.push_str(" hello ");
            },
            |s: &str| -> bool { s.len() > 10 },
        )
        .await;
        println!("res: {}", res);
    });

    spawner.spawn(async {
        println!("before StringProcessFuture!");
        // 创建定时器Future，并等待它完成
        let res = StringProcessFuture::new(
            |s: &mut String| {
                s.push_str(" world ");
            },
            |s: &str| -> bool { s.len() > 20 },
        )
        .await;
        println!("res: {}", res);
    });

    // drop掉任务，这样执行器就知道任务已经完成，不会再有新的任务进来
    drop(spawner);

    // 运行执行器直到任务队列为空
    executor.run();
}

fn main() {
    println!("\nstring_process_demo:");
    string_process_demo()
}
```