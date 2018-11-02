# Synchronized

---

tags:

- hotspot jvm
- lock
- concurrent

categories:

- hotspot jvm

### procedure

![javalock](/Users/liuhongxuan/notes/picture/javalock.png)

### bytecodeInterpreter

> ```c++
> CASE(_monitorenter): {
>         oop lockee = STACK_OBJECT(-1);
>         // derefing's lockee ought to provoke implicit null check
>         CHECK_NULL(lockee);
>         // find a free monitor or one already allocated for this object
>         // if we find a matching object then we need a new monitor
>         // since this is recursive enter
>         BasicObjectLock* limit = istate->monitor_base();
>         BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
>         BasicObjectLock* entry = NULL;
>         while (most_recent != limit ) {
>           if (most_recent->obj() == NULL) entry = most_recent;
>           else if (most_recent->obj() == lockee) break;
>           most_recent++;
>         }
>         if (entry != NULL) {
>           entry->set_obj(lockee);
>           markOop displaced = lockee->mark()->set_unlocked();
>           entry->lock()->set_displaced_header(displaced);
>           if (Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
>             // Is it simple recursive case?
>             if (THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
>               entry->lock()->set_displaced_header(NULL);
>             } else {
>               CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
>             }
>           }
>           UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
>         } else {
>           istate->set_msg(more_monitors);
>           UPDATE_PC_AND_RETURN(0); // Re-execute
>         }
>       }
> ```
>
> > 1. 找到已经分配的或者free的monitor
> > 2. cas 给object的markword赋值
> > 3. cas失败，如果是线程重入，算成功
> > 4. call InterpreterRuntime::monitorenter

### InterpreterRuntime::monitorenter

> ```c++
> IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
> #ifdef ASSERT
>   thread->last_frame().interpreter_frame_verify_monitor(elem);
> #endif
>   if (PrintBiasedLockingStatistics) {
>     Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
>   }
>   Handle h_obj(thread, elem->obj());
>   assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
>          "must be NULL or an object");
>   if (UseBiasedLocking) {
>     // Retry fast entry if bias is revoked to avoid unnecessary inflation
>     ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
>   } else {
>     ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
>   }
>   assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
>          "must be NULL or an object");
> #ifdef ASSERT
>   thread->last_frame().interpreter_frame_verify_monitor(elem);
> #endif
> IRT_END
> ```
>
> #### summarize
>
> 1. 如果用偏向锁，进如fast_enter
> 2. 否则进如slow_enter
> 3. **偏向锁的逻辑懒得看了**，直接看slow_enter

### slow_enter

> ```c++
> // -----------------------------------------------------------------------------
> // Interpreter/Compiler Slow Case
> // This routine is used to handle interpreter/compiler slow case
> // We don't need to use fast path here, because it must have been
> // failed in the interpreter/compiler code.
> void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
>   markOop mark = obj->mark();
>   assert(!mark->has_bias_pattern(), "should not see bias pattern here");
> 
>   if (mark->is_neutral()) {
>     // Anticipate successful CAS -- the ST of the displaced mark must
>     // be visible <= the ST performed by the CAS.
>     lock->set_displaced_header(mark);
>     if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
>       TEVENT (slow_enter: release stacklock) ;
>       return ;
>     }
>     // Fall through to inflate() ...
>   } else
>   if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
>     assert(lock != mark->locker(), "must not re-lock the same lock");
>     assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
>     lock->set_displaced_header(NULL);
>     return;
>   }
> 
> #if 0
>   // The following optimization isn't particularly useful.
>   if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
>     lock->set_displaced_header (NULL) ;
>     return ;
>   }
> #endif
> 
>   // The object header will never be displaced to this lock,
>   // so it does not matter what the value is, except that it
>   // must be non-zero to avoid looking like a re-entrant lock,
>   // and must not look locked either.
>   lock->set_displaced_header(markOopDesc::unused_mark());
>   ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
> }
> ```
>
> 1. 如果当前没有上锁，即取markword后三位为001，并且cas成功，就返回(目测是轻量级锁的逻辑)
> 2. mark->has_locker()判断后两位是不是轻量级锁，即后两位是不是00，如果是，并且是当前线程，返回
> 3. inflate and enter

### enter

> ```c++
> void ATTR ObjectMonitor::enter(TRAPS) {
>   // The following code is ordered to check the most common cases first
>   // and to reduce RTS->RTO cache line upgrades on SPARC and IA32 processors.
>   Thread * const Self = THREAD ;
>   void * cur ;
> 
>   cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
>   if (cur == NULL) {
>      // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
>      assert (_recursions == 0   , "invariant") ;
>      assert (_owner      == Self, "invariant") ;
>      // CONSIDER: set or assert OwnerIsThread == 1
>      return ;
>   }
> 
>   if (cur == Self) {
>      // TODO-FIXME: check for integer overflow!  BUGID 6557169.
>      _recursions ++ ;
>      return ;
>   }
> 
>   if (Self->is_lock_owned ((address)cur)) {
>     assert (_recursions == 0, "internal state error");
>     _recursions = 1 ;
>     // Commute owner from a thread-specific on-stack BasicLockObject address to
>     // a full-fledged "Thread *".
>     _owner = Self ;
>     OwnerIsThread = 1 ;
>     return ;
>   }
> 
>   // We've encountered genuine contention.
>   assert (Self->_Stalled == 0, "invariant") ;
>   Self->_Stalled = intptr_t(this) ;
> 
>   // Try one round of spinning *before* enqueueing Self
>   // and before going through the awkward and expensive state
>   // transitions.  The following spin is strictly optional ...
>   // Note that if we acquire the monitor from an initial spin
>   // we forgo posting JVMTI events and firing DTRACE probes.
>   if (Knob_SpinEarly && TrySpin (Self) > 0) {
>      assert (_owner == Self      , "invariant") ;
>      assert (_recursions == 0    , "invariant") ;
>      assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
>      Self->_Stalled = 0 ;
>      return ;
>   }
> 
>   assert (_owner != Self          , "invariant") ;
>   assert (_succ  != Self          , "invariant") ;
>   assert (Self->is_Java_thread()  , "invariant") ;
>   JavaThread * jt = (JavaThread *) Self ;
>   assert (!SafepointSynchronize::is_at_safepoint(), "invariant") ;
>   assert (jt->thread_state() != _thread_blocked   , "invariant") ;
>   assert (this->object() != NULL  , "invariant") ;
>   assert (_count >= 0, "invariant") ;
> 
>   // Prevent deflation at STW-time.  See deflate_idle_monitors() and is_busy().
>   // Ensure the object-monitor relationship remains stable while there's contention.
>   Atomic::inc_ptr(&_count);
> 
>   EventJavaMonitorEnter event;
> 
>   { // Change java thread status to indicate blocked on monitor enter.
>     JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);
> 
>     DTRACE_MONITOR_PROBE(contended__enter, this, object(), jt);
>     if (JvmtiExport::should_post_monitor_contended_enter()) {
>       JvmtiExport::post_monitor_contended_enter(jt, this);
>     }
> 
>     OSThreadContendState osts(Self->osthread());
>     ThreadBlockInVM tbivm(jt);
> 
>     Self->set_current_pending_monitor(this);
> 
>     // TODO-FIXME: change the following for(;;) loop to straight-line code.
>     for (;;) {
>       jt->set_suspend_equivalent();
>       // cleared by handle_special_suspend_equivalent_condition()
>       // or java_suspend_self()
> 
>       EnterI (THREAD) ;
> 
>       if (!ExitSuspendEquivalent(jt)) break ;
> 
>       //
>       // We have acquired the contended monitor, but while we were
>       // waiting another thread suspended us. We don't want to enter
>       // the monitor while suspended because that would surprise the
>       // thread that suspended us.
>       //
>           _recursions = 0 ;
>       _succ = NULL ;
>       exit (false, Self) ;
> 
>       jt->java_suspend_self();
>     }
>     Self->set_current_pending_monitor(NULL);
>   }
> 
>   Atomic::dec_ptr(&_count);
>   assert (_count >= 0, "invariant") ;
>   Self->_Stalled = 0 ;
> 
>   // Must either set _recursions = 0 or ASSERT _recursions == 0.
>   assert (_recursions == 0     , "invariant") ;
>   assert (_owner == Self       , "invariant") ;
>   assert (_succ  != Self       , "invariant") ;
>   assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
> 
>   // The thread -- now the owner -- is back in vm mode.
>   // Report the glorious news via TI,DTrace and jvmstat.
>   // The probe effect is non-trivial.  All the reportage occurs
>   // while we hold the monitor, increasing the length of the critical
>   // section.  Amdahl's parallel speedup law comes vividly into play.
>   //
>   // Another option might be to aggregate the events (thread local or
>   // per-monitor aggregation) and defer reporting until a more opportune
>   // time -- such as next time some thread encounters contention but has
>   // yet to acquire the lock.  While spinning that thread could
>   // spinning we could increment JVMStat counters, etc.
> 
>   DTRACE_MONITOR_PROBE(contended__entered, this, object(), jt);
>   if (JvmtiExport::should_post_monitor_contended_entered()) {
>     JvmtiExport::post_monitor_contended_entered(jt, this);
>   }
> 
>   if (event.should_commit()) {
>     event.set_klass(((oop)this->object())->klass());
>     event.set_previousOwner((TYPE_JAVALANGTHREAD)_previous_owner_tid);
>     event.set_address((TYPE_ADDRESS)(uintptr_t)(this->object_addr()));
>     event.commit();
>   }
> 
>   if (ObjectMonitor::_sync_ContendedLockAttempts != NULL) {
>      ObjectMonitor::_sync_ContendedLockAttempts->inc() ;
>   }
> }
> ```
>
> 1. cas设置ObjectMonitor的owner为当前线程，如果失败了则返回，说明objectmonitor已经有owner了
> 2. 如果owner本来就是自己，`_recursions ++` ，返回
> 3. 如果之前的BasicLockObject是一个在当前线程栈上的锁？？？，
> 4. 设置当前线程的_Stalled为这个objectMonitor
> 5. 先自旋一段时间，看能不能获取锁，能获取则返回
> 6. 不能的话，设置当前线程的_current_pending_monitor
> 7. EnterI

### EnterI

> ```c++
> void ATTR ObjectMonitor::EnterI (TRAPS) {
>     Thread * Self = THREAD ;
>     assert (Self->is_Java_thread(), "invariant") ;
>     assert (((JavaThread *) Self)->thread_state() == _thread_blocked   , "invariant") ;
> 
>     // Try the lock - TATAS
>     if (TryLock (Self) > 0) {
>         assert (_succ != Self              , "invariant") ;
>         assert (_owner == Self             , "invariant") ;
>         assert (_Responsible != Self       , "invariant") ;
>         return ;
>     }
> 
>     DeferredInitialize () ;
> 
>     // We try one round of spinning *before* enqueueing Self.
>     //
>     // If the _owner is ready but OFFPROC we could use a YieldTo()
>     // operation to donate the remainder of this thread's quantum
>     // to the owner.  This has subtle but beneficial affinity
>     // effects.
> 
>     if (TrySpin (Self) > 0) {
>         assert (_owner == Self        , "invariant") ;
>         assert (_succ != Self         , "invariant") ;
>         assert (_Responsible != Self  , "invariant") ;
>         return ;
>     }
> 
>     // The Spin failed -- Enqueue and park the thread ...
>     assert (_succ  != Self            , "invariant") ;
>     assert (_owner != Self            , "invariant") ;
>     assert (_Responsible != Self      , "invariant") ;
> 
>     // Enqueue "Self" on ObjectMonitor's _cxq.
>     //
>     // Node acts as a proxy for Self.
>     // As an aside, if were to ever rewrite the synchronization code mostly
>     // in Java, WaitNodes, ObjectMonitors, and Events would become 1st-class
>     // Java objects.  This would avoid awkward lifecycle and liveness issues,
>     // as well as eliminate a subset of ABA issues.
>     // TODO: eliminate ObjectWaiter and enqueue either Threads or Events.
>     //
> 
>     ObjectWaiter node(Self) ;
>     Self->_ParkEvent->reset() ;
>     node._prev   = (ObjectWaiter *) 0xBAD ;
>     node.TState  = ObjectWaiter::TS_CXQ ;
> 
>     // Push "Self" onto the front of the _cxq.
>     // Once on cxq/EntryList, Self stays on-queue until it acquires the lock.
>     // Note that spinning tends to reduce the rate at which threads
>     // enqueue and dequeue on EntryList|cxq.
>     ObjectWaiter * nxt ;
>     for (;;) {
>         node._next = nxt = _cxq ;
>         if (Atomic::cmpxchg_ptr (&node, &_cxq, nxt) == nxt) break ;
> 
>         // Interference - the CAS failed because _cxq changed.  Just retry.
>         // As an optional optimization we retry the lock.
>         if (TryLock (Self) > 0) {
>             assert (_succ != Self         , "invariant") ;
>             assert (_owner == Self        , "invariant") ;
>             assert (_Responsible != Self  , "invariant") ;
>             return ;
>         }
>     }
> 
>     // Check for cxq|EntryList edge transition to non-null.  This indicates
>     // the onset of contention.  While contention persists exiting threads
>     // will use a ST:MEMBAR:LD 1-1 exit protocol.  When contention abates exit
>     // operations revert to the faster 1-0 mode.  This enter operation may interleave
>     // (race) a concurrent 1-0 exit operation, resulting in stranding, so we
>     // arrange for one of the contending thread to use a timed park() operations
>     // to detect and recover from the race.  (Stranding is form of progress failure
>     // where the monitor is unlocked but all the contending threads remain parked).
>     // That is, at least one of the contended threads will periodically poll _owner.
>     // One of the contending threads will become the designated "Responsible" thread.
>     // The Responsible thread uses a timed park instead of a normal indefinite park
>     // operation -- it periodically wakes and checks for and recovers from potential
>     // strandings admitted by 1-0 exit operations.   We need at most one Responsible
>     // thread per-monitor at any given moment.  Only threads on cxq|EntryList may
>     // be responsible for a monitor.
>     //
>     // Currently, one of the contended threads takes on the added role of "Responsible".
>     // A viable alternative would be to use a dedicated "stranding checker" thread
>     // that periodically iterated over all the threads (or active monitors) and unparked
>     // successors where there was risk of stranding.  This would help eliminate the
>     // timer scalability issues we see on some platforms as we'd only have one thread
>     // -- the checker -- parked on a timer.
> 
>     if ((SyncFlags & 16) == 0 && nxt == NULL && _EntryList == NULL) {
>         // Try to assume the role of responsible thread for the monitor.
>         // CONSIDER:  ST vs CAS vs { if (Responsible==null) Responsible=Self }
>         Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
>     }
> 
>     // The lock have been released while this thread was occupied queueing
>     // itself onto _cxq.  To close the race and avoid "stranding" and
>     // progress-liveness failure we must resample-retry _owner before parking.
>     // Note the Dekker/Lamport duality: ST cxq; MEMBAR; LD Owner.
>     // In this case the ST-MEMBAR is accomplished with CAS().
>     //
>     // TODO: Defer all thread state transitions until park-time.
>     // Since state transitions are heavy and inefficient we'd like
>     // to defer the state transitions until absolutely necessary,
>     // and in doing so avoid some transitions ...
> 
>     TEVENT (Inflated enter - Contention) ;
>     int nWakeups = 0 ;
>     int RecheckInterval = 1 ;
> 
>     for (;;) {
> 
>         if (TryLock (Self) > 0) break ;
>         assert (_owner != Self, "invariant") ;
> 
>         if ((SyncFlags & 2) && _Responsible == NULL) {
>            Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
>         }
> 
>         // park self
>         if (_Responsible == Self || (SyncFlags & 1)) {
>             TEVENT (Inflated enter - park TIMED) ;
>             Self->_ParkEvent->park ((jlong) RecheckInterval) ;
>             // Increase the RecheckInterval, but clamp the value.
>             RecheckInterval *= 8 ;
>             if (RecheckInterval > 1000) RecheckInterval = 1000 ;
>         } else {
>             TEVENT (Inflated enter - park UNTIMED) ;
>             Self->_ParkEvent->park() ;
>         }
> 
>         if (TryLock(Self) > 0) break ;
> 
>         // The lock is still contested.
>         // Keep a tally of the # of futile wakeups.
>         // Note that the counter is not protected by a lock or updated by atomics.
>         // That is by design - we trade "lossy" counters which are exposed to
>         // races during updates for a lower probe effect.
>         TEVENT (Inflated enter - Futile wakeup) ;
>         if (ObjectMonitor::_sync_FutileWakeups != NULL) {
>            ObjectMonitor::_sync_FutileWakeups->inc() ;
>         }
>         ++ nWakeups ;
> 
>         // Assuming this is not a spurious wakeup we'll normally find _succ == Self.
>         // We can defer clearing _succ until after the spin completes
>         // TrySpin() must tolerate being called with _succ == Self.
>         // Try yet another round of adaptive spinning.
>         if ((Knob_SpinAfterFutile & 1) && TrySpin (Self) > 0) break ;
> 
>         // We can find that we were unpark()ed and redesignated _succ while
>         // we were spinning.  That's harmless.  If we iterate and call park(),
>         // park() will consume the event and return immediately and we'll
>         // just spin again.  This pattern can repeat, leaving _succ to simply
>         // spin on a CPU.  Enable Knob_ResetEvent to clear pending unparks().
>         // Alternately, we can sample fired() here, and if set, forgo spinning
>         // in the next iteration.
> 
>         if ((Knob_ResetEvent & 1) && Self->_ParkEvent->fired()) {
>            Self->_ParkEvent->reset() ;
>            OrderAccess::fence() ;
>         }
>         if (_succ == Self) _succ = NULL ;
> 
>         // Invariant: after clearing _succ a thread *must* retry _owner before parking.
>         OrderAccess::fence() ;
>     }
> 
>     // Egress :
>     // Self has acquired the lock -- Unlink Self from the cxq or EntryList.
>     // Normally we'll find Self on the EntryList .
>     // From the perspective of the lock owner (this thread), the
>     // EntryList is stable and cxq is prepend-only.
>     // The head of cxq is volatile but the interior is stable.
>     // In addition, Self.TState is stable.
> 
>     assert (_owner == Self      , "invariant") ;
>     assert (object() != NULL    , "invariant") ;
>     // I'd like to write:
>     //   guarantee (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
>     // but as we're at a safepoint that's not safe.
> 
>     UnlinkAfterAcquire (Self, &node) ;
>     if (_succ == Self) _succ = NULL ;
> 
>     assert (_succ != Self, "invariant") ;
>     if (_Responsible == Self) {
>         _Responsible = NULL ;
>         OrderAccess::fence(); // Dekker pivot-point
> 
>         // We may leave threads on cxq|EntryList without a designated
>         // "Responsible" thread.  This is benign.  When this thread subsequently
>         // exits the monitor it can "see" such preexisting "old" threads --
>         // threads that arrived on the cxq|EntryList before the fence, above --
>         // by LDing cxq|EntryList.  Newly arrived threads -- that is, threads
>         // that arrive on cxq after the ST:MEMBAR, above -- will set Responsible
>         // non-null and elect a new "Responsible" timer thread.
>         //
>         // This thread executes:
>         //    ST Responsible=null; MEMBAR    (in enter epilog - here)
>         //    LD cxq|EntryList               (in subsequent exit)
>         //
>         // Entering threads in the slow/contended path execute:
>         //    ST cxq=nonnull; MEMBAR; LD Responsible (in enter prolog)
>         //    The (ST cxq; MEMBAR) is accomplished with CAS().
>         //
>         // The MEMBAR, above, prevents the LD of cxq|EntryList in the subsequent
>         // exit operation from floating above the ST Responsible=null.
>     }
> 
>     // We've acquired ownership with CAS().
>     // CAS is serializing -- it has MEMBAR/FENCE-equivalent semantics.
>     // But since the CAS() this thread may have also stored into _succ,
>     // EntryList, cxq or Responsible.  These meta-data updates must be
>     // visible __before this thread subsequently drops the lock.
>     // Consider what could occur if we didn't enforce this constraint --
>     // STs to monitor meta-data and user-data could reorder with (become
>     // visible after) the ST in exit that drops ownership of the lock.
>     // Some other thread could then acquire the lock, but observe inconsistent
>     // or old monitor meta-data and heap data.  That violates the JMM.
>     // To that end, the 1-0 exit() operation must have at least STST|LDST
>     // "release" barrier semantics.  Specifically, there must be at least a
>     // STST|LDST barrier in exit() before the ST of null into _owner that drops
>     // the lock.   The barrier ensures that changes to monitor meta-data and data
>     // protected by the lock will be visible before we release the lock, and
>     // therefore before some other thread (CPU) has a chance to acquire the lock.
>     // See also: http://gee.cs.oswego.edu/dl/jmm/cookbook.html.
>     //
>     // Critically, any prior STs to _succ or EntryList must be visible before
>     // the ST of null into _owner in the *subsequent* (following) corresponding
>     // monitorexit.  Recall too, that in 1-0 mode monitorexit does not necessarily
>     // execute a serializing instruction.
> 
>     if (SyncFlags & 8) {
>        OrderAccess::fence() ;
>     }
>     return ;
> }
> ```
>
> 1. 先尝试用cas锁
>
> 2. 在spin一轮，看能不能获得锁
>
> 3. 没办法了，只能创建一个ObjectWaiter node(Self)
>
> 4. 把这个waiter node push到cxq的头部，cxq是LL of recently-arrived threads blocked on entry
>
> 5. cas 失败，说明头变了，立马cas try lock一次
>
> 6. 如果nxt，_EntryList都是空的，把当前thread设置为这个monitor的_Responsible thread
>
> 7. 死循环
>
>    >1. 如果cas获得锁成功，退出
>    >2. 如果这个线程是当前的Responsible thread，park 1 ms，否则 一直park，等别人来唤醒
>    >3. 唤醒后，检查cas能否成功
>
> 8. 清理ObjectWaiter

### MonitorExit

> ```c++
> IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))
> #ifdef ASSERT
>   thread->last_frame().interpreter_frame_verify_monitor(elem);
> #endif
>   Handle h_obj(thread, elem->obj());
>   assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
>          "must be NULL or an object");
>   if (elem == NULL || h_obj()->is_unlocked()) {
>     THROW(vmSymbols::java_lang_IllegalMonitorStateException());
>   }
>   ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
>   // Free entry. This must be done here, since a pending exception might be installed on
>   // exit. If it is not cleared, the exception handling code will try to unlock the monitor again.
>   elem->set_obj(NULL);
> #ifdef ASSERT
>   thread->last_frame().interpreter_frame_verify_monitor(elem);
> #endif
> IRT_END
> ```
>
>

### slow_exit

> ```c++
> void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
>   assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");
>   // if displaced header is null, the previous enter is recursive enter, no-op
>   markOop dhw = lock->displaced_header();
>   markOop mark ;
>   if (dhw == NULL) {
>      // Recursive stack-lock.
>      // Diagnostics -- Could be: stack-locked, inflating, inflated.
>      mark = object->mark() ;
>      assert (!mark->is_neutral(), "invariant") ;
>      if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
>         assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
>      }
>      if (mark->has_monitor()) {
>         ObjectMonitor * m = mark->monitor() ;
>         assert(((oop)(m->object()))->mark() == mark, "invariant") ;
>         assert(m->is_entered(THREAD), "invariant") ;
>      }
>      return ;
>   }
> 
>   mark = object->mark() ;
> 
>   // If the object is stack-locked by the current thread, try to
>   // swing the displaced header from the box back to the mark.
>   if (mark == (markOop) lock) {
>      assert (dhw->is_neutral(), "invariant") ;
>      if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
>         TEVENT (fast_exit: release stacklock) ;
>         return;
>      }
>   }
> 
>   ObjectSynchronizer::inflate(THREAD, object)->exit (true, THREAD) ;
> }
> ```
>
> 1. 如果displace_header为null，说明是重入锁
> 2. 如果是stack-locked，cas 解锁
> 3. exit

### ObjectMonitor::exit

> ```c++
> void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) {
>    Thread * Self = THREAD ;
>    if (THREAD != _owner) {
>      if (THREAD->is_lock_owned((address) _owner)) {
>        // Transmute _owner from a BasicLock pointer to a Thread address.
>        // We don't need to hold _mutex for this transition.
>        // Non-null to Non-null is safe as long as all readers can
>        // tolerate either flavor.
>        assert (_recursions == 0, "invariant") ;
>        _owner = THREAD ;
>        _recursions = 0 ;
>        OwnerIsThread = 1 ;
>      } else {
>        // NOTE: we need to handle unbalanced monitor enter/exit
>        // in native code by throwing an exception.
>        // TODO: Throw an IllegalMonitorStateException ?
>        TEVENT (Exit - Throw IMSX) ;
>        assert(false, "Non-balanced monitor enter/exit!");
>        if (false) {
>           THROW(vmSymbols::java_lang_IllegalMonitorStateException());
>        }
>        return;
>      }
>    }
> 
>    if (_recursions != 0) {
>      _recursions--;        // this is simple recursive enter
>      TEVENT (Inflated exit - recursive) ;
>      return ;
>    }
> 
>    // Invariant: after setting Responsible=null an thread must execute
>    // a MEMBAR or other serializing instruction before fetching EntryList|cxq.
>    if ((SyncFlags & 4) == 0) {
>       _Responsible = NULL ;
>    }
> 
> #if INCLUDE_TRACE
>    // get the owner's thread id for the MonitorEnter event
>    // if it is enabled and the thread isn't suspended
>    if (not_suspended && Tracing::is_event_enabled(TraceJavaMonitorEnterEvent)) {
>      _previous_owner_tid = SharedRuntime::get_java_tid(Self);
>    }
> #endif
> 
>    for (;;) {
>       assert (THREAD == _owner, "invariant") ;
> 
> 
>       if (Knob_ExitPolicy == 0) {
>          // release semantics: prior loads and stores from within the critical section
>          // must not float (reorder) past the following store that drops the lock.
>          // On SPARC that requires MEMBAR #loadstore|#storestore.
>          // But of course in TSO #loadstore|#storestore is not required.
>          // I'd like to write one of the following:
>          // A.  OrderAccess::release() ; _owner = NULL
>          // B.  OrderAccess::loadstore(); OrderAccess::storestore(); _owner = NULL;
>          // Unfortunately OrderAccess::release() and OrderAccess::loadstore() both
>          // store into a _dummy variable.  That store is not needed, but can result
>          // in massive wasteful coherency traffic on classic SMP systems.
>          // Instead, I use release_store(), which is implemented as just a simple
>          // ST on x64, x86 and SPARC.
>          OrderAccess::release_store_ptr (&_owner, NULL) ;   // drop the lock
>          OrderAccess::storeload() ;                         // See if we need to wake a successor
>          if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
>             TEVENT (Inflated exit - simple egress) ;
>             return ;
>          }
>          TEVENT (Inflated exit - complex egress) ;
> 
>          // Normally the exiting thread is responsible for ensuring succession,
>          // but if other successors are ready or other entering threads are spinning
>          // then this thread can simply store NULL into _owner and exit without
>          // waking a successor.  The existence of spinners or ready successors
>          // guarantees proper succession (liveness).  Responsibility passes to the
>          // ready or running successors.  The exiting thread delegates the duty.
>          // More precisely, if a successor already exists this thread is absolved
>          // of the responsibility of waking (unparking) one.
>          //
>          // The _succ variable is critical to reducing futile wakeup frequency.
>          // _succ identifies the "heir presumptive" thread that has been made
>          // ready (unparked) but that has not yet run.  We need only one such
>          // successor thread to guarantee progress.
>          // See http://www.usenix.org/events/jvm01/full_papers/dice/dice.pdf
>          // section 3.3 "Futile Wakeup Throttling" for details.
>          //
>          // Note that spinners in Enter() also set _succ non-null.
>          // In the current implementation spinners opportunistically set
>          // _succ so that exiting threads might avoid waking a successor.
>          // Another less appealing alternative would be for the exiting thread
>          // to drop the lock and then spin briefly to see if a spinner managed
>          // to acquire the lock.  If so, the exiting thread could exit
>          // immediately without waking a successor, otherwise the exiting
>          // thread would need to dequeue and wake a successor.
>          // (Note that we'd need to make the post-drop spin short, but no
>          // shorter than the worst-case round-trip cache-line migration time.
>          // The dropped lock needs to become visible to the spinner, and then
>          // the acquisition of the lock by the spinner must become visible to
>          // the exiting thread).
>          //
> 
>          // It appears that an heir-presumptive (successor) must be made ready.
>          // Only the current lock owner can manipulate the EntryList or
>          // drain _cxq, so we need to reacquire the lock.  If we fail
>          // to reacquire the lock the responsibility for ensuring succession
>          // falls to the new owner.
>          //
>          if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
>             return ;
>          }
>          TEVENT (Exit - Reacquired) ;
>       } else {
>          if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
>             OrderAccess::release_store_ptr (&_owner, NULL) ;   // drop the lock
>             OrderAccess::storeload() ;
>             // Ratify the previously observed values.
>             if (_cxq == NULL || _succ != NULL) {
>                 TEVENT (Inflated exit - simple egress) ;
>                 return ;
>             }
> 
>             // inopportune interleaving -- the exiting thread (this thread)
>             // in the fast-exit path raced an entering thread in the slow-enter
>             // path.
>             // We have two choices:
>             // A.  Try to reacquire the lock.
>             //     If the CAS() fails return immediately, otherwise
>             //     we either restart/rerun the exit operation, or simply
>             //     fall-through into the code below which wakes a successor.
>             // B.  If the elements forming the EntryList|cxq are TSM
>             //     we could simply unpark() the lead thread and return
>             //     without having set _succ.
>             if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
>                TEVENT (Inflated exit - reacquired succeeded) ;
>                return ;
>             }
>             TEVENT (Inflated exit - reacquired failed) ;
>          } else {
>             TEVENT (Inflated exit - complex egress) ;
>          }
>       }
> 
>       guarantee (_owner == THREAD, "invariant") ;
> 
>       ObjectWaiter * w = NULL ;
>       int QMode = Knob_QMode ;
> 
>       if (QMode == 2 && _cxq != NULL) {
>           // QMode == 2 : cxq has precedence over EntryList.
>           // Try to directly wake a successor from the cxq.
>           // If successful, the successor will need to unlink itself from cxq.
>           w = _cxq ;
>           assert (w != NULL, "invariant") ;
>           assert (w->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
>           ExitEpilog (Self, w) ;
>           return ;
>       }
> 
>       if (QMode == 3 && _cxq != NULL) {
>           // Aggressively drain cxq into EntryList at the first opportunity.
>           // This policy ensure that recently-run threads live at the head of EntryList.
>           // Drain _cxq into EntryList - bulk transfer.
>           // First, detach _cxq.
>           // The following loop is tantamount to: w = swap (&cxq, NULL)
>           w = _cxq ;
>           for (;;) {
>              assert (w != NULL, "Invariant") ;
>              ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
>              if (u == w) break ;
>              w = u ;
>           }
>           assert (w != NULL              , "invariant") ;
> 
>           ObjectWaiter * q = NULL ;
>           ObjectWaiter * p ;
>           for (p = w ; p != NULL ; p = p->_next) {
>               guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
>               p->TState = ObjectWaiter::TS_ENTER ;
>               p->_prev = q ;
>               q = p ;
>           }
> 
>           // Append the RATs to the EntryList
>           // TODO: organize EntryList as a CDLL so we can locate the tail in constant-time.
>           ObjectWaiter * Tail ;
>           for (Tail = _EntryList ; Tail != NULL && Tail->_next != NULL ; Tail = Tail->_next) ;
>           if (Tail == NULL) {
>               _EntryList = w ;
>           } else {
>               Tail->_next = w ;
>               w->_prev = Tail ;
>           }
> 
>           // Fall thru into code that tries to wake a successor from EntryList
>       }
> 
>       if (QMode == 4 && _cxq != NULL) {
>           // Aggressively drain cxq into EntryList at the first opportunity.
>           // This policy ensure that recently-run threads live at the head of EntryList.
> 
>           // Drain _cxq into EntryList - bulk transfer.
>           // First, detach _cxq.
>           // The following loop is tantamount to: w = swap (&cxq, NULL)
>           w = _cxq ;
>           for (;;) {
>              assert (w != NULL, "Invariant") ;
>              ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
>              if (u == w) break ;
>              w = u ;
>           }
>           assert (w != NULL              , "invariant") ;
> 
>           ObjectWaiter * q = NULL ;
>           ObjectWaiter * p ;
>           for (p = w ; p != NULL ; p = p->_next) {
>               guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
>               p->TState = ObjectWaiter::TS_ENTER ;
>               p->_prev = q ;
>               q = p ;
>           }
> 
>           // Prepend the RATs to the EntryList
>           if (_EntryList != NULL) {
>               q->_next = _EntryList ;
>               _EntryList->_prev = q ;
>           }
>           _EntryList = w ;
> 
>           // Fall thru into code that tries to wake a successor from EntryList
>       }
> 
>       w = _EntryList  ;
>       if (w != NULL) {
>           // I'd like to write: guarantee (w->_thread != Self).
>           // But in practice an exiting thread may find itself on the EntryList.
>           // Lets say thread T1 calls O.wait().  Wait() enqueues T1 on O's waitset and
>           // then calls exit().  Exit release the lock by setting O._owner to NULL.
>           // Lets say T1 then stalls.  T2 acquires O and calls O.notify().  The
>           // notify() operation moves T1 from O's waitset to O's EntryList. T2 then
>           // release the lock "O".  T2 resumes immediately after the ST of null into
>           // _owner, above.  T2 notices that the EntryList is populated, so it
>           // reacquires the lock and then finds itself on the EntryList.
>           // Given all that, we have to tolerate the circumstance where "w" is
>           // associated with Self.
>           assert (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
>           ExitEpilog (Self, w) ;
>           return ;
>       }
> 
>       // If we find that both _cxq and EntryList are null then just
>       // re-run the exit protocol from the top.
>       w = _cxq ;
>       if (w == NULL) continue ;
> 
>       // Drain _cxq into EntryList - bulk transfer.
>       // First, detach _cxq.
>       // The following loop is tantamount to: w = swap (&cxq, NULL)
>       for (;;) {
>           assert (w != NULL, "Invariant") ;
>           ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
>           if (u == w) break ;
>           w = u ;
>       }
>       TEVENT (Inflated exit - drain cxq into EntryList) ;
> 
>       assert (w != NULL              , "invariant") ;
>       assert (_EntryList  == NULL    , "invariant") ;
> 
>       // Convert the LIFO SLL anchored by _cxq into a DLL.
>       // The list reorganization step operates in O(LENGTH(w)) time.
>       // It's critical that this step operate quickly as
>       // "Self" still holds the outer-lock, restricting parallelism
>       // and effectively lengthening the critical section.
>       // Invariant: s chases t chases u.
>       // TODO-FIXME: consider changing EntryList from a DLL to a CDLL so
>       // we have faster access to the tail.
> 
>       if (QMode == 1) {
>          // QMode == 1 : drain cxq to EntryList, reversing order
>          // We also reverse the order of the list.
>          ObjectWaiter * s = NULL ;
>          ObjectWaiter * t = w ;
>          ObjectWaiter * u = NULL ;
>          while (t != NULL) {
>              guarantee (t->TState == ObjectWaiter::TS_CXQ, "invariant") ;
>              t->TState = ObjectWaiter::TS_ENTER ;
>              u = t->_next ;
>              t->_prev = u ;
>              t->_next = s ;
>              s = t;
>              t = u ;
>          }
>          _EntryList  = s ;
>          assert (s != NULL, "invariant") ;
>       } else {
>          // QMode == 0 or QMode == 2
>          _EntryList = w ;
>          ObjectWaiter * q = NULL ;
>          ObjectWaiter * p ;
>          for (p = w ; p != NULL ; p = p->_next) {
>              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
>              p->TState = ObjectWaiter::TS_ENTER ;
>              p->_prev = q ;
>              q = p ;
>          }
>       }
> 
>       // In 1-0 mode we need: ST EntryList; MEMBAR #storestore; ST _owner = NULL
>       // The MEMBAR is satisfied by the release_store() operation in ExitEpilog().
> 
>       // See if we can abdicate to a spinner instead of waking a thread.
>       // A primary goal of the implementation is to reduce the
>       // context-switch rate.
>       if (_succ != NULL) continue;
> 
>       w = _EntryList  ;
>       if (w != NULL) {
>           guarantee (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
>           ExitEpilog (Self, w) ;
>           return ;
>       }
>    }
> }
> ```
>
> 1. 如果是重入锁，_recursions--
> 2. QMode == 2，目测是公平锁，取最近来的cxq，unpark
> 3. QMode == 3 ，先把cxq置为null，把从cxq取出来的waiter，连接在_EntryList后面
> 4. QMode == 4 ，先把cxq置为null，把从cxq取出来的waiter，连接在_EntryList前面
> 5. 然后取出来_EntryList第一个元素，unpark
> 6. 如果_EntryList为null，QMode == 1，reverse _EntryList 然后同理
> 7. 总的来说，找一个waiter，把它unpark

