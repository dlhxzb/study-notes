## Ring buffer
Tasks 存在如下ring buffer中，👉 [queue.rs](https://github.com/tokio-rs/tokio/blob/tokio-1.26.0/tokio/src/runtime/scheduler/multi_thread/queue.rs#L36-L57)
* `head: AtomicU64`（根据系统不同也可为U32）
* `tail: AtomicU32`（也可U16）
* `buffer: Box<[...<T>; 256]>`
  
`head`会unpack成两个U32：real和steal。real指实际head，steal指正在被其它线程偷取的task
### 当前 Thread 拿取本地的 Task
* load head
* if real==steal，说明无人偷取
  * real+=1,steal+=1
  * CAS
* if real!=steal，说明正在偷取
  * head+=1
  * CAS

### 偷取其他 Thread 的任务
* load head
* if real!=steal，说明其它线程正在偷取，等待重试
* if real==steal，说明无人偷取
  * head+=n，预留出n（一半）个task为偷取区域
  * CAS
  * n个task偷取到自己队列后，steal=head
  * CAS 

使用一个原子变量存储 Steal 和 Head 两个值。使用CAS 操作，保证操作过程中，没有其他人进行改动，如果有则重试。
同一时间只有一个偷窃操作能够成功。  
上述的方法就实现了仅仅使用一个原子变量也能保证并发安全的 ring buffer，非常巧妙。

### Work Steal 的其他优化
除了上述的核心数据结构，还有一些其他的技巧，被用以提高任务偷取的效率，这里简单列出几项：  
限制同时实施任务偷取的线程数量。由于等待线程唤醒是一次性唤醒所有人，所以有可能发生大家都在偷其他线程的任务，竞争太激烈。
为了避免上述情况的发生，Tokio 默认只允许一半的线程实施任务偷取，剩下的线程继续进入睡眠。  
偷取对象的公平性。所谓“薅羊毛不能盯着一头羊薅”，偷取目标的选择也应该是随机的，因此初试偷窃目标是通过一个随机数生成器来决定，保证了公平性，让队列的任务更加平均。  
偷取操作不应该太频繁，所以每次偷取任务的数量也不能太少，所以 Tokio 采取了“批发”的策略，每次都偷取当前队列一半数量的任务。  

[Tokio 解析之任务调度](https://baijiahao.baidu.com/s?id=1746023143258422548&wfr=spider&for=pc)