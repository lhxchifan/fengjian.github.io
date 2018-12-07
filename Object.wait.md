# Object.wait

### 具体实现

    1. registerNatives注册了JVM_MonitorWait
    2. 取到对应class的handle，创建JavaThreadInObjectWaitState来监控wait时间，然后调用ObjectSynchronizer::wait
```
JVM_ENTRY(void, JVM_MonitorWait(JNIEnv* env, jobject handle, jlong ms))
  JVMWrapper("JVM_MonitorWait");
  Handle obj(THREAD, JNIHandles::resolve_non_null(handle));
  JavaThreadInObjectWaitState jtiows(thread, ms != 0);
  if (JvmtiExport::should_post_monitor_wait()) {
    JvmtiExport::post_monitor_wait((JavaThread *)THREAD, (oop)obj(), ms);
  }
  ObjectSynchronizer::wait(obj, ms, CHECK);
JVM_END
```
    1. 取到handle
    2. ObjectSynchronizer::调用wait

### ObjectSynchronizer::wait
```
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  //是否使用偏向锁，默认为true
  if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  if (millis < 0) {
    TEVENT (wait - throw IAX) ;
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }
  ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  monitor->wait(millis, true, THREAD);

  /* This dummy call is in place to get around dtrace bug 6254741.  Once
     that's fixed we can uncomment the following line and remove the call */
  // DTRACE_MONITOR_PROBE(waited, monitor, obj(), THREAD);
  dtrace_waited_probe(monitor, obj, THREAD);
}
```
    1. revoke_and_rebias中会查看是否有在使用偏向锁，如果在用，撤销偏向
    2. 调用ObjectSynchronizer::inflate, 主要目的是把轻量级锁膨胀为重量级锁
    3. 调用monitor的wait


### ObjectMonitor
    // The ObjectMonitor class is used to implement JavaMonitors which have
    // transformed from the lightweight structure of the thread stack to a
    // heavy weight lock due to contention
```
//Inflate light weight monitor to heavy weight monitor
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
  // Inflate mutates the heap ...
  // Relaxing assertion for bug 6320749.
  assert (Universe::verify_in_progress() ||
          !SafepointSynchronize::is_at_safepoint(), "invariant") ;

  for (;;) {
      const markOop mark = object->mark() ;
      assert (!mark->has_bias_pattern(), "invariant") ;

      // The mark can be in one of the following states:
      // *  Inflated     - just return
      // *  Stack-locked - coerce it to inflated
      // *  INFLATING    - busy wait for conversion to complete
      // *  Neutral      - aggressively inflate the object.
      // *  BIASED       - Illegal.  We should never see this

      // CASE: inflated
      if (mark->has_monitor()) {
          ObjectMonitor * inf = mark->monitor() ;
          assert (inf->header()->is_neutral(), "invariant");
          assert (inf->object() == object, "invariant") ;
          assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
          return inf ;
      }

      // CASE: inflation in progress - inflating over a stack-lock.
      // Some other thread is converting from stack-locked to inflated.
      // Only that thread can complete inflation -- other threads must wait.
      // The INFLATING value is transient.
      // Currently, we spin/yield/park and poll the markword, waiting for inflation to finish.
      // We could always eliminate polling by parking the thread on some auxiliary list.
      if (mark == markOopDesc::INFLATING()) {
         TEVENT (Inflate: spin while INFLATING) ;
         ReadStableMark(object) ;
         continue ;
      }

      // CASE: stack-locked
      // Could be stack-locked either by this thread or by some other thread.
      //
      // Note that we allocate the objectmonitor speculatively, _before_ attempting
      // to install INFLATING into the mark word.  We originally installed INFLATING,
      // allocated the objectmonitor, and then finally STed the address of the
      // objectmonitor into the mark.  This was correct, but artificially lengthened
      // the interval in which INFLATED appeared in the mark, thus increasing
      // the odds of inflation contention.
      //
      // We now use per-thread private objectmonitor free lists.
      // These list are reprovisioned from the global free list outside the
      // critical INFLATING...ST interval.  A thread can transfer
      // multiple objectmonitors en-mass from the global free list to its local free list.
      // This reduces coherency traffic and lock contention on the global free list.
      // Using such local free lists, it doesn't matter if the omAlloc() call appears
      // before or after the CAS(INFLATING) operation.
      // See the comments in omAlloc().
    
    //存在轻量级锁
      if (mark->has_locker()) {
          ObjectMonitor * m = omAlloc (Self) ;
          // Optimistically prepare the objectmonitor - anticipate successful CAS
          // We do this before the CAS in order to minimize the length of time
          // in which INFLATING appears in the mark.
          m->Recycle();
          m->_Responsible  = NULL ;
          m->OwnerIsThread = 0 ;
          m->_recursions   = 0 ;
          m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;   // Consider: maintain by type/class

          markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
          if (cmp != mark) {
             omRelease (Self, m, true) ;
             continue ;       // Interference -- just retry
          }

          // We've successfully installed INFLATING (0) into the mark-word.
          // This is the only case where 0 will appear in a mark-work.
          // Only the singular thread that successfully swings the mark-word
          // to 0 can perform (or more precisely, complete) inflation.
          //
          // Why do we CAS a 0 into the mark-word instead of just CASing the
          // mark-word from the stack-locked value directly to the new inflated state?
          // Consider what happens when a thread unlocks a stack-locked object.
          // It attempts to use CAS to swing the displaced header value from the
          // on-stack basiclock back into the object header.  Recall also that the
          // header value (hashcode, etc) can reside in (a) the object header, or
          // (b) a displaced header associated with the stack-lock, or (c) a displaced
          // header in an objectMonitor.  The inflate() routine must copy the header
          // value from the basiclock on the owner's stack to the objectMonitor, all
          // the while preserving the hashCode stability invariants.  If the owner
          // decides to release the lock while the value is 0, the unlock will fail
          // and control will eventually pass from slow_exit() to inflate.  The owner
          // will then spin, waiting for the 0 value to disappear.   Put another way,
          // the 0 causes the owner to stall if the owner happens to try to
          // drop the lock (restoring the header from the basiclock to the object)
          // while inflation is in-progress.  This protocol avoids races that might
          // would otherwise permit hashCode values to change or "flicker" for an object.
          // Critically, while object->mark is 0 mark->displaced_mark_helper() is stable.
          // 0 serves as a "BUSY" inflate-in-progress indicator.


          // fetch the displaced mark from the owner's stack.
          // The owner can't die or unwind past the lock while our INFLATING
          // object is in the mark.  Furthermore the owner can't complete
          // an unlock on the object, either.
          markOop dmw = mark->displaced_mark_helper() ;
          assert (dmw->is_neutral(), "invariant") ;

          // Setup monitor fields to proper values -- prepare the monitor
          m->set_header(dmw) ;

          // Optimization: if the mark->locker stack address is associated
          // with this thread we could simply set m->_owner = Self and
          // m->OwnerIsThread = 1. Note that a thread can inflate an object
          // that it has stack-locked -- as might happen in wait() -- directly
          // with CAS.  That is, we can avoid the xchg-NULL .... ST idiom.
          m->set_owner(mark->locker());
          m->set_object(object);
          // TODO-FIXME: assert BasicLock->dhw != 0.

          // Must preserve store ordering. The monitor state must
          // be stable at the time of publishing the monitor address.
          guarantee (object->mark() == markOopDesc::INFLATING(), "invariant") ;
          object->release_set_mark(markOopDesc::encode(m));

          // Hopefully the performance counters are allocated on distinct cache lines
          // to avoid false sharing on MP systems ...
          if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
          TEVENT(Inflate: overwrite stacklock) ;
          if (TraceMonitorInflation) {
            if (object->is_instance()) {
              ResourceMark rm;
              tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                (void *) object, (intptr_t) object->mark(),
                object->klass()->external_name());
            }
          }
          return m ;
      }

      // CASE: neutral
      // TODO-FIXME: for entry we currently inflate and then try to CAS _owner.
      // If we know we're inflating for entry it's better to inflate by swinging a
      // pre-locked objectMonitor pointer into the object header.   A successful
      // CAS inflates the object *and* confers ownership to the inflating thread.
      // In the current implementation we use a 2-step mechanism where we CAS()
      // to inflate and then CAS() again to try to swing _owner from NULL to Self.
      // An inflateTry() method that we could call from fast_enter() and slow_enter()
      // would be useful.

      assert (mark->is_neutral(), "invariant");
      ObjectMonitor * m = omAlloc (Self) ;
      // prepare m for installation - set monitor to initial state
      m->Recycle();
      m->set_header(mark);
      m->set_owner(NULL);
      m->set_object(object);
      m->OwnerIsThread = 1 ;
      m->_recursions   = 0 ;
      m->_Responsible  = NULL ;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;       // consider: keep metastats by type/class

      if (Atomic::cmpxchg_ptr (markOopDesc::encode(m), object->mark_addr(), mark) != mark) {
          m->set_object (NULL) ;
          m->set_owner  (NULL) ;
          m->OwnerIsThread = 0 ;
          m->Recycle() ;
          omRelease (Self, m, true) ;
          m = NULL ;
          continue ;
          // interference - the markword changed - just retry.
          // The state-transitions are one-way, so there's no chance of
          // live-lock -- "Inflated" is an absorbing state.
      }

      // Hopefully the performance counters are allocated on distinct
      // cache lines to avoid false sharing on MP systems ...
      if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
      TEVENT(Inflate: overwrite neutral) ;
      if (TraceMonitorInflation) {
        if (object->is_instance()) {
          ResourceMark rm;
          tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
            (void *) object, (intptr_t) object->mark(),
            object->klass()->external_name());
        }
      }
      return m ;
  }
}
```
    1. 如果本身存在一个monitor，返回monitor
    2. 如果INFLATING，则continue
    3. 如果有轻量级锁, 把monitor 重新初始化，设置头部mark为inflating（0），设置ObjectMonitor的header为mark->displaced_mark_helper, 设置owner为mark的locker，设置object为当前object，把这个objectMonitor的markOopDesc::encode(m)设置到object的mark上（从这里可以看出来，锁都被设置在Oop的mark头部），
    4. 调用objectMonitor::wait

### omAlloc

```
ObjectMonitor * ATTR ObjectSynchronizer::omAlloc (Thread * Self) {
    // A large MAXPRIVATE value reduces both list lock contention
    // and list coherency traffic, but also tends to increase the
    // number of objectMonitors in circulation as well as the STW
    // scavenge costs.  As usual, we lean toward time in space-time
    // tradeoffs.
    const int MAXPRIVATE = 1024 ;
    for (;;) {
        ObjectMonitor * m ;

        // 1: try to allocate from the thread's local omFreeList.
        // Threads will attempt to allocate first from their local list, then
        // from the global list, and only after those attempts fail will the thread
        // attempt to instantiate new monitors.   Thread-local free lists take
        // heat off the ListLock and improve allocation latency, as well as reducing
        // coherency traffic on the shared global list.
        m = Self->omFreeList ;
        if (m != NULL) {
           Self->omFreeList = m->FreeNext ;
           Self->omFreeCount -- ;
           // CONSIDER: set m->FreeNext = BAD -- diagnostic hygiene
           guarantee (m->object() == NULL, "invariant") ;
           if (MonitorInUseLists) {
             m->FreeNext = Self->omInUseList;
             Self->omInUseList = m;
             Self->omInUseCount ++;
             // verifyInUse(Self);
           } else {
             m->FreeNext = NULL;
           }
           return m ;
        }

        // 2: try to allocate from the global gFreeList
        // CONSIDER: use muxTry() instead of muxAcquire().
        // If the muxTry() fails then drop immediately into case 3.
        // If we're using thread-local free lists then try
        // to reprovision the caller's free list.
        if (gFreeList != NULL) {
            // Reprovision the thread's omFreeList.
            // Use bulk transfers to reduce the allocation rate and heat
            // on various locks.
            Thread::muxAcquire (&ListLock, "omAlloc") ;
            for (int i = Self->omFreeProvision; --i >= 0 && gFreeList != NULL; ) {
                MonitorFreeCount --;
                ObjectMonitor * take = gFreeList ;
                gFreeList = take->FreeNext ;
                guarantee (take->object() == NULL, "invariant") ;
                guarantee (!take->is_busy(), "invariant") ;
                take->Recycle() ;
                omRelease (Self, take, false) ;
            }
            Thread::muxRelease (&ListLock) ;
            Self->omFreeProvision += 1 + (Self->omFreeProvision/2) ;
            if (Self->omFreeProvision > MAXPRIVATE ) Self->omFreeProvision = MAXPRIVATE ;
            TEVENT (omFirst - reprovision) ;

            const int mx = MonitorBound ;
            if (mx > 0 && (MonitorPopulation-MonitorFreeCount) > mx) {
              // We can't safely induce a STW safepoint from omAlloc() as our thread
              // state may not be appropriate for such activities and callers may hold
              // naked oops, so instead we defer the action.
              InduceScavenge (Self, "omAlloc") ;
            }
            continue;
        }

        // 3: allocate a block of new ObjectMonitors
        // Both the local and global free lists are empty -- resort to malloc().
        // In the current implementation objectMonitors are TSM - immortal.
        assert (_BLOCKSIZE > 1, "invariant") ;
        ObjectMonitor * temp = new ObjectMonitor[_BLOCKSIZE];

        // NOTE: (almost) no way to recover if allocation failed.
        // We might be able to induce a STW safepoint and scavenge enough
        // objectMonitors to permit progress.
        if (temp == NULL) {
            vm_exit_out_of_memory (sizeof (ObjectMonitor[_BLOCKSIZE]), OOM_MALLOC_ERROR,
                                   "Allocate ObjectMonitors");
        }

        // Format the block.
        // initialize the linked list, each monitor points to its next
        // forming the single linked free list, the very first monitor
        // will points to next block, which forms the block list.
        // The trick of using the 1st element in the block as gBlockList
        // linkage should be reconsidered.  A better implementation would
        // look like: class Block { Block * next; int N; ObjectMonitor Body [N] ; }

        for (int i = 1; i < _BLOCKSIZE ; i++) {
           temp[i].FreeNext = &temp[i+1];
        }

        // terminate the last monitor as the end of list
        temp[_BLOCKSIZE - 1].FreeNext = NULL ;

        // Element [0] is reserved for global list linkage
        temp[0].set_object(CHAINMARKER);

        // Consider carving out this thread's current request from the
        // block in hand.  This avoids some lock traffic and redundant
        // list activity.

        // Acquire the ListLock to manipulate BlockList and FreeList.
        // An Oyama-Taura-Yonezawa scheme might be more efficient.
        Thread::muxAcquire (&ListLock, "omAlloc [2]") ;
        MonitorPopulation += _BLOCKSIZE-1;
        MonitorFreeCount += _BLOCKSIZE-1;

        // Add the new block to the list of extant blocks (gBlockList).
        // The very first objectMonitor in a block is reserved and dedicated.
        // It serves as blocklist "next" linkage.
        temp[0].FreeNext = gBlockList;
        gBlockList = temp;

        // Add the new string of objectMonitors to the global free list
        temp[_BLOCKSIZE - 1].FreeNext = gFreeList ;
        gFreeList = temp + 1;
        Thread::muxRelease (&ListLock) ;
        TEVENT (Allocate block of monitors) ;
    }
}
```
    1. 从当前线程的omFreeList中，取出一个ObjectMonitor，omFreeList是一个ObjectMonitor组成的链表，链表的next是ObjectMonitor的FreeNext
    2. 如果从ThreadLocal取失败，则从gFreeList（global free list）中取，这时要先上锁，取一个出来，然后把这个放在thread的omFreeList中
    3. new _BLOCKSIZE 个ObjectMonitor，把temp[1] - temp[_BLOCKSIZE]他们当成链表连起来，next节点是ObjectMonitor的FreeNext，锁住ListLock，（temp[0].FreeNext = gBlockList，gBlockList指向temp[0]）， 然后把temp[_BLOCKSIZE - 1].FreeNext = gFreeList,把gFreeList连在后面，然后gFreeList = temp[1]


### ObjectMonitor::wait

```
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
   Thread * const Self = THREAD ;
   assert(Self->is_Java_thread(), "Must be Java thread!");
   JavaThread *jt = (JavaThread *)THREAD;

   DeferredInitialize () ;

   // Throw IMSX or IEX.
   CHECK_OWNER();

   EventJavaMonitorWait event;

   // check for a pending interrupt
   if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
     // post monitor waited event.  Note that this is past-tense, we are done waiting.
     if (JvmtiExport::should_post_monitor_waited()) {
        // Note: 'false' parameter is passed here because the
        // wait was not timed out due to thread interrupt.
        JvmtiExport::post_monitor_waited(jt, this, false);
     }
     if (event.should_commit()) {
       post_monitor_wait_event(&event, 0, millis, false);
     }
     TEVENT (Wait - Throw IEX) ;
     THROW(vmSymbols::java_lang_InterruptedException());
     return ;
   }

   TEVENT (Wait) ;

   assert (Self->_Stalled == 0, "invariant") ;
   Self->_Stalled = intptr_t(this) ;
   jt->set_current_waiting_monitor(this);

   // create a node to be put into the queue
   // Critically, after we reset() the event but prior to park(), we must check
   // for a pending interrupt.
   ObjectWaiter node(Self);
   node.TState = ObjectWaiter::TS_WAIT ;
   Self->_ParkEvent->reset() ;
   OrderAccess::fence();          // ST into Event; membar ; LD interrupted-flag

   // Enter the waiting queue, which is a circular doubly linked list in this case
   // but it could be a priority queue or any data structure.
   // _WaitSetLock protects the wait queue.  Normally the wait queue is accessed only
   // by the the owner of the monitor *except* in the case where park()
   // returns because of a timeout of interrupt.  Contention is exceptionally rare
   // so we use a simple spin-lock instead of a heavier-weight blocking lock.

   Thread::SpinAcquire (&_WaitSetLock, "WaitSet - add") ;
   AddWaiter (&node) ;
   Thread::SpinRelease (&_WaitSetLock) ;

   if ((SyncFlags & 4) == 0) {
      _Responsible = NULL ;
   }
   intptr_t save = _recursions; // record the old recursion count
   _waiters++;                  // increment the number of waiters
   _recursions = 0;             // set the recursion level to be 1
   exit (true, Self) ;                    // exit the monitor
   guarantee (_owner != Self, "invariant") ;

   // As soon as the ObjectMonitor's ownership is dropped in the exit()
   // call above, another thread can enter() the ObjectMonitor, do the
   // notify(), and exit() the ObjectMonitor. If the other thread's
   // exit() call chooses this thread as the successor and the unpark()
   // call happens to occur while this thread is posting a
   // MONITOR_CONTENDED_EXIT event, then we run the risk of the event
   // handler using RawMonitors and consuming the unpark().
   //
   // To avoid the problem, we re-post the event. This does no harm
   // even if the original unpark() was not consumed because we are the
   // chosen successor for this monitor.
   if (node._notified != 0 && _succ == Self) {
      node._event->unpark();
   }

   // The thread is on the WaitSet list - now park() it.
   // On MP systems it's conceivable that a brief spin before we park
   // could be profitable.
   //
   // TODO-FIXME: change the following logic to a loop of the form
   //   while (!timeout && !interrupted && _notified == 0) park()

   int ret = OS_OK ;
   int WasNotified = 0 ;
   { // State transition wrappers
     OSThread* osthread = Self->osthread();
     OSThreadWaitState osts(osthread, true);
     {
       ThreadBlockInVM tbivm(jt);
       // Thread is in thread_blocked state and oop access is unsafe.
       jt->set_suspend_equivalent();

       if (interruptible && (Thread::is_interrupted(THREAD, false) || HAS_PENDING_EXCEPTION)) {
           // Intentionally empty
       } else
       if (node._notified == 0) {
         if (millis <= 0) {
            Self->_ParkEvent->park () ;
         } else {
            ret = Self->_ParkEvent->park (millis) ;
         }
       }

       // were we externally suspended while we were waiting?
       if (ExitSuspendEquivalent (jt)) {
          // TODO-FIXME: add -- if succ == Self then succ = null.
          jt->java_suspend_self();
       }

     } // Exit thread safepoint: transition _thread_blocked -> _thread_in_vm


     // Node may be on the WaitSet, the EntryList (or cxq), or in transition
     // from the WaitSet to the EntryList.
     // See if we need to remove Node from the WaitSet.
     // We use double-checked locking to avoid grabbing _WaitSetLock
     // if the thread is not on the wait queue.
     //
     // Note that we don't need a fence before the fetch of TState.
     // In the worst case we'll fetch a old-stale value of TS_WAIT previously
     // written by the is thread. (perhaps the fetch might even be satisfied
     // by a look-aside into the processor's own store buffer, although given
     // the length of the code path between the prior ST and this load that's
     // highly unlikely).  If the following LD fetches a stale TS_WAIT value
     // then we'll acquire the lock and then re-fetch a fresh TState value.
     // That is, we fail toward safety.

     if (node.TState == ObjectWaiter::TS_WAIT) {
         Thread::SpinAcquire (&_WaitSetLock, "WaitSet - unlink") ;
         if (node.TState == ObjectWaiter::TS_WAIT) {
            DequeueSpecificWaiter (&node) ;       // unlink from WaitSet
            assert(node._notified == 0, "invariant");
            node.TState = ObjectWaiter::TS_RUN ;
         }
         Thread::SpinRelease (&_WaitSetLock) ;
     }

     // The thread is now either on off-list (TS_RUN),
     // on the EntryList (TS_ENTER), or on the cxq (TS_CXQ).
     // The Node's TState variable is stable from the perspective of this thread.
     // No other threads will asynchronously modify TState.
     guarantee (node.TState != ObjectWaiter::TS_WAIT, "invariant") ;
     OrderAccess::loadload() ;
     if (_succ == Self) _succ = NULL ;
     WasNotified = node._notified ;

     // Reentry phase -- reacquire the monitor.
     // re-enter contended monitor after object.wait().
     // retain OBJECT_WAIT state until re-enter successfully completes
     // Thread state is thread_in_vm and oop access is again safe,
     // although the raw address of the object may have changed.
     // (Don't cache naked oops over safepoints, of course).

     // post monitor waited event. Note that this is past-tense, we are done waiting.
     if (JvmtiExport::should_post_monitor_waited()) {
       JvmtiExport::post_monitor_waited(jt, this, ret == OS_TIMEOUT);
     }

     if (event.should_commit()) {
       post_monitor_wait_event(&event, node._notifier_tid, millis, ret == OS_TIMEOUT);
     }

     OrderAccess::fence() ;

     assert (Self->_Stalled != 0, "invariant") ;
     Self->_Stalled = 0 ;

     assert (_owner != Self, "invariant") ;
     ObjectWaiter::TStates v = node.TState ;
     if (v == ObjectWaiter::TS_RUN) {
         enter (Self) ;
     } else {
         guarantee (v == ObjectWaiter::TS_ENTER || v == ObjectWaiter::TS_CXQ, "invariant") ;
         ReenterI (Self, &node) ;
         node.wait_reenter_end(this);
     }

     // Self has reacquired the lock.
     // Lifecycle - the node representing Self must not appear on any queues.
     // Node is about to go out-of-scope, but even if it were immortal we wouldn't
     // want residual elements associated with this thread left on any lists.
     guarantee (node.TState == ObjectWaiter::TS_RUN, "invariant") ;
     assert    (_owner == Self, "invariant") ;
     assert    (_succ != Self , "invariant") ;
   } // OSThreadWaitState()

   jt->set_current_waiting_monitor(NULL);

   guarantee (_recursions == 0, "invariant") ;
   _recursions = save;     // restore the old recursion count
   _waiters--;             // decrement the number of waiters

   // Verify a few postconditions
   assert (_owner == Self       , "invariant") ;
   assert (_succ  != Self       , "invariant") ;
   assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;

   if (SyncFlags & 32) {
      OrderAccess::fence() ;
   }

   // check if the notification happened
   if (!WasNotified) {
     // no, it could be timeout or Thread.interrupt() or both
     // check for interrupt event, otherwise it is timeout
     if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
       TEVENT (Wait - throw IEX from epilog) ;
       THROW(vmSymbols::java_lang_InterruptedException());
     }
   }

   // NOTE: Spurious wake up will be consider as timeout.
   // Monitor notify has precedence over thread interrupt.
}
```
    1. 延迟初始化DeferredInitialize
    2. 如果可以interrupt 并且已经被interrupt了，就直接返回
    3. 设置thread的_Stalled为当前monitor，设置当前javathread的_current_waiting_monitor为当前monitor
    4. 创建一个ObjectWaiter(这是一个线程代理)，设置state为ObjectWaiter::TS_WAIT
    5. reset 当前线程的parkevent
    6. OrderAccess::fence()
    7. 自旋获得锁_WaitSetLock
    8. 把waiter加入_WaitSet
    9. 调用exit，退出了ObjectMonitor！！！
    10. 取到osthread，创建一个OSThreadWaitState，如果_notified为0，park ()
    11. 被唤醒了，状态如果还是TS_WAIT，从_WaitSet取出来，TSstate设为ObjectWaiter::TS_RUN
    12. 重新enter monitor(锁住)
    其中 Self->_ParkEvent->park () 会调用pthread_cond_wait()