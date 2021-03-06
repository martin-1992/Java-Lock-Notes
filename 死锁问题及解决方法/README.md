## 死锁问题及解决方法
　　双方持有锁，并等待对方释放锁。比如线程 A 持有 A 锁，等待对方释放 B 锁，线程 B 持有 B 锁，等待对方释放 B 锁。

### 死锁产生的条件

- 互斥，共享资源 X 和 Y 只能被一个线程占用；
- 占有且等待，比如线程 T1 获得共享资源 X，会不释放该共享资源，继续等待共享资源 Y；
- 不可抢占，其它线程不能强行抢占线程 T1 占有的资源；
- 循环等待，线程 T1 等待线程 T2 释放资源，线程 T2 等待线程 T1 释放资源。

### 破坏占有且等待条件
　　该条件是获取一个资源（锁），然后继续等待获取下个资源（锁）。

- 只要一次性获取所有资源（锁）即可，比如线程 T1 要一次性获取资源 X 和 Y；
- 规定一个线程只能获取一个锁。

### 破坏不可抢占条件
　　即能主动释放占有的资源，比如使用定时锁，规定时间内获取不到，则会释放。<br />
　　注意，使用 synchronized 关键字，没有时限的，获取不到锁时，线程会一直进入阻塞状态，无法打断。解决方法是不使用 synchronized，而用 java.util.concurrent 的包。

### 破坏循环等待条件
　　循环等待是双方互相等待对方的资源，只要对资源（id）进行排序，申请资源时，按从小到大的顺序申请。即只能由 A（小的 ID） -> B（大的 ID），不能 A <-> B，这样就不会产生互相等待对方的资源了。<br />