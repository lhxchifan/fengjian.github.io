# ArrayBlockingQueue

## 前置参考 

[参考LinkedBlockingQueue](LinkedBlockingQueue.md)

## 结构

> ```java
> /** The queued items */
>     final Object[] items;
> 
>     /** items index for next take, poll, peek or remove */
>     int takeIndex;
> 
>     /** items index for next put, offer, or add */
>     int putIndex;
> 
>     /** Number of elements in the queue */
>     int count;
> 
>     /*
>      * Concurrency control uses the classic two-condition algorithm
>      * found in any textbook.
>      */
> 
>     /** Main lock guarding all access */
>     final ReentrantLock lock;
> 
>     /** Condition for waiting takes */
>     private final Condition notEmpty;
> 
>     /** Condition for waiting puts */
>     private final Condition notFull;
> ```
>
> 1. 只包含一个锁
> 2. 同样包含两个condition

## take

> ```java
> public E take() throws InterruptedException {
>         final ReentrantLock lock = this.lock;
>         lock.lockInterruptibly();
>         try {
>             while (count == 0)
>                 notEmpty.await();
>             return dequeue();
>         } finally {
>             lock.unlock();
>         }
>     }
> ```
>
> 1. 锁住lock
> 2. 判断count == 0,则notEmpty wait
> 3. 出队一个item

## put

> ```java
> public void put(E e) throws InterruptedException {
>         checkNotNull(e);
>         final ReentrantLock lock = this.lock;
>         lock.lockInterruptibly();
>         try {
>             while (count == items.length)
>                 notFull.await();
>             enqueue(e);
>         } finally {
>             lock.unlock();
>         }
>     }
> ```
>
> 1. 锁住lock
> 2. 判断count是否等于items的长度,如果是,则notFull wait
> 3. 入队