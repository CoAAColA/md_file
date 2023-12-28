# Concurrency Control



## task #1 Lock Manager

任何无效的锁操作会使事务的状态变为ABORTED同时抛出异常

一个失败的锁操作（例如死锁）不会导致异常，但是在这一次lock request中，LM应该返回false

`LockTable()`和`LockRow()`都是阻塞操作，它们都需要等待锁被授予后才返回。如果事务在申请锁的同时`aborted`了，那么LM不授予锁且返回`false`

对于同一个资源，可以同时授予多个互相兼容的锁，前提是按照`FIFO`来授予。

`Table`支持所有的锁操作。
`Row`不支持意向锁，尝试对`Row`申请意向锁会抛出`ATTEMPTED_INTENTION_LOCK_ON_ROW`异常

对`Row`申请锁时，必须在对应的`Table`上持有对应的锁，否则会抛出`TABLE_LOCK_NOT_PRESENT`异常

在申请锁时，如果已经持有了同样的锁，那么直接返回true。否则升级锁：

1. 检查升级的前置条件
2. 释放当前的锁，保留升级的位置
3. 等到新锁的授予

一个升级的加锁申请应该比其他等待在相同资源上的加锁申请优先级要高。

不兼容的锁升级应该抛出`INCOMPATIBLE_UPGRADE`异常

同时多个锁升级会抛出`UPGRADE_CONFLICT`异常

对没有持锁的资源解锁会抛出`ATTEMPTED_UNLOCK_BUT_NO_LOCK_HELD`异常

解锁一个`table`的前提是该事务在对应的`row`中没有持锁，否则抛出`TABLE_UNLOCKED_BEFORE_UNLOCKING_ROWS`异常

解锁后对等待该资源授予锁的事务进行一次授予。

只有释放`S`或者`X`锁才会改变事务状态



## task #2 Deadlock Detection

每次死锁检测线程唤醒后，都重新构建`waits-for graph`

死锁检测算法每次都返回最年轻的事务`id`，设置该事务的状态为`ABORTED`

死锁检测需要打破所有`waits-for graph`中存在的循环，`ABORTED`状态的事务不参与建图

由于兼容锁的存在，一个事务可能依赖多个事务

当一个正在等待锁授予的事务被`ABORTED`后，需要将其从等待队列中移除并显式`abort`



## task #3 Concurrent Query Execution

如果`lock/unlock`操作失败，那么我们需要`abort`事务，同时撤销事务之前的写操作。我们会对每个事务都维护一个`weite set`。

如果一个`executor`获取一个锁时失败了，则抛出一个`ExecutionException`异常。

在一个事务内，可能有多个查询会访问同一资源。

