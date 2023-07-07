- [std](#std)
  - [channel创建](#channel创建)
    - [channel()](#channel)
    - [sync\_channel(bound:usize)](#sync_channelboundusize)
  - [send](#send)
  - [recv](#recv)
- [tokio](#tokio)
  - [channel创建](#channel创建-1)
  - [send](#send-1)
  - [recv](#recv-1)
  - [tx drop](#tx-drop)

# std
## channel创建
mpsc内部封装的都是mpmc，mpmc目前也只有这一个用的地方，它是来自 crossbeam-channel实现  
两种channel：
### channel() 
Sender和Receiver都是Counter的裸指针
```rust
struct Counter<C> {
    /// The number of senders associated with the channel.
    senders: AtomicUsize,

    /// The number of receivers associated with the channel.
    receivers: AtomicUsize,

    /// Set to `true` if the last sender or the last receiver reference deallocates the channel.
    destroy: AtomicBool,

    /// The internal channel.
    chan: C, // list::Channel
}

// list::Channel
pub(crate) struct Channel<T> {
    /// The head of the channel.
    head: CachePadded<Position<T>>,

    /// The tail of the channel.
    tail: CachePadded<Position<T>>,

    /// Receivers waiting while the channel is empty and not disconnected.
    receivers: SyncWaker,
}

// 原子链表
struct Position<T> {
    /// The index in the channel.
    index: AtomicUsize,

    /// The block in the linked list.
    block: AtomicPtr<Block<T>>,
}
```

### sync_channel(bound:usize)
不同的是`list::Channel`换成`array::Channel`，这个Channel里包含一个数组而不是链表

## send
`list::Channel`由于有无限缓冲区，send立即成功；`array::Channel`要在loop里等待缓冲区可用  
写完数据对observers unpark

## recv
loop，无数据时`thread::park()`

# tokio
## channel创建
`fn channel(semaphore)`创建tx、rx，入参要求实现Semaphore trait
  
```rust
pub(crate) trait Semaphore {
    fn is_idle(&self) -> bool;
    fn add_permit(&self);
    fn close(&self);
    fn is_closed(&self) -> bool;
}
```
  
`bounded`用的`semaphore::Semaphore`在[mutex](/rust/mutex.md#tokio) 介绍过，锁<Waitlist> + 原子计数
```rust
pub(crate) struct Semaphore {
    waiters: Mutex<Waitlist>,
    /// The current number of available permits in the semaphore.
    permits: AtomicUsize,
}
```
  
`unbounded` 的channel入参则仅需一个 AtomicUsize
  
channel创建出`Chan`,tx、rx一样都只是`Arc<Chan>`套壳
  
```rust
pub(super) struct Chan<T, S> {
    /// Notifies all tasks listening for the receiver being dropped.
    notify_rx_closed: Notify,

    /// Handle to the push half of the lock-free list.
    tx: list::Tx<T>,    // 原子链表，存数据

    /// Coordinates access to channel's capacity.
    semaphore: S,

    /// Receiver waker. Notified when a value is pushed into the channel.
    rx_waker: AtomicWaker,

    /// Tracks the number of outstanding sender handles.
    ///
    /// When this drops to zero, the send half of the channel is closed.
    tx_count: AtomicUsize,

    /// Only accessed by `Rx` handle.
    rx_fields: UnsafeCell<RxFields<T>>,
}

pub struct Notify {
    // `state` uses 2 bits to store one of `EMPTY`,
    // `WAITING` or `NOTIFIED`. The rest of the bits
    // are used to store the number of times `notify_waiters`
    // was called.
    //
    // Throughout the code there are two assumptions:
    // - state can be transitioned *from* `WAITING` only if
    //   `waiters` lock is held
    // - number of times `notify_waiters` was called can
    //   be modified only if `waiters` lock is held
    state: AtomicUsize,
    waiters: Mutex<WaitList>,
}

struct RxFields<T> {
    /// Channel receiver. This field is only accessed by the `Receiver` type.
    list: list::Rx<T>,

    /// `true` if `Rx::close` is called.
    rx_closed: bool,
}

// list::Tx
pub(crate) struct Tx<T> {
    /// Tail in the `Block` mpmc list.
    block_tail: AtomicPtr<Block<T>>,

    /// Position to push the next message. This references a block and offset
    /// into the block.
    tail_position: AtomicUsize,
}

// list::Rx
pub(crate) struct Rx<T> {
    /// Pointer to the block being processed.
    head: NonNull<Block<T>>,

    /// Next slot index to process.
    index: usize,

    /// Pointer to the next block pending release.
    free_head: NonNull<Block<T>>,
}
```
  
Tx与Rx指向同一个Block<T>(原子链表)

## send
bounded要先从chan.semaphore中acquire 1，这里存在异步等待。  
然后与unbounded相同，push到chan.tx，chan.rx_waker.wake()

## recv
从链表pop，空Pending，tx关闭Ready(None)，注册waker等待send wake

## tx drop
tx_count-=1,最后一个的时候在链表中推入一个flg=TX_CLOSED的数据