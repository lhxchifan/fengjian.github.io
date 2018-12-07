# Object.notify

## ObjectSynchronizer::notify

```c++
void ObjectSynchronizer::notify(Handle obj, TRAPS) {
 if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  markOop mark = obj->mark();
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    return;
  }
  ObjectSynchronizer::inflate(THREAD, obj())->notify(THREAD);
}
```

> 1. 如果启用的偏向锁，撤销偏向
> 2. 如果当前有轻量级锁，并且属于这个线程，就直接返回了，说明没有其他人在wait
> 3. notify

## ObjectMonitor::notify

```c++
void ObjectMonitor::notify(TRAPS) {
  CHECK_OWNER();
  if (_WaitSet == NULL) {
     TEVENT (Empty-Notify) ;
     return ;
  }
  DTRACE_MONITOR_PROBE(notify, this, object(), THREAD);

  int Policy = Knob_MoveNotifyee ;

  Thread::SpinAcquire (&_WaitSetLock, "WaitSet - notify") ;
  ObjectWaiter * iterator = DequeueWaiter() ;
  if (iterator != NULL) {
     TEVENT (Notify1 - Transfer) ;
     guarantee (iterator->TState == ObjectWaiter::TS_WAIT, "invariant") ;
     guarantee (iterator->_notified == 0, "invariant") ;
     if (Policy != 4) {
        iterator->TState = ObjectWaiter::TS_ENTER ;
     }
     iterator->_notified = 1 ;
     Thread * Self = THREAD;
     iterator->_notifier_tid = Self->osthread()->thread_id();

     ObjectWaiter * List = _EntryList ;
     if (List != NULL) {
        assert (List->_prev == NULL, "invariant") ;
        assert (List->TState == ObjectWaiter::TS_ENTER, "invariant") ;
        assert (List != iterator, "invariant") ;
     }

     if (Policy == 0) {       // prepend to EntryList
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
             List->_prev = iterator ;
             iterator->_next = List ;
             iterator->_prev = NULL ;
             _EntryList = iterator ;
        }
     } else
     if (Policy == 1) {      // append to EntryList
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
            // CONSIDER:  finding the tail currently requires a linear-time walk of
            // the EntryList.  We can make tail access constant-time by converting to
            // a CDLL instead of using our current DLL.
            ObjectWaiter * Tail ;
            for (Tail = List ; Tail->_next != NULL ; Tail = Tail->_next) ;
            assert (Tail != NULL && Tail->_next == NULL, "invariant") ;
            Tail->_next = iterator ;
            iterator->_prev = Tail ;
            iterator->_next = NULL ;
        }
     } else
     if (Policy == 2) {      // prepend to cxq
         // prepend to cxq
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
            iterator->TState = ObjectWaiter::TS_CXQ ;
            for (;;) {
                ObjectWaiter * Front = _cxq ;
                iterator->_next = Front ;
                if (Atomic::cmpxchg_ptr (iterator, &_cxq, Front) == Front) {
                    break ;
                }
            }
         }
     } else
     if (Policy == 3) {      // append to cxq
        iterator->TState = ObjectWaiter::TS_CXQ ;
        for (;;) {
            ObjectWaiter * Tail ;
            Tail = _cxq ;
            if (Tail == NULL) {
                iterator->_next = NULL ;
                if (Atomic::cmpxchg_ptr (iterator, &_cxq, NULL) == NULL) {
                   break ;
                }
            } else {
                while (Tail->_next != NULL) Tail = Tail->_next ;
                Tail->_next = iterator ;
                iterator->_prev = Tail ;
                iterator->_next = NULL ;
                break ;
            }
        }
     } else {
        ParkEvent * ev = iterator->_event ;
        iterator->TState = ObjectWaiter::TS_RUN ;
        OrderAccess::fence() ;
        ev->unpark() ;
     }

     if (Policy < 4) {
       iterator->wait_reenter_begin(this);
     }

     // _WaitSetLock protects the wait queue, not the EntryList.  We could
     // move the add-to-EntryList operation, above, outside the critical section
     // protected by _WaitSetLock.  In practice that's not useful.  With the
     // exception of  wait() timeouts and interrupts the monitor owner
     // is the only thread that grabs _WaitSetLock.  There's almost no contention
     // on _WaitSetLock so it's not profitable to reduce the length of the
     // critical section.
  }

  Thread::SpinRelease (&_WaitSetLock) ;

  if (iterator != NULL && ObjectMonitor::_sync_Notifications != NULL) {
     ObjectMonitor::_sync_Notifications->inc() ;
  }
}
```

> 1. `CHECK_OWNER()` 会把以前是偏向锁的，`_owner`设为当前thread
> 2. 如果`_WaitSet`为空，说明没有人wait，直接返回
> 3. 锁住`_WaitSetLock`
> 4. `DequeueWaiter` 出队一个waiter，叫`iterator`
> 5. 设置`_notified`，`_notifier_tid`
> 6. 如果Policy == 0，把 `iterator` 放在_EntryList最前面
> 7. 如果Policy == 1，把`iterator` 放在_EntryList最后面
> 8. 如果Policy == 2，通过cas 把`iterator` 赋给 `_cxq`
> 9. 如果Policy == 3，把`iterator`放在`_cxq`的末尾
> 10. 否则，unpark
> 11. 如果Policy < 4, wait_reenter_begin，这里会设置线程状态为`java_lang_Thread::BLOCKED_ON_MONITOR_ENTER`, **???剩下的没看懂**
> 12. 释放锁，`_sync_Notifications`+1

## ObjectMonitor::notifyAll

```c++
void ObjectMonitor::notifyAll(TRAPS) {
  CHECK_OWNER();
  ObjectWaiter* iterator;
  if (_WaitSet == NULL) {
      TEVENT (Empty-NotifyAll) ;
      return ;
  }
  DTRACE_MONITOR_PROBE(notifyAll, this, object(), THREAD);

  int Policy = Knob_MoveNotifyee ;
  int Tally = 0 ;
  Thread::SpinAcquire (&_WaitSetLock, "WaitSet - notifyall") ;

  for (;;) {
     iterator = DequeueWaiter () ;
     if (iterator == NULL) break ;
     TEVENT (NotifyAll - Transfer1) ;
     ++Tally ;

     // Disposition - what might we do with iterator ?
     // a.  add it directly to the EntryList - either tail or head.
     // b.  push it onto the front of the _cxq.
     // For now we use (a).

     guarantee (iterator->TState == ObjectWaiter::TS_WAIT, "invariant") ;
     guarantee (iterator->_notified == 0, "invariant") ;
     iterator->_notified = 1 ;
     Thread * Self = THREAD;
     iterator->_notifier_tid = Self->osthread()->thread_id();
     if (Policy != 4) {
        iterator->TState = ObjectWaiter::TS_ENTER ;
     }

     ObjectWaiter * List = _EntryList ;
     if (List != NULL) {
        assert (List->_prev == NULL, "invariant") ;
        assert (List->TState == ObjectWaiter::TS_ENTER, "invariant") ;
        assert (List != iterator, "invariant") ;
     }

     if (Policy == 0) {       // prepend to EntryList
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
             List->_prev = iterator ;
             iterator->_next = List ;
             iterator->_prev = NULL ;
             _EntryList = iterator ;
        }
     } else
     if (Policy == 1) {      // append to EntryList
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
            // CONSIDER:  finding the tail currently requires a linear-time walk of
            // the EntryList.  We can make tail access constant-time by converting to
            // a CDLL instead of using our current DLL.
            ObjectWaiter * Tail ;
            for (Tail = List ; Tail->_next != NULL ; Tail = Tail->_next) ;
            assert (Tail != NULL && Tail->_next == NULL, "invariant") ;
            Tail->_next = iterator ;
            iterator->_prev = Tail ;
            iterator->_next = NULL ;
        }
     } else
     if (Policy == 2) {      // prepend to cxq
         // prepend to cxq
         iterator->TState = ObjectWaiter::TS_CXQ ;
         for (;;) {
             ObjectWaiter * Front = _cxq ;
             iterator->_next = Front ;
             if (Atomic::cmpxchg_ptr (iterator, &_cxq, Front) == Front) {
                 break ;
             }
         }
     } else
     if (Policy == 3) {      // append to cxq
        iterator->TState = ObjectWaiter::TS_CXQ ;
        for (;;) {
            ObjectWaiter * Tail ;
            Tail = _cxq ;
            if (Tail == NULL) {
                iterator->_next = NULL ;
                if (Atomic::cmpxchg_ptr (iterator, &_cxq, NULL) == NULL) {
                   break ;
                }
            } else {
                while (Tail->_next != NULL) Tail = Tail->_next ;
                Tail->_next = iterator ;
                iterator->_prev = Tail ;
                iterator->_next = NULL ;
                break ;
            }
        }
     } else {
        ParkEvent * ev = iterator->_event ;
        iterator->TState = ObjectWaiter::TS_RUN ;
        OrderAccess::fence() ;
        ev->unpark() ;
     }

     if (Policy < 4) {
       iterator->wait_reenter_begin(this);
     }

     // _WaitSetLock protects the wait queue, not the EntryList.  We could
     // move the add-to-EntryList operation, above, outside the critical section
     // protected by _WaitSetLock.  In practice that's not useful.  With the
     // exception of  wait() timeouts and interrupts the monitor owner
     // is the only thread that grabs _WaitSetLock.  There's almost no contention
     // on _WaitSetLock so it's not profitable to reduce the length of the
     // critical section.
  }

  Thread::SpinRelease (&_WaitSetLock) ;

  if (Tally != 0 && ObjectMonitor::_sync_Notifications != NULL) {
     ObjectMonitor::_sync_Notifications->inc(Tally) ;
  }
}
```

