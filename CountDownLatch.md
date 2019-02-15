# CountDownLatch

### core code
```
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```
CountDownLatch is a wrapper of class sync


```
public void countDown() {
    sync.releaseShared(1);
}
```
release时，去用cas减sync的state的值
```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
acquireSharedInterruptibly中会自旋得检查state的值是否为0。

总的来说，就是都使用shared模式，自旋得acquire state为0的情况