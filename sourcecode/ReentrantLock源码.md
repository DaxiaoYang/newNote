# ReentrantLock源码

[TOC]

## 创建

```java
Lock lock = new ReentrantLock();
// 默认选择非公平的同步器 因为吞吐量更高 且公平锁在获取锁的时间和避免线程饥饿情况上并没有更好的表现
// 公平锁即按照加锁先后顺序分配锁
sync = new NonfairSync();
```



## 加锁

```java
lock.lock()

// 1.执行NonfairSync的lock方法
final void lock() {
  // 与hashmap里面的思想一样 都是先试一个fast path
  if (compareAndSetState(0, 1))
    // state:0表示当前没有线程获取锁
    // state 0 -> 1 表示当前线程成功加锁
    // 设置锁的拥有者为当前线程
    setExclusiveOwnerThread(Thread.currentThread());
  else
   	// 加锁失败了 或者不是当前线程第一次加锁
    acquire(1);
}

// 2.执行AQS的acquire(1)
public final void acquire(int arg) {
  if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}

// 2.1 tryAcquire先尝试获取 
// 2.1.1.NonFairSync 会无视之前排队的线程（非公平的体现）直接尝试CAS
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    if (compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  // 可重入的体现 如果加锁线程就是拥有锁的线程 state++
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
// 2.1.2.FairSync 会判断第一个元素（head.next.thread）是不是当前线程
protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    // 关键在hasQueuedPredecessors
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}

// 3.尝试获取失败 AQS addWaiter 将线程将加入同步队列尾部（双向链表实现）

private Node addWaiter(Node mode) {
  			// 一个链表结点就是一个代表等待锁的线程
        Node node = new Node(Thread.currentThread(), mode);
       // 尝试CAS 使得 node.prev = tail;
			 // tail.next = node; 顺序不一样 但是逻辑是一样的
  		 // tail = node;
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
  			// 说明链表还未初始化 或者发送竞争 CAS失败
        enq(node);
        return node;
    }

// 3.1 for loop中CAS tail 链表初始化逻辑也在这
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
          	// 初始化 head = new Node() 第一个节点是虚拟结点
            if (t == null) { // Must initialize
              	// 只允许一个成功 
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
              	// 一样的逻辑 node.prev = tail;
              	// tail.next = node;
              	// tail = node;
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

// 4. 对已经加入到链表中的结点的操作
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
              	// 如果没有等待的线程 且抢占了锁
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    // head = head.next
                  	setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
              	// shouldParkAfterFailedAcquire 改一下前驱结点的waitStatus 将其改为SIGNAL表明自己需要唤醒
              	// 然后再执行一轮循环 再尝试抢占一下
              	// parkAndCheckInterrupt这里会阻塞住自己
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```



## 释放锁

```java
// ReentrantLock
public void unlock() {
        sync.release(1);
}

// AQS
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
          	// 释放成功之后 将等待的线程 即虚拟头结点的后面一个结点 唤醒其中的线程 
          // h.waitStatus == -1(SIGNAL) 表示后继结点中的线程需要唤醒
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
}

// AQS 1 -> 0 因为只有一个线程能执行这个操作 所以没有竞争 所以不用CAS
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
}

// 唤醒线程
private void unparkSuccessor(Node node) {
        Node s = node.next;
  			// 如果后继结点不存在或者是thread cancelled掉了
        if (s == null || s.waitStatus > 0) {
            s = null;
          	// 从后向前遍历 找到真正的后继结点 
          	// 因为enq方法中是先 node.prev = tail 不管哪一个执行tail = node成功 程序都是正确的
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
  			// 唤醒排在第一个的且需要唤醒的线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```





## 等待某个条件变量

```java
// AQS中的ConditionObject new ConditionObject()
// 就是一个条件队列 单链表实现 
// Node还是那个Node 但是mode = CONDITION next指针用nextWaiter表示 
// 最初只有head 和 tail
Condition condition = lock.newCondition();

// 获取了锁之后才能执行的操作 相当于临界区
lock.lock();
condition.await();
lock.unlock();

// await()方法 方法返回的时候 当前线程还是持有锁的
public final void await() throws InterruptedException {
            // 将表示当前线程的结点加入条件队列
            Node node = addConditionWaiter();
  					// 获取当前state的数值 并set state = 0表示释放锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
  					// 当还没有转移到同步队列时
            while (!isOnSyncQueue(node)) {
              	// 阻塞当前线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
  					// acquireQueued帮助放到同步队列的线程 获取锁 0 -> state(之前保存的状态)
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }


  private Node addConditionWaiter() {
    Node t = lastWaiter;
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 也是懒加载 
    if (t == null)
      firstWaiter = node;
    else
      t.nextWaiter = node;
    lastWaiter = node;
    return node;
  }
```



## 释放某个条件变量

```java
lock.lock();
condition.signalAll();
lock.unlock();

// signal会唤醒等待时间最长的线程
public final void signalAll() {
  // 获取条件队列队头
  Node first = firstWaiter;
  if (first != null)
    doSignalAll(first);
}



private void doSignalAll(Node first) {
  lastWaiter = firstWaiter = null;
  do {
    // 将所有条件队列中的结点 都加入到同步队列中
    Node next = first.nextWaiter;
    first.nextWaiter = null;
    transferForSignal(first);
    first = next;
  } while (first != null);
}

final boolean transferForSignal(Node node) {
  // waitStatus: -2 -> 0
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
         return false;
  			// 加入到同步队列中
  			// p为node的前驱
        Node p = enq(node);
        int ws = p.waitStatus;
  			// 前驱结点设置为 waitStatus -> Node.SIGNAL 表明当前线程需要被唤醒 以供后面的unlock
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

// 在这里会从同步队列上 唤醒 一个从等待队列中移动过来的结点（线程）
lock.unlock();
```











