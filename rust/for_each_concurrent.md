## futures::stream::StreamExt::for_each_concurrent
这是来自futures-rs库的一个常用并发函数。与join/join_all类似，在单线程内并发执行多个future。  
根据自身经验，多用于proxy/dispatch之类任务分派的场合，如果是耗算力任务要配合spawn。

## 源码解析
[source](https://github.com/rust-lang/futures-rs/blob/0.3.28/futures-util/src/stream/stream/for_each_concurrent.rs#L75) 👈  
```rust
    pub struct ForEachConcurrent<St, Fut, F> {
        #[pin]
        stream: Option<St>, // 数据源
        f: F, // 用户定义处理函数
        futures: FuturesUnordered<Fut>, // 执行中的Future集合
        limit: Option<NonZeroUsize>, // 并发数限制
    }
```
`poll`内loop循环：
* 检查limit是否达到，None的话不限制
* 从stream中取数据，配合f压入futures
* futures.poll_next_unpin可驱动集合中所有future