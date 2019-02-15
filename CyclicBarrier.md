# CyclicBarrier

和CountDownLatch类似，await到count为0时，即可pass

### 核心代码


```
 ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //当前代，为了可以复用，CountDownLatch不可以复用，只有需要打破barrier的时候，才会修改generation
            final Generation g = generation;
            
            //被打破了
            if (g.broken)
                throw new BrokenBarrierException();

            //线程被打断了
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            //每次await，减少count
            int index = --count;
            //为0的时候才唤醒所有线程
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    //barrier突破的callback
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    //没有设置超时
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

//打破barrier
```
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

### next generation
```
 private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```

总的来说，每个线程跑到await，就await()在这里，然后等最后一个跑进来，count为0，则notifyAll