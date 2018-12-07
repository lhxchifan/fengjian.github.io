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
>    4. 如果当前这个链表的节点数大于8,把链表变成一个树,即 TreeBins
> 7. 

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