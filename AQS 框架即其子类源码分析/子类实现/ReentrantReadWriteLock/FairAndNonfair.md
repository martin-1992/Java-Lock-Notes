### FairSync
　　公平模式下，读锁和写锁都需要判断队列中是否有节点线程在等待获取锁，如果有，则将当前线程加入队列末尾中。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```

#### hasQueuedPredecessors
　　返回 true 有两种情况，只有 false 才可以继续尝试获取锁，true 是将当前线程包装成节点添加到同步队列末尾。
  
- 同步队列不为空，有等待的节点（线程），且头节点的下个节点为空；
- 同步队列不为空，节点线程不为当前线程，当前线程要加入等待队列。

```java
public final boolean hasQueuedPredecessors() {

    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

### NonfairSync
　　非公平模式下，写锁无需判断队列中是否有等待线程，直接使用 CAS 尝试获取锁，获取失败才进入同步队列添加到末尾。而读锁会进行判断，队列中头节点的下个节点 head.next 是否为获取写锁的线程，如果是，则不使用 CAS 抢锁，先让队列中的线程获取写锁。

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
         * block if the thread that momentarily appears to be head
         * of queue, if one exists, is a waiting writer.  This is
         * only a probabilistic effect since a new reader will not
         * block if there is a waiting writer behind other enabled
         * readers that have not yet drained from the queue.
         */
        return apparentlyFirstQueuedIsExclusive();
    }
}
```

#### apparentlyFirstQueuedIsExclusive
　　返回 true 的情况，即该阻塞队列中的头节点的下个节点线程为独占模式，前面提到过独占模式为写锁，这里是让写锁先获取锁，避免写锁一直等待。

- 头节点不为空；
- 头节点的下个节点不为空且为独占模式；
- 头节点的下个节点线程不为空；

```java
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```
