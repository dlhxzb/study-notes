# 起因
```rust
async {
    let mut x = [0; 128];
    future_foo(&mut x).await;
    println!("{:?}", x);
}
```
会编译成：
```rust
struct FutureFoo<'a> {
    buf: &'a mut [u8], // 指向下面的`x`字段
}

struct AsyncFuture {
    x: [u8; 128],
    future_foo: FutureFoo<'what_lifetime?>,
}
```
自引用形成了，一旦AsyncFuture移动，buf指向的地址变的不合法了。如果能将Future 在内存中固定到一个位置，就可以避免这种问题的发生。

# Struct std::pin::Pin
`Pin`是一层壳，只有内部`Unpin`的时候才能用safe方法拿到内部mut，否则只有unsafe get_unchecked_mut。  
避免了内部修改也就避免了swap等内存位置变化，同时move的也仅是这层壳
```rust
pub struct Pin<P> {
    pub pointer: P,
}

impl<P> DerefMut for Pin<P>
where
    P: DerefMut,
    <P as Deref>::Target: Unpin,
```

# Trait std::marker::Unpin
它表明一个类型可以随意被移动，事实上，绝大多数类型都不在意是否被移动，它们都自动实现了 Unpin trait。

## 堆or栈
#### 固定在栈上，对于Unpin与!Unpin（需要unsafe）有两种方式
* `fn new(pointer: P) -> Pin<P>`要求`<P as Deref>::Target: Unpin`
* `unsafe fn new_unchecked(pointer: P) -> Pin<P>`随意
* 
#### 用Box::pin固定在堆上
Rust大部分类型都是分配在栈上的，通过栈变量管理堆内存（e.g.String）。  
将一个 !Unpin 类型的值固定到堆上，会给予该值一个稳定的内存地址，它指向的堆中的值在 Pin 后是无法被移动的。  
而且与固定在栈上不同，我们知道堆上的值在整个生命周期内都会被稳稳地固定住。
async 函数返回的 Future 默认就是 !Unpin 的。  
但是，在实际应用中，一些函数会要求它们处理的 Future 是 Unpin 的，此时，若你使用的 Future 是 !Unpin 的，必须要使用以下的方法先将 Future 进行固定:
* Box::pin， 创建一个 Pin<Box<T>>，因为自动impl<T, A> Unpin for Box<T, A>
* pin_utils::pin_mut!， 创建一个 Pin<&mut T>

固定后获得的 Pin<Box<T>> 和 Pin<&mut T> 既可以用于 Future ，又会自动实现 Unpin。
[Rust语言圣经-定海神针 Pin 和 Unpin](https://course.rs/advance/async/pin-unpin.html#%E5%AE%9A%E6%B5%B7%E7%A5%9E%E9%92%88-pin-%E5%92%8C-unpin)