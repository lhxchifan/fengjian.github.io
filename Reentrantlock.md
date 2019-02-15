# Reentrantlock

## Sync(分别被公平锁和不公平锁继承) 

> ```java
>  abstract static class Sync extends AbstractQueuedSynchronizer {
>         private static final long serialVersionUID = -5179523762034025860L;
> 
>         /**
>          * Performs {@link Lock#lock}. The main reason for subclassing
>          * is to allow fast path for nonfair version.
>          */
>         abstract void lock();
> 
>         /**
>          * Performs non-fair tryLock.  tryAcquire is implemented in
>          * subclasses, but both need nonfair try for trylock method.
>          */
>         final boolean nonfairTryAcquire(int acquires) {
>             final Thread current = Thread.currentThread();
>             int c = getState();
>             if (c == 0) {
>                 if (compareAndSetState(0, acquires)) {
>                     setExclusiveOwnerThread(current);
>                     return true;
>                 }
>             }
>             else if (current == getExclusiveOwnerThread()) {
>                 int nextc = c + acquires;
>                 if (nextc < 0) // overflow
>                     throw new Error("Maximum lock count exceeded");
>                 setState(nextc);
>                 return true;
>             }
>             return false;
>         }
> 
>         protected final boolean tryRelease(int releases) {
>             int c = getState() - releases;
>             if (Thread.currentThread() != getExclusiveOwnerThread())
>                 throw new IllegalMonitorStateException();
>             boolean free = false;
>             if (c == 0) {
>                 free = true;
>                 setExclusiveOwnerThread(null);
>             }
>             setState(c);
>             return free;
>         }
> 
>         protected final boolean isHeldExclusively() {
>             // While we must in general read state before owner,
>             // we don't need to do so to check if current thread is owner
>             return getExclusiveOwnerThread() == Thread.currentThread();
>         }
> 
>         final ConditionObject newCondition() {
>             return new ConditionObject();
>         }
> 
>         // Methods relayed from outer class
> 
>         final Thread getOwner() {
>             return getState() == 0 ? null : getExclusiveOwnerThread();
>         }
> 
>         final int getHoldCount() {
>             return isHeldExclusively() ? getState() : 0;
>         }
> 
>         final boolean isLocked() {
>             return getState() != 0;
>         }
> 
>         /**
>          * Reconstitutes the instance from a stream (that is, deserializes it).
>          */
>         private void readObject(java.io.ObjectInputStream s)
>             throws java.io.IOException, ClassNotFoundException {
>             s.defaultReadObject();
>             setState(0); // reset to unlocked state
>         }
> ```
>
> ```java
> public final void acquire(int arg) {
>         if (!tryAcquire(arg) &&
>             acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
>             selfInterrupt();
>     }
> ```
>
> ```java
> /**
>      * Acquires in exclusive uninterruptible mode for thread already in
>      * queue. Used by condition wait methods as well as acquire.
>      *
>      * @param node the node
>      * @param arg the acquire argument
>      * @return {@code true} if interrupted while waiting
>      */ 
> final boolean acquireQueued(final Node node, int arg) {
>         boolean failed = true;
>         try {
>             boolean interrupted = false;
>             for (;;) {
>                 final Node p = node.predecessor();
>                 if (p == head && tryAcquire(arg)) {
>                     setHead(node);
>                     p.next = null; // help GC
>                     failed = false;
>                     return interrupted;
>                 }
>                 if (shouldParkAfterFailedAcquire(p, node) &&
>                     parkAndCheckInterrupt())
>                     interrupted = true;
>             }
>         } finally {
>             if (failed)
>                 cancelAcquire(node);
>         }
>     }
> ```
>
> ```java
> private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
>         int ws = pred.waitStatus;
>         if (ws == Node.SIGNAL)
>             /*
>              * This node has already set status asking a release
>              * to signal it, so it can safely park.
>              */
>             return true;
>         if (ws > 0) {
>             /*
>              * Predecessor was cancelled. Skip over predecessors and
>              * indicate retry.
>              */
>             do {
>                 node.prev = pred = pred.prev;
>             } while (pred.waitStatus > 0);
>             pred.next = node;
>         } else {
>             /*
>              * waitStatus must be 0 or PROPAGATE.  Indicate that we
>              * need a signal, but don't park yet.  Caller will need to
>              * retry to make sure it cannot acquire before parking.
>              */
>             compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
>         }
>         return false;
>     }
> ```
>
> ```java
> /**
>      * Creates and enqueues node for current thread and given mode.
>      *
>      * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
>      * @return the new node
>      */
> private Node addWaiter(Node mode) {
>         Node node = new Node(Thread.currentThread(), mode);
>         // Try the fast path of enq; backup to full enq on failure
>         Node pred = tail;
>         if (pred != null) {
>             node.prev = pred;
>             if (compareAndSetTail(pred, node)) {
>                 pred.next = node;
>                 return node;
>             }
>         }
>         enq(node);
>         return node;
>     }
> ```
>
>
>
> 1. 调用lock,会调用下面两种锁的lock方法,然后会调用`acquire`
> 2. 如果tryAcquire成功,则获取到锁
> 3. 如果tryAcquire失败,则调用addWaiter新建一个node,把node加入等待队列的队尾
> 4. 调用acquireQueued
>    1. 取node的pre节点,如果pre是头节点,再次尝试acquire(猜测是可能很快能获取锁,所以再试一试),如果失败则调用shouldParkAfterFailedAcquire
>    2. 调用shouldParkAfterFailedAcquire,判断是否能park
>    3. park,直到被打断
>
>

## NonfairSync

> ```java
> static final class NonfairSync extends Sync {
>         private static final long serialVersionUID = 7316153563782823691L;
> 
>         /**
>          * Performs lock.  Try immediate barge, backing up to normal
>          * acquire on failure.
>          */
>         final void lock() {
>             if (compareAndSetState(0, 1))
>                 setExclusiveOwnerThread(Thread.currentThread());
>             else
>                 acquire(1);
>         }
> 
>         protected final boolean tryAcquire(int acquires) {
>             return nonfairTryAcquire(acquires);
>         }
>     }
> ```
>
> 1. 不公平锁,先cas,修改state,修改成功则设置exclusiveOwnerThread为当前thread,失败则acquire
> 2. nonfair trylock
>    1. 首先,如果state == 0, cas 设置state为1, 然后设置exclusiveOwnerThread为当前线程
>    2. 如果当前线程已经拥有这个锁,设置next state 为state + acquires
> 3. 否则加入等待队列

## FairSync

> ```java
> static final class FairSync extends Sync {
>         private static final long serialVersionUID = -3000897897090466540L;
> 
>         final void lock() {
>             acquire(1);
>         }
> 
>         /**
>          * Fair version of tryAcquire.  Don't grant access unless
>          * recursive call or no waiters or is first.
>          */
>         protected final boolean tryAcquire(int acquires) {
>             final Thread current = Thread.currentThread();
>             int c = getState();
>             if (c == 0) {
>                 if (!hasQueuedPredecessors() &&
>                     compareAndSetState(0, acquires)) {
>                     setExclusiveOwnerThread(current);
>                     return true;
>                 }
>             }
>             else if (current == getExclusiveOwnerThread()) {
>                 int nextc = c + acquires;
>                 if (nextc < 0)
>                     throw new Error("Maximum lock count exceeded");
>                 setState(nextc);
>                 return true;
>             }
>             return false;
>         }
>     }
> ```
>
> 1. 公平锁获取锁时直接acquire
> 2. acquire首先判断当前state, 如果为0, 先判断当前有没有存在等待的节点,没有的话,cas设置state,然后设置exclusiveOwnerThread
> 3. 如果当前线程是重入,则设置state + acquires
> 4. 否则加入等待队列

## Release

> ```java
>  public final boolean release(int arg) {
>         if (tryRelease(arg)) {
>             Node h = head;
>             if (h != null && h.waitStatus != 0)
>                 unparkSuccessor(h);
>             return true;
>         }
>         return false;
>     }
> ```
>
>

> ```java
> private void unparkSuccessor(Node node) {
>         /*
>          * If status is negative (i.e., possibly needing signal) try
>          * to clear in anticipation of signalling.  It is OK if this
>          * fails or if status is changed by waiting thread.
>          */
>         int ws = node.waitStatus;
>         if (ws < 0)
>             compareAndSetWaitStatus(node, ws, 0);
> 
>         /*
>          * Thread to unpark is held in successor, which is normally
>          * just the next node.  But if cancelled or apparently null,
>          * traverse backwards from tail to find the actual
>          * non-cancelled successor.
>          */
>         Node s = node.next;
>         if (s == null || s.waitStatus > 0) {
>             s = null;
>             for (Node t = tail; t != null && t != node; t = t.prev)
>                 if (t.waitStatus <= 0)
>                     s = t;
>         }
>         if (s != null)
>             LockSupport.unpark(s.thread);
>     }
> ```
>
> 1. release成功,则取出head节点
> 2. 设置waitStatus为0
> 3. 一般默认继承者是next,如果next不能继承,则从队列尾向前找个继承者