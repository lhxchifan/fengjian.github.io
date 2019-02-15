# CopyOnWriteArrayList

## structure

> ```java
> final transient ReentrantLock lock = new ReentrantLock();
> 
> /** The array, accessed only via getArray/setArray. */
> private transient volatile Object[] array;
> ```
>
> 1. only contains a lock and an array of Object

## add

> ```java
> /**
>      * Appends the specified element to the end of this list.
>      *
>      * @param e element to be appended to this list
>      * @return {@code true} (as specified by {@link Collection#add})
>      */
>     public boolean add(E e) {
>         final ReentrantLock lock = this.lock;
>         lock.lock();
>         try {
>             Object[] elements = getArray();
>             int len = elements.length;
>             Object[] newElements = Arrays.copyOf(elements, len + 1);
>             newElements[len] = e;
>             setArray(newElements);
>             return true;
>         } finally {
>             lock.unlock();
>         }
>     }
> ```
>
> 1. first, lock
> 2. get now elements, get length, calculate new length of array(length + 1)
> 3.  copy all elements to a new array, and set last element as added object
> 4. set array and return

## Conclusion

> 1. same as arraylist, but cost much on write
> 2. read operation will not lock the array, read is fast