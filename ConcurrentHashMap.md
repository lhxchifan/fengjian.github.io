# ConcurrentHashMap

## 结构

> 1. `Node` 表示hash桶中链表节点
> 2. **内部用了`table`和`nextTable`**
> 3. `Segment`好像是为了兼容以前的序列化
> 4. `ForwardingNode` 用于转移操作时插入在头结点
> 5. `ReservationNode` 用于计算`computeIfAbsent`和`compute`
> 6. `TreeNode` 表示使用在`Treebins`中的节点
> 7. `TreeBin`用于表示一组`TreeNodes`, 并且维持一组读写锁
> 8. `TableStack`遍历table时，记录table的length，current index
> 9. `Traverser`各种iterator的基类
> 10. `MapEntry`暴露给Iterator的entry

## 关键成员变量

> 1. `sizeCtl`用于控制table的初始化和resize

## 初始化

> 1. 啥也不干，或者设置初始capacity，设置sizeCtl

## PUT

> 1. key/value null -> throws npe
> 2. 计算hash值
> 3. 如果`table`为null，初始化table
> 4. 用table的size - 1 & hash 调用tabAt去取tab的位置，如果为null，直接new一个Node，cas赋值到该位置
> 5. **如果当前tab不是null，并且hash为MOVED，调用`helpTransfer` ??? **
> 6. 如果都不是，锁住当前tab，
>    1. 如果当前tab没有被修改，
>    2. 如果hash值大于0，遍历tab，在末尾添加新node或者覆盖已经存在的key
>    3. 如果hash值不大于0，并且f是`TreeBin`，调用`putTreeVal`
> 7. 如果当前这个链表的节点数大于8,把链表变成一个树,即 TreeBins
> 8. 调用`addCount`,增加当前map的size

## addCount

> ```java
> private final void addCount(long x, int check) {
>     CounterCell[] as; long b, s;
>     if ((as = counterCells) != null ||
>         !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
>         CounterCell a; long v; int m;
>         boolean uncontended = true;
>         if (as == null || (m = as.length - 1) < 0 ||
>             (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
>             !(uncontended =
>               U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
>             fullAddCount(x, uncontended);
>             return;
>         }
>         if (check <= 1)
>             return;
>         s = sumCount();
>     }
>     if (check >= 0) {
>         Node<K,V>[] tab, nt; int n, sc;
>         while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
>                (n = tab.length) < MAXIMUM_CAPACITY) {
>             int rs = resizeStamp(n);
>             if (sc < 0) {
>                 if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
>                     sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
>                     transferIndex <= 0)
>                     break;
>                 if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
>                     transfer(tab, nt);
>             }
>             else if (U.compareAndSwapInt(this, SIZECTL, sc,
>                                          (rs << RESIZE_STAMP_SHIFT) + 2))
>                 transfer(tab, null);
>             s = sumCount();
>         }
>     }
> }
> ```

> 1. 如果`counterCells`不为`null`,或者cas baseCount = baseCount + 1失败
>
>    1. 如果`counterCells`为`null`,或者counterCells的length小于1,或者取`ThreadLocalRandom.getProbe()&length -1`的位置的CounterCell为null,或者cas CounterCell的value失败,调用`fullAddCount`, 跟LongAdder的add一个原理,原子加1
>    2. 如果check < 1,返回.`if <= 1 only check if uncontended`
>
> 2. 如果check >= 0,
>
>    1. 取s为所有counterCells的value之和,如果`s > sizeCtl && table != null && table的length小于最大的大小,则resize`
>
>    2. 记录下resize 的stamp,`Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1))`,取当前length的前缀0个数或上1 << 15 ???
>
>    3. 如果sc < 0, 即正在初始化或者resize,如果-1,则是正在初始化,如果-(1 + n),表示有n个线程在resize
>
>       1.  检查扩容标志,如果扩容标志不相等,说明扩容已经完成了,
>       2. 以及一些其他检查是否在resize的标志
>       3. 如果不需要协助resize, break,或者调用`transfer`
>
>    4. 如果sc>=0, 说明不在resize, cas 一个前16位为rs(resize stamp)扩容标志,后16位为2的值到sizeCtl
>
>       1. 调用`transfer`
>

## Transfer

> ```java
> private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
>         int n = tab.length, stride;
>         if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
>             stride = MIN_TRANSFER_STRIDE; // subdivide range
>         if (nextTab == null) {            // initiating
>             try {
>                 @SuppressWarnings("unchecked")
>                 Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
>                 nextTab = nt;
>             } catch (Throwable ex) {      // try to cope with OOME
>                 sizeCtl = Integer.MAX_VALUE;
>                 return;
>             }
>             nextTable = nextTab;
>             transferIndex = n;
>         }
>         int nextn = nextTab.length;
>         ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
>         boolean advance = true;
>         boolean finishing = false; // to ensure sweep before committing nextTab
>         for (int i = 0, bound = 0;;) {
>             Node<K,V> f; int fh;
>             while (advance) {
>                 int nextIndex, nextBound;
>                 if (--i >= bound || finishing)
>                     advance = false;
>                 else if ((nextIndex = transferIndex) <= 0) {
>                     i = -1;
>                     advance = false;
>                 }
>                 else if (U.compareAndSwapInt
>                          (this, TRANSFERINDEX, nextIndex,
>                           nextBound = (nextIndex > stride ?
>                                        nextIndex - stride : 0))) {
>                     bound = nextBound;
>                     i = nextIndex - 1;
>                     advance = false;
>                 }
>             }
>             if (i < 0 || i >= n || i + n >= nextn) {
>                 int sc;
>                 if (finishing) {
>                     nextTable = null;
>                     table = nextTab;
>                     sizeCtl = (n << 1) - (n >>> 1);
>                     return;
>                 }
>                 if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
>                     if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
>                         return;
>                     finishing = advance = true;
>                     i = n; // recheck before commit
>                 }
>             }
>             else if ((f = tabAt(tab, i)) == null)
>                 advance = casTabAt(tab, i, null, fwd);
>             else if ((fh = f.hash) == MOVED)
>                 advance = true; // already processed
>             else {
>                 synchronized (f) {
>                     if (tabAt(tab, i) == f) {
>                         Node<K,V> ln, hn;
>                         if (fh >= 0) {
>                             int runBit = fh & n;
>                             Node<K,V> lastRun = f;
>                             for (Node<K,V> p = f.next; p != null; p = p.next) {
>                                 int b = p.hash & n;
>                                 if (b != runBit) {
>                                     runBit = b;
>                                     lastRun = p;
>                                 }
>                             }
>                             if (runBit == 0) {
>                                 ln = lastRun;
>                                 hn = null;
>                             }
>                             else {
>                                 hn = lastRun;
>                                 ln = null;
>                             }
>                             for (Node<K,V> p = f; p != lastRun; p = p.next) {
>                                 int ph = p.hash; K pk = p.key; V pv = p.val;
>                                 if ((ph & n) == 0)
>                                     ln = new Node<K,V>(ph, pk, pv, ln);
>                                 else
>                                     hn = new Node<K,V>(ph, pk, pv, hn);
>                             }
>                             setTabAt(nextTab, i, ln);
>                             setTabAt(nextTab, i + n, hn);
>                             setTabAt(tab, i, fwd);
>                             advance = true;
>                         }
>                         else if (f instanceof TreeBin) {
>                             TreeBin<K,V> t = (TreeBin<K,V>)f;
>                             TreeNode<K,V> lo = null, loTail = null;
>                             TreeNode<K,V> hi = null, hiTail = null;
>                             int lc = 0, hc = 0;
>                             for (Node<K,V> e = t.first; e != null; e = e.next) {
>                                 int h = e.hash;
>                                 TreeNode<K,V> p = new TreeNode<K,V>
>                                     (h, e.key, e.val, null, null);
>                                 if ((h & n) == 0) {
>                                     if ((p.prev = loTail) == null)
>                                         lo = p;
>                                     else
>                                         loTail.next = p;
>                                     loTail = p;
>                                     ++lc;
>                                 }
>                                 else {
>                                     if ((p.prev = hiTail) == null)
>                                         hi = p;
>                                     else
>                                         hiTail.next = p;
>                                     hiTail = p;
>                                     ++hc;
>                                 }
>                             }
>                             ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
>                                 (hc != 0) ? new TreeBin<K,V>(lo) : t;
>                             hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
>                                 (lc != 0) ? new TreeBin<K,V>(hi) : t;
>                             setTabAt(nextTab, i, ln);
>                             setTabAt(nextTab, i + n, hn);
>                             setTabAt(tab, i, fwd);
>                             advance = true;
>                         }
>                     }
>                 }
>             }
>         }
>     }
> ```
>
> 1. 设置步长
> 2. 如果没有新的tab的地址,初始化一个一倍大小的新tabs,transferIndex设置为n,即之前的size
> 3. 新建一个ForwardingNode fwd
> 4. 先取边界,取当前处理的index,边界是index - 步长,index相当于是取n-1,然后倒序一个一个处理
> 5. 如果当前index的tab为null, 把index的tab设置为fwd,即tab数组的最后一个tab为fwd
> 6. 如果index的tab不是null,且hash值不是moved, 锁住这个tab
>    1. 如果当前tab的hash值大于0,runBit为hash值&n(**之前未扩容的时候,取的是n-1**),对tab的链表子节点,找到一个最后的一个不与前面相等的节点,如果runbit是0,则取出来的节点为尾节点,否则当做头节点,
>    2. 将tab中的节点,以`(ph & n) == 0`为界,分为两个链表,为0的设置到新tab的i位置,不为0的设置到i+n位置,把老tab的位置i设置为fwd
>    3. 如果tab是一个TreeBin,同理拆分红黑树

## putTreeVal

> 1. 如果root是null,新建一个treenode
> 2. 如果插入节点的hash值小于当前节点的hash值,dir = -1,反之,dir =1
> 3. 如果hash值相等,并且key相等,直接返回
> 4. **如果key不是comparable的,或者如果当前节点key的类型和待添加的key类型不一致???**
> 5. 二叉查找树,找到

## initTable

> 1. 当`table`为`null`时，如果`sizeCtl < 0`,说明正在初始化，当前线程yield
> 2. 比较`sizeCtl` 和 `sc`的值，cas -1到`sizeCtl`
> 3. 如果table为null， new一个sc的size的Node数组
> 4. sc = sc - sc >>> 2, 把sc赋值给sizeCtl，返回table

## TabAt

> 1. `ABASE = U.arrayBaseOffset(ak);` ABASE表示array的起始地址
>
>    ```c++
>    UNSAFE_ENTRY(jint, Unsafe_ArrayBaseOffset(JNIEnv *env, jobject unsafe, jclass acls))
>      UnsafeWrapper("Unsafe_ArrayBaseOffset");
>      int base, scale;
>      getBaseAndScale(base, scale, acls, CHECK_0);
>      return field_offset_from_byte_offset(base);
>    UNSAFE_END
>    
>    static void getBaseAndScale(int& base, int& scale, jclass acls, TRAPS) {
>      if (acls == NULL) {
>        THROW(vmSymbols::java_lang_NullPointerException());
>      }
>      oop      mirror = JNIHandles::resolve_non_null(acls);
>      Klass* k      = java_lang_Class::as_Klass(mirror);
>      if (k == NULL || !k->oop_is_array()) {
>        THROW(vmSymbols::java_lang_InvalidClassException());
>      } else if (k->oop_is_objArray()) {
>        base  = arrayOopDesc::base_offset_in_bytes(T_OBJECT);
>        scale = heapOopSize;
>      } else if (k->oop_is_typeArray()) {
>        TypeArrayKlass* tak = TypeArrayKlass::cast(k);
>        base  = tak->array_header_in_bytes();
>        assert(base == arrayOopDesc::base_offset_in_bytes(tak->element_type()), "array_header_size semantics ok");
>        scale = (1 << tak->log2_element_size());
>      } else {
>        ShouldNotReachHere();
>      }
>    }
>    ```
>
> 2. `int scale = U.arrayIndexScale(ak);`, `ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);`
>
>    ```c++
>    UNSAFE_ENTRY(jint, Unsafe_ArrayIndexScale(JNIEnv *env, jobject unsafe, jclass acls))
>      UnsafeWrapper("Unsafe_ArrayIndexScale");
>      int base, scale;
>      getBaseAndScale(base, scale, acls, CHECK_0);
>      // This VM packs both fields and array elements down to the byte.
>      // But watch out:  If this changes, so that array references for
>      // a given primitive type (say, T_BOOLEAN) use different memory units
>      // than fields, this method MUST return zero for such arrays.
>      // For example, the VM used to store sub-word sized fields in full
>      // words in the object layout, so that accessors like getByte(Object,int)
>      // did not really do what one might expect for arrays.  Therefore,
>      // this function used to report a zero scale factor, so that the user
>      // would know not to attempt to access sub-word array elements.
>      // // Code for unpacked fields:
>      // if (scale < wordSize)  return 0;
>    
>      // The following allows for a pretty general fieldOffset cookie scheme,
>      // but requires it to be linear in byte offset.
>      return field_offset_from_byte_offset(scale) - field_offset_from_byte_offset(0);
>    UNSAFE_END
>    ```
>
> 3. 获取tab
>
>    ```c++
>    UNSAFE_ENTRY(jobject, Unsafe_GetObjectVolatile(JNIEnv *env, jobject unsafe, jobject obj, jlong offset))
>      UnsafeWrapper("Unsafe_GetObjectVolatile");
>      oop p = JNIHandles::resolve(obj);
>      void* addr = index_oop_from_field_offset_long(p, offset);
>      volatile oop v;
>      if (UseCompressedOops) {
>        volatile narrowOop n = *(volatile narrowOop*) addr;
>        (void)const_cast<oop&>(v = oopDesc::decode_heap_oop(n));
>      } else {
>        (void)const_cast<oop&>(v = *(volatile oop*) addr);
>      }
>      OrderAccess::acquire();
>      return JNIHandles::make_local(env, v);
>    UNSAFE_END
>    ```