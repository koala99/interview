  <h1>多线程相关问题</h1>
 
  1. int自增线程安全：  
  AtomicInteger，一个提供原子操作的Integer的类。在Java语言中，++i和i++操作并不是线程安全的，在使用的时候，不可避免的会用到synchronized关键字。而AtomicInteger则通过一种线程安全的加减操作接口。 来看AtomicInteger提供的接口。
  
 int 适合于单线程变量存取，开销小，速度快

 AtomicInteger适合多线程变量存取，能够保证线程安全，但是速度较慢

 AtomicInteger,如果在线程较多的情况下，效率会变的很低，因为没有加锁，其他线程会频繁打断存取的过程，导致较低

 



