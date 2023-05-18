# Send & Sync

多线程并发访问的两种情况：
* 同类型多个变量内部存在共享数据（浅拷贝）- Send约束：没有浅拷贝类型都是Send，如果有浅拷贝，必须保证副本并发读写没有数据竞争
* 同一变量多个不可变引用 - Sync约束：没有内部可变性的类型都实现了Sync，内部可变的要保证引用读写没有数据竞争
   
大多数情况下编译器会自动推导实现这两个trait。  
有自动实现 `impl<T:Sync> Send for &T`。反之逻辑也成立，但是编译器不为&T: Send自动实现T:Sync。  

### Send
类型自身可以在线程间移动。这个移动可以是move，也可以是copy   
通常移动本身都是允许的（特例：MutexGuard语义上不允许线程外drop），主要是移动后不能存在数据竞争。  
  
个人理解移动后按照是否可能数据竞争分两种情况：
* 没有拷贝，就无并发访问，没有数据竞争：
  * 只能move，不存在多个同类型对象操作同一内存的，只被一个线程持有。比如&mut、Mutex
* 有拷贝，必有并发访问，可能有数据竞争：
  * 自身深拷贝，自身肯定没问题，只要内部T:Send即可。比如基本类型、RefCell、Vec、Box、&
  * 自身浅拷贝，首先对自身数据是否有原子/锁保护，比如Arc的引用计数。其次浅拷贝导致内部泛型T允许并发访问即T:Sync。最后其所拥有的泛型T允许移动T:Send

反过来理解，什么样的类型是!Send？**有浅拷贝方法且内部可变性会导致数据竞争的**

### Sync
类型的引用可以多线程并发访问（由于内部可变性有些不可变访问也是会改值，比如clone改引用计数）  
T: Sync <-> &T: Send  
有趣：从Sync定义可以得知如果T是Sync，那么&T就是Sync的，那么&mut T也是Sync的，因为& &mut T相当于& &T，是read-only  

* Send：
  * 1.从创建它的线程以外的线程`可变`地访问该值（包括drop）；
  * 2.从创建它的线程以外的线程`不可变`地访问该值（不一定并行，可以是所有权在线程间move）
* Sync：从多个线程通过`不可变引用`并行访问值
  
### 区别
* Send + !Sync：T可以多个线程间移动，但不能并行访问同一个&T。有内部可变但没有浅拷贝的（Refcell）显然允许自身转移，但不能把引用给多个线程
* !Send + Sync：不可变地从多个线程并行访问值，可变访问只能发生在创建它的线程上（MutexGuard）。
  这种情况比较少见，想象如果Arc允许as_mut，并且拷贝要求&mut（没有内部可变性），那么它的不可变引用并发访问就没问题（Sync）。但是它的副本被多线程as_mut就数据竞争了（!Send）

### 举个栗子
* &T：`Send(T:Sync) + Sync(T:Sync)`  
  * Send：因为&T:copy，移动到其他线程后&T会引发并行访问，又因为&是只读的，所以只需满足T:Sync`不可变引用`并行访问即可。T本身并没有转移，无需T:Send  
  * Sync：要满足&&T并行访问，等于&T并行访问，即T:Sync
* &mut T：`Send(T:Send) + Sync(T:Sync)`  
  * Send: 可变引用无法copy，在其生效期间编译器也允许有其它引用，因此转移到其他线程后不存在并发访问
  可变引用无法copy，因此将它们发送到其他线程不允许从多个线程并行访问，因此即使 T 不是 Sync，&mut T 也可以Send。当然，T 仍然必须是Send，至少不能Send就不能线程外drop。
  * Sync：&&mut相当于&，满足T:Sync即可
* Rc<T>：`!Send + !Sync`  
  * Send：它可以clone，如果允许线程间转移（Send）,多个线程就会拿到指向同一数据的Rc,它们并行clone/drop将会数据竞争。
  * Sync: 不可变访问clone(&self)同样数据竞争。
* Arc<T>：`Send(T:Sync+Send) + Sync(T:Sync+Send)`  
  * Send：这主要表现得像 &T。它可以被clone，所以发送到其他线程需要T:Sync。而Arc<T> 可能将最后的T保留在任意线程上，所以与&T不同它还需要T能转移T:Send
  * Sync：&Arc<T>被并发访问时，因为Arc基本没有内部可变性，所以相当于不可变引用&T的并发访问，这就要求T:Sync。与Mutex类似，防止T内部可变性引起数据竞争T:Send
* MutexGuard：`!Send + Sync(T:Sync)`  
  * Send：在另一个线程上销毁 MutexGuard 是不合理的（它要unlock Mutex），因此不能Send。
  * Sync：如果其内部值可以不可变地从多个线程并行访问（T:Sync），那么这种不可变访问在 &MutexGuard 本身上也是安全的
* Mutex：`Send(T:Send) + Sync(T:Send)`  
  * Send：其自身不能clone，移动时唯一，只要T允许移动(T:Send)Mutex就也可移动。  
  * Sync：&Mutex由于内部有原子保护，不存在同时并发访问T（会排队），所以不需要T:Sync。但是因为其内部可变性如果T:!Send就可能有另外的锁包含T的浅拷贝，此时两个锁还是会数据竞争
* RwLock<T>: `Send(T:Send) + Sync(T:Sync+Send)`  
  * Send：与Mutex类似，由于自身不能拷贝，移动时只有内部T可移动即可T:Send  
  * Sync：与Mutex不同，读访问可以并发（T:Sync）。这与Arc类似，T如果有浅拷贝就要防止内部可变性引起数据竞争T:Send
* RefCell<T>：`Send(T:Send) + !Sync`   
  * Send：它的clone是深拷贝，移动时只要满足T:Send即可
  * Sync：RefCell用Cell来保护其内部可变性，在多线程并发下这将是一场数据竞争，必不可能Sync
* Box<T>: `Send(T:Send) + Sync(T:Sync)`
  * Send：Box没有浅拷贝，它能不能move取决于内部T能不能move(T:Send)
  * Sync：&Box<T>没有内部可变性，能不能并发访问取决于&T即T:Sync

[Alice's comment](https://stackoverflow.com/questions/68704717/is-the-sync-trait-a-strict-subset-of-the-send-trait-what-implements-sync-withou/68708557#68708557)