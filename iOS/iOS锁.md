## iOS锁

### iOS锁类型

1. OSSpinLock
2. Dispatch_semaphore
3. Pthread_mutex
4. Pthread_mutex(recursive)
5. NSLock
6. NSCondition
7. NSConditionLock
8. NSRecursiveLock
9. @synchronized

### 什么是锁

在多线程中可能会访问同一个资源，很容易引起数据错乱和数据安全的问题。这样我们就要保证一次只有一个线程访问这一块资源。

### 比较熟悉的锁

@synchronized、NSLock、NSRecursiveLock

### 引申

##### 自旋锁

自旋锁于互斥锁类似，但是区别是，自旋锁不会造成调用的休眠，如果自旋锁已经被其他的持有，调用者就会一直在那循坏查看是否释放。因为其不会引起调用者的休眠，所以其效率就高于互斥锁。

但是自旋锁的问题是，自旋锁在其他调用者占用的时候，一直不会休眠运行，所以会一直占用着CPU，如果短时间不能获得锁，就会降低CPU效率。

##### 互斥锁

互斥锁属于sleep-wating类型的锁，例如在一个双核机器 Core_0 和 Core_1上分别有A、B两个线程，假如线程B想通过 pthread_mutex_lock操作去获取一个临界区的锁，而这个锁此时被线程A所持有，线程A就会阻塞（blocking）,Core_1此时就会进行上下文切换，将A线程置于等待中，Core_1就可以去执行其他的任务，比如线程C,而不必一直等待。自旋锁则不然，他是 busy-wating ,线程A如果是使用 pthread_spin_lock 请求操作锁，就会在 Core_1 上不停进行锁请求，直到得到这个锁为止。

### OSSpinLock

自旋锁，性能最高的锁，缺点就是上述自旋锁的缺点，但是iOS的 OSSpinLock 暴露出不安全的问题，详情见[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

简述，就是iOS维护了5个线程优先级，高优先级的线程始终会先于低线程的执行，一个线程不会受到比他低优先级线程干扰。但是这种线程调度算法会产生潜在的优先级反转的问题，从而破坏 spin lock.

### NSLock

这是iOS的基础锁，属于互斥锁，遵循NSLocking，是iOS对 pthread_mutex 进行的封装，使用方法就是简单的lock，unlock，try lock，lockBeforeDate。

NSLock保证了线程安全性，但是如果连续两次锁定，上次还未释放，就会造成堵塞，这时候就要用到了递归锁。

### NSRecursiveLock

递归锁，是为了解决在一个线程中多次锁定的问题，而不会造成死锁。 记录上锁和解锁的次数，当二者平衡的时候，才会释放锁

### NSCondition

带条件的锁



### @synchronized

递归锁

