# LinkedBlockingQueue

### 

## 结构

> ```java
> static class Node<E> {
>         E item;
> 
>         /**
>          * One of:
>          * - the real successor Node
>          * - this Node, meaning the successor is head.next
>          * - null, meaning there is no successor (this is the last node)
>          */
>         Node<E> next;
> 
>         Node(E x) { item = x; }
>     }
> ```
>
> ```
>     /** Lock held by take, poll, etc */
>     private final ReentrantLock takeLock = new ReentrantLock();
> 
>     /** Wait queue for waiting takes */
>     private final Condition notEmpty = takeLock.newCondition();
> 
>     /** Lock held by put, offer, etc */
>     private final ReentrantLock putLock = new ReentrantLock();
> 
>     /** Wait queue for waiting puts */
>     private final Condition notFull = putLock.newCondition();
> ```
>
>

> 1. 声明了一个链表节点
> 2. 声明了两个可重入锁, 取了锁的condition

## enqueue

> ```java
> /**
>      * Links node at end of queue.
>      *
>      * @param node the node
>      */
>     private void enqueue(Node<E> node) {
>         // assert putLock.isHeldByCurrentThread();
>         // assert last.next == null;
>         last = last.next = node;
>     }
> ```
>
> 1. 这里重点关注入队时,仅将元素放入队尾

## dequeue

> ```java
> /**
>      * Removes a node from head of queue.
>      *
>      * @return the node
>      */
>     private E dequeue() {
>         // assert takeLock.isHeldByCurrentThread();
>         // assert head.item == null;
>         Node<E> h = head;
>         Node<E> first = h.next;
>         h.next = h; // help GC
>         head = first;
>         E x = first.item;
>         first.item = null;
>         return x;
>     }
> ```
>
> 1. 这里注意,出队时,仅从头部拿出一个item,不操作last
> 2. **为什么可以用两个锁一个锁住头 一个锁住尾呢,因为初始size为0时,head和last指向同一个空的node,此时只能put,不能take,后续head和last分离,则不会操作同一个指针**

## take

> ```java
> public E take() throws InterruptedException {
>         E x;
>         int c = -1;
>         final AtomicInteger count = this.count;
>         final ReentrantLock takeLock = this.takeLock;
>         takeLock.lockInterruptibly();
>         try {
>             while (count.get() == 0) {
>                 notEmpty.await();
>             }
>             x = dequeue();
>             c = count.getAndDecrement();
>             if (c > 1)
>                 notEmpty.signal();
>         } finally {
>             takeLock.unlock();
>         }
>         if (c == capacity)
>             signalNotFull();
>         return x;
>     }
> ```
>
> 1. 锁住takelock
> 2. 如果当前count为0,则notEmpty condition等待
> 3. 否则取出一个
> 4. 如果剩余的数量大于1,则notify notEmpty
> 5. 解锁takelock
> 6. 如果当前c等于容量,notify notFull**(这里为什么如果c==capacity,则notify notfull, 因为c是dequeue之前的值,现在已经-1了,所以是不满的)**

## put

> ```java
> public void put(E e) throws InterruptedException {
>         if (e == null) throw new NullPointerException();
>         // Note: convention in all put/take/etc is to preset local var
>         // holding count negative to indicate failure unless set.
>         int c = -1;
>         Node<E> node = new Node<E>(e);
>         final ReentrantLock putLock = this.putLock;
>         final AtomicInteger count = this.count;
>         putLock.lockInterruptibly();
>         try {
>             /*
>              * Note that count is used in wait guard even though it is
>              * not protected by lock. This works because count can
>              * only decrease at this point (all other puts are shut
>              * out by lock), and we (or some other waiting put) are
>              * signalled if it ever changes from capacity. Similarly
>              * for all other uses of count in other wait guards.
>              */
>             while (count.get() == capacity) {
>                 notFull.await();
>             }
>             enqueue(node);
>             c = count.getAndIncrement();
>             if (c + 1 < capacity)
>                 notFull.signal();
>         } finally {
>             putLock.unlock();
>         }
>         if (c == 0)
>             signalNotEmpty();
>     }
> ```
>
> 1. 如果放入null,throw npe
> 2. 锁住putlock
> 3. 如果当前count == capacity, 则notFull wait
> 4. 入队,count-1 并且返回之前的值给c
> 5. 如果c + 1< capacity, 意味着入队后的队列依然不满,则notify notFull
> 6. 解锁putlock
> 7. 如果入队前的capacity为0,则入队后为1,notify notEmpty

## offer

> ```java
> public boolean offer(E e) {
>         if (e == null) throw new NullPointerException();
>         final AtomicInteger count = this.count;
>         if (count.get() == capacity)
>             return false;
>         int c = -1;
>         Node<E> node = new Node<E>(e);
>         final ReentrantLock putLock = this.putLock;
>         putLock.lock();
>         try {
>             if (count.get() < capacity) {
>                 enqueue(node);
>                 c = count.getAndIncrement();
>                 if (c + 1 < capacity)
>                     notFull.signal();
>             }
>         } finally {
>             putLock.unlock();
>         }
>         if (c == 0)
>             signalNotEmpty();
>         return c >= 0;
>     }
> ```
>
> 和put的区别为,如果当前队列满了,则不放入,返回false,放入成功返回true