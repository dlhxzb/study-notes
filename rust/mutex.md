- [std](#std)
  - [lock](#lock)
  - [unlock(MutexGuard::drop)](#unlockmutexguarddrop)
- [tokio](#tokio)
  - [lock](#lock-1)
  - [unlock](#unlock)
- [parking\_lot](#parking_lot)
  - [lock](#lock-2)
  - [unlock](#unlock-1)
- [区别](#区别)
- [补充std RwLock](#补充std-rwlock)
  - [write](#write)
    - [RwLockWriteGuard::Drop](#rwlockwriteguarddrop)
      - [`wake_writer_or_readers`:](#wake_writer_or_readers)
  - [read](#read)
    - [RwLockReadGuard::Drop](#rwlockreadguarddrop)


# std
[RTFSC](https://github.com/rust-lang/rust/blob/stable/library/std/src/sync/mutex.rs#L175)

```rust
pub struct Mutex<T: ?Sized> {
    inner: sys::Mutex,
    poison: poison::Flag,   // AtomicBool
    data: UnsafeCell<T>,
}

// sys::Mutex
pub struct Mutex {
    /// 0: unlocked
    /// 1: locked, no other threads waiting
    /// 2: locked, and other threads waiting (contended)
    futex: AtomicU32,
}
```

简单易懂，0/1/2三种状态，没人锁/没人等/有人等  
为避免在无竞争的时候也去内核取锁，这里用原子在用户态先尝试  

## lock
* 1. 调sys::Mutex::lock
  * 1.1. 尝试`compare_exchange`状态 0->1，成功则退出
  * 1.2. 失败则进入`lock_contended`
    * 1.2.1. 调spin：循环100次，发现state 0/2就返回  
      每次循环后调`spin_loop`，这个函数会调CPU指令`pause`来优化等待，与`thread::yield_now`调`libc::sched_yield`进入内核不一样
    * 1.2.2. state == 0，做下`1.1.`
    * 1.2.3. loop
      * 1.2.3.1. state swap成2，如果原值是0，说明锁上了，退出
      * 1.2.3.2. futex_wait：再看一眼state是不是2，是的话调libc::syscall(libc::SYS_futex..)进内核阻塞
      * 1.2.3.3. 同`1.2.1.`调`spin`
* 2. 锁上了，返回MutexGuard

```rust
    libc::syscall(
        libc::SYS_futex,
        futex as *const AtomicU32,  // state地址作为锁的ID
        libc::FUTEX_WAIT_BITSET | libc::FUTEX_PRIVATE_FLAG,
        expected,       // 2
        timespec.as_ref().map_or(null(), |t| t as *const libc::timespec),
        null::<u32>(), // This argument is unused for FUTEX_WAIT_BITSET.
        !0u32,         // A full bitmask, to make it behave like a regular FUTEX_WAIT.
    )
```

## unlock(MutexGuard::drop)
futex.swap(0)，当原值是2（有人等）时调 futex_wake -> syscall
```rust
    libc::syscall(
        libc::SYS_futex,
        futex as *const AtomicU32,
        libc::FUTEX_WAKE | libc::FUTEX_PRIVATE_FLAG,
        1,  // 只唤醒一个
    )
```

# tokio
[RTFSC](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/mutex.rs#L129)
  
```rust
pub struct Mutex<T: ?Sized> {
    s: semaphore::Semaphore,    
    c: UnsafeCell<T>,
}

pub(crate) struct Semaphore {
    // 默认std::mutex，feature = "parking_lot"时用parking_lot::Mutex            
    waiters: Mutex<Waitlist>, 
    /// The current number of available permits in the semaphore.
    permits: AtomicUsize,   // Mutex设为1，但要左移1位，最后一位保存Closed状态
}

struct Waitlist {
    queue: LinkedList<Waiter, <Waiter as linked_list::Link>::Target>,
    closed: bool,
}
```

## lock
* Semaphore::acquire(1).await -> poll_acquire
  * 从permits compare_exchange，成功返Ready
  * 失败调 waiters.lock() -> Mutex::lock
  * 注册Waker
  * Pending

## unlock
* Semaphore::release(1)
  * permits fetch_add
  * drop(waiters) // parking_lot::MutexGuard
  * wakers.wake_all()

# parking_lot
它的state划分的更细，把已上锁和有人等分成了2个bit

```rust
pub type Mutex<T> = lock_api::Mutex<RawMutex, T>;

pub struct RawMutex {
    /// This atomic integer holds the current state of the mutex instance. Only the two lowest bits
    /// are used. See `LOCKED_BIT` and `PARKED_BIT` for the bitmask for these bits.
    ///
    /// # State table:
    ///
    /// PARKED_BIT | LOCKED_BIT | Description
    ///     0      |     0      | The mutex is not locked, nor is anyone waiting for it.
    /// -----------+------------+------------------------------------------------------------------
    ///     0      |     1      | The mutex is locked by exactly one thread. No other thread is
    ///            |            | waiting for it.
    /// -----------+------------+------------------------------------------------------------------
    ///     1      |     0      | The mutex is not locked. One or more thread is parked or about to
    ///            |            | park. At least one of the parked threads are just about to be
    ///            |            | unparked, or a thread heading for parking might abort the park.
    /// -----------+------------+------------------------------------------------------------------
    ///     1      |     1      | The mutex is locked by exactly one thread. One or more thread is
    ///            |            | parked or about to park, waiting for the lock to become available.
    ///            |            | In this state, PARKED_BIT is only ever cleared when a bucket lock
    ///            |            | is held (i.e. in a parking_lot_core callback). This ensures that
    ///            |            | we never end up in a situation where there are parked threads but
    ///            |            | PARKED_BIT is not set (which would result in those threads
    ///            |            | potentially never getting woken up).
    state: AtomicU8,
}
```

## lock
RawMutex::lock
* loop
  * 没锁上锁，return
  * 没人等 调一次`spin`，`spin`有个累计调用计数：
    * `spin`在前3次调std `spin_loop`,CPU指令 pause 优化等待
    * 4~10次调`thread::yield_now`进入内核
    * 10次以上返回false
  *  `spin` true-> continue, false 继续下一步
  *  没人等 设置为 有人等
  
然后该上锁了，事情从这里开始变得有趣起来。。  
这里有个全局的`HashTable`，用来存锁，RawMutex的地址位移后作为下标，从`HashTable`中取得一个Bucket
```rust
static HASHTABLE: AtomicPtr<HashTable> = AtomicPtr::new(ptr::null_mut());

struct HashTable {
    // Hash buckets for the table
    entries: Box<[Bucket]>,

    // Number of bits used for the hash function
    hash_bits: u32,

    // Previous table. This is only kept to keep leak detectors happy.
    _prev: *const HashTable,
}

struct Bucket {
    // Lock protecting the queue
    mutex: WordLock,

    // Linked list of threads waiting on this bucket
    queue_head: Cell<*const ThreadData>,
    queue_tail: Cell<*const ThreadData>,

    // Next time at which point be_fair should be set
    fair_timeout: UnsafeCell<FairTimeout>,
}

pub struct WordLock {
    state: AtomicUsize,
}
```
  
`Bucket::mutex`实际是个AtomicUsize，对它lock与之前RawMutex::lock类似，  
loop里面尝试修改原子值，spin，再然后
* 把自己入queue
* while futex!=0 syscall(SYS_futex)进内核阻塞
* reset spin计数，更新state继续loop

## unlock
略

# 区别
锁的实现都是在用户态自旋再切换到内核态，不同的是tokio返回Pending，parking_log多一层全局HashTable

# 补充std RwLock

```rust
pub struct RwLock<T: ?Sized> {
    inner: sys::RwLock,
    poison: poison::Flag,
    data: UnsafeCell<T>,
}

// sys::RwLock
pub struct RwLock {
    // The state consists of a 30-bit reader counter, a 'readers waiting' flag, and a 'writers waiting' flag.
    // Bits 0..30:
    //   0: Unlocked
    //   1..=0x3FFF_FFFE: Locked by N readers
    //   0x3FFF_FFFF: Write locked
    // Bit 30: Readers are waiting on this futex.
    // Bit 31: Writers are waiting on the writer_notify futex.
    state: AtomicU32,
    // The 'condition variable' to notify writers through.
    // Incremented on every signal.
    writer_notify: AtomicU32, // 每次内核唤醒通知都+1，用来传递内核当前值
}
```
* state`Bit 0~29`是读写锁，写满是写锁，否则是读锁（reader数量）
* `Bit 30` 表示读等待，只有上了写锁或写锁刚释放时会有读等待  
* `Bit 31` 表示写等待，后面会介绍写优先级高于读

## write
sys::RwLock:  
* state==0（没锁） 没有读写等待时立即成功返回
* spin直到没锁或者有写等待（为公平竞争写锁？）
* loop
  * state没锁则尝试写state（Write locked + Bit 31 Writers waiting），return
  * 以writer_notify地址为ID调用futex_wait，进内核阻塞
  * spin直到没锁或者有writer等待
之后返回`RwLockWriteGuard`

### RwLockWriteGuard::Drop
#### `wake_writer_or_readers`:
* 只有写等待时（state Bit 31），
  * state清0
  * writer_notify+=1
  * futex_wake(writer_notify地址)，唤醒其它写等待
* 读写都有等待时（state Bit 30 31），**优先写**
  * state保持读等待，清掉写等待
  * writer_notify+=1
  * futex_wake(writer_notify地址)，唤醒其它写等待
* 只有读等待 （state Bit 30）
  * state清0
  * futex_wake(state)，唤醒所有读等待

## read
* 上锁（state+=1），条件是is_read_lockable: reader没max，没有写等待，没有读等待（无写有读等待产生在写释放时，此时优先写）
* spin直到没有写锁或有读等待或有写等待
* loop
  * is_read_lockable则上锁
  * 写`Bit 30`
  * 以state地址为ID调用futex_wait，进内核阻塞
* spin直到没有写锁或有读等待或有写等待

### RwLockReadGuard::Drop
* state-=1  
* 当最后一个reader释放且有写等待时(state=0，Bit31)：[wake_writer_or_readers](#wake_writer_or_readers)
