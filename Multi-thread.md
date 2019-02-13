###多线程安全

```
读MrPeak杂货铺：
https://zhuanlan.zhihu.com/p/23998703
```

####属性类型分为引用类型和值类型。

####对引用类型属性的操作分为对指针和内存进行操作。

####举例：

```
@property (atomic, assign)int count; //值类型
@property (atomic, strong)NSString *name; //引用类型

int count = 0;
self.name = @"william";//对指针操作
[self.name rangeOfString:@"william"];//读取name属性的内存

```

---

+ 内存的访问是串行的，并不会导致内存数据的错乱或者应用的crash
+ 如果读写（load or store）的内存长度小于等于地址总线的长度，那么读写的操作是原子的，一次完成。比如bool，int，long在64位系统下的单次读写都是原子操作

####atomic有两点用处

#####1、生成原子操作的setter和getter。

#######在setter和getter方法中自动加锁。

#####2、设置Memory Barrier

######在OC中所有的锁最后都会设置Memory Barrier

> Memory Barrier能够保证内存的操作顺序，按照我们代码的书写顺序来。

* atomic声明只是setter和getter是原子性操作，但是在对变量操作过程中，不一定是原子性的，所以atomic不一定能够保证线程安全。

> 如何保证线程安全：
> 只需要保证操作是atomic的即可

> 最容易出错或者crash的地方是对指针指向的内存区域进行操作时，需要进行加锁处理。

> * @synchronized(token)
> * NSLock
> * dispatch_semaphore_t
> * OSSpinLock

> 性能损耗依次从上到下更小




















