# go 并发之 Mutex&WaitGroup

## Mutex

锁，sync 包下，提供 Lock 和 Unlock 方法，所有在 Lock 和 Unlock 之间的代码都只能由一个 goroutine 执行，避免竞态条件。如果一个协程已经持有了锁（Lock），其他协程试图获得该锁时会被阻塞，直到 Mutex 解除锁定（Unlock）。

可以用容量为 1 的 channel 处理竞态条件。例子：todo

## RWMutex

读写锁。RWMutex 基于 Mutex，在 Mutex 的基础上增加了读、写的信号量，使用了类似引用计数的读锁数量。Lock 和 Unlock 方法用于申请和释放写锁，RLock 和 RUnlock 方法用于申请和释放读锁。

读写锁适合读多写少的场景。

读锁与读锁兼容，读锁与写锁互斥，写锁和写锁互斥。在锁释放后才可以继续申请互斥的锁：

* 可以同时申请多个读锁。
* 有读锁时申请写锁将阻塞，有写锁时申请读锁将阻塞。
* 只要有写锁，申请读锁和写锁都将阻塞。

无论时 Mutex 还是 RWMutex 都不会和 goroutine 进行关联，可以在一个 goroutine 申请锁，在另一个 goroutine 释放锁。

## 如何选择 Mutex，RWMutex 和 channel

作并发控制你喜欢用哪个，哪个快，为什么

## WaitGroup