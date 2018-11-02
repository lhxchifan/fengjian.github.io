# MarkOopDesc

---

tags:

- java
- hotspot jvm

categories:

- hotspot jvm

### 怪异的用法

> ```c++
> class markOopDesc: public oopDesc {
>  private:
>   // Conversion
>   uintptr_t value() const { return (uintptr_t) this; }
> }
> ```
>
> `markOop` 实际上是用它的指针来存实际的bitfields，非常怪异

### 协议定义

> ```c++
> // Bit-format of an object header (most significant first, big endian layout below):
> //
> //  32 bits:
> //  --------
> //             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
> //             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
> //             size:32 ------------------------------------------>| (CMS free block)
> //             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
> //
> //  64 bits:
> //  --------
> //  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
> //  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
> //  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
> //  size:64 ----------------------------------------------------->| (CMS free block)
> //
> //  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
> //  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
> //  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
> //  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
> ```

###  状态

> ```c++
> //    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
> //    [0           | epoch | age | 1 | 01]       lock is anonymously biased
> //
> //  - the two lock bits are used to describe three states: locked/unlocked and monitor.
> //
> //    [ptr             | 00]  locked             ptr points to real header on stack
> //    [header      | 0 | 01]  unlocked           regular object header
> //    [ptr             | 10]  monitor            inflated lock (header is wapped out)
> //    [ptr             | 11]  marked             used by markSweep to mark an object
> //                                               not valid at any other time
> ```
>
> ![markoop](/Users/liuhongxuan/notes/picture/markoop.png)