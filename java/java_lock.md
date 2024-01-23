#Java 锁

## 一、乐观锁 VS 悲观锁
乐观锁和悲观锁是基于对数据的并发操作的态度，悲观锁认为自己在操作数据时不允许其它线程同时操作数据，最常见的悲观锁：syncchronized和lock实现类都是悲观锁
这两个关键字所触达的是独享数据的操作，也是在实际工作中使用最多的锁。
<br>

乐观锁：自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据时判断有没有别的线程更新了这个数据。如果没有更新这个数据，当前线程将自己修改的数据成功写入
如果数据被其他线程修改，则根据不同的方式实现不同的操作。

乐观锁在Java中通过无锁编程来实现，最常用的是CAS算法，Java原子类中的递增操作就是CAS自旋实现。


### 1.1 自旋锁
自旋锁就是对比和交换，核心逻辑是CAS机制：
1. 如果内存中的值和原来预存的值相等，就将修改后的值保存到内存中
2. 如果内存中的值和预期的值不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作，直到重试成功。


实例：**AtomicInteger**
```
// 创建Unsafe类的实例
private static final Unsafe unsafe = Unsafe.getUnsafe();
// 成员变量value是在内存地址中距离当前对象首地址的偏移量, 具体赋值是在下面的静态代码块中中进行的
private static final long valueOffset;

static { 
   
    try { 
   
        // 类加载的时候，在静态代码块中获取变量value的偏移量
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { 
    throw new Error(ex); }
}
// 当前AtomicInteger原子类的value值
private volatile int value;

// 类似于i++操作
public final int getAndIncrement() { 
   
	// this代表当前AtomicInteger类型的对象，valueOffset表示value成员变量的偏移量
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
================================上方为AtomicInteger类中的方法，下方为Unsafe类中的方法=========================================================
// 此方法的作用：获取内存地址为原子对象首地址+原子对象value属性地址偏移量, 并将该变量值加上delta
public final int getAndAddInt(Object obj, long offset, int delta) { 
   
    int v;
    do { 
   
    	// 通过对象和偏移量获取变量值作为期望值，在修改该内存偏移位置的值时与原始进行比较
    	// 此方法中采用volatile的底层原理,保证了内存可见性，所有线程都从内存中获取变量vlaue的值，所有线程看到的值一致。
        v= this.getIntVolatile(obj, offset);
    /* while中的compareAndSwapInt()方法尝试修改v的值,具体地, 该方法也会通过obj和offset获取变量的值 如果这个值和v不一样, 说明其他线程修改了obj+offset地址处的值, 此时compareAndSwapInt()返回false, 继续循环 如果这个值和v一样, 说明没有其他线程修改obj+offset地址处的值, 此时可以将obj+offset地址处的值改为v+delta, compareAndSwapInt()返回true, 退出循环 Unsafe类中的compareAndSwapInt()方法是原子操作, 所以compareAndSwapInt()修改obj+offset地址处的值的时候不会被其他线程中断 */
    } while(!this.compareAndSwapInt(obj, offset, v, v + delta));

    return v;
}
```

如果直接使用private volatile int value;，只能保证可见性，无法保证原子性，也就是单纯使用volatile无法保证数据多线程的同步问题。
这里穿插一个知识点，为何volatile不能保证原子性？

**volatile特性**
1. 保证可见性
2. 不保证原子性
3. 禁止指令重排


为何不保证原子性，原子性如何定义？
```
public volatile int num = 0;
public void add() {
	num ++;
}
```

多线程进行++运算时，无法保证指令级别的原子操作，A线程从主存中拿到原始值，在进行++运算时，当前线程可能会被中断，CPU切换到其他线程，导致另外一个线程拿到的也是相同的初始值进行++计算，最终2个线程的计算结果就无法保证原子性。

原子性：原子性指的是一个操作是不可分割、不可中断的，要么全部执行并且执行的过程不会被任何因素打断，要么就全不执行。

AtomicInteger是如何实现原子性的，通过CAS自旋锁如何实现原子性？这就是自旋锁的原理，通过volatile和while循环判断，来解决原子性。
```
    /**
     * Atomically adds the given value to the current value of a field
     * or array element within the given object {@code o}
     * at the given {@code offset}.
     *
     * @param o object/array to update the field/element in
     * @param offset field/element offset
     * @param delta the value to add
     * @return the previous value
     * @since 1.8
     */
    @IntrinsicCandidate
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));
        return v;
    }
```

在回顾一下自旋锁的核心思想：对比和交换：
1. 如果内存中的值和原来预存的值相等，就将修改后的值保存到内存中
2. 如果内存中的值和预期的值不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作，直到重试成功。


weakCompareAndSetInt是汇编层保证的原子操作：

```
inline jint Atomic::cmpxchg(jint exchange_value, volatile jint* dest, jint compare_value) {
	int mp = os::is_MP()；
	__asm {
		mov edx, dest
		mov ecx, exchange_value
		mov eax, compare_value
		LOCK_IF_MP(mp)
		cmpxchg dword ptr[edx],ecx
	}
}
```

cmpxchg 是 intel CPU 指令集中的一条指令， 这条指令经常用来实现原子锁， 我们来看 intel 文档中对这条指令的介绍：

Compares the value in the AL, AX, EAX, or RAX register with the first operand (destination operand). If the two values are equal, the second operand (source operand) is loaded into the destination operand. Otherwise, the destination operand is loaded into the AL, AX, EAX or RAX register. RAX register is available only in 64-bit mode.

参见文章:[Linux 中的 cmpxchg 宏](https://coderatwork.cn/posts/linux-cmpxchg/)
```
cmpxchg(ptr, old, new)
{
    if (*ptr == __old) {
        *ptr == __new;
        return __old;
    } else {
        return __new;
    }
}
```
这就对应上了，在实际的工作中，什么场景下才会用到这种自旋锁取代悲观锁：syncchronized和lock
比如实际工作中使用比较多的集合：ArrayList、HashMap，这些都是线程不安全的，在实际项目中大多是多线程读写，如何处理，直接使用Collections.synchronizedList悲观锁来处理吗，当然这样是没问题的，但是是否还有更高效的方式？

SynchronizedList实现方式：采用synchronized关键字对需要改变数据的方法加锁，这个锁需要内核态去切换，对性能上是有一定的损耗。
```
static class SynchronizedList<E>
        extends SynchronizedCollection<E>
        implements List<E> {
        @java.io.Serial
        private static final long serialVersionUID = -7754090372962971524L;

        @SuppressWarnings("serial") // Conditionally serializable
        final List<E> list;

        SynchronizedList(List<E> list) {
            super(list);
            this.list = list;
        }
        SynchronizedList(List<E> list, Object mutex) {
            super(list, mutex);
            this.list = list;
        }

        public boolean equals(Object o) {
            if (this == o)
                return true;
            synchronized (mutex) {return list.equals(o);}
        }
        public int hashCode() {
            synchronized (mutex) {return list.hashCode();}
        }

        public E get(int index) {
            synchronized (mutex) {return list.get(index);}
        }
        public E set(int index, E element) {
            synchronized (mutex) {return list.set(index, element);}
        }
        public void add(int index, E element) {
            synchronized (mutex) {list.add(index, element);}
        }
        public E remove(int index) {
            synchronized (mutex) {return list.remove(index);}
        }

```

如果不用synchronized，那么用自旋锁是否可以实现ArrayList上锁，但是list的操作又不是可见性，也不具有原子性，使用自旋锁的思想.

思路：使用链表+自旋锁实现列表
```

import java.util.concurrent.atomic.AtomicReference;

public class Node<T> {
    private AtomicReference<Node<T>> next = null;
    private T d;
    private int size;

    public AtomicReference<Node<T>> getNext() {
        return next;
    }

    public void setNext(AtomicReference<Node<T>> next) {
        this.next = next;
    }

    public int getSize() {
        return size;
    }

    public void setSize(int size) {
        this.size = size;
    }

    public T getD() {
        return d;
    }

    public void setD(T d) {
        this.d = d;
    }
}

```

```

import java.util.concurrent.atomic.AtomicReference;

public class MessageList<T> {
    Node<T> first;
    Node<T> end;

    private Node<T> nextNode = new Node<>();
    MessageList() {
        first = new Node<>();
        first.setNext(new AtomicReference<>(nextNode));
        end = first;
    }

    public void add(Node<T> data) {
        System.out.println("add node:" + data.getD() + " Thead Name:" + Thread.currentThread().getName());
        do {
            data.setSize(end.getSize() + 1);
            data.setNext(new AtomicReference<>(nextNode));
        } while (!end.getNext().compareAndSet(nextNode, data));
         end = data;
         System.out.println("add node success:" + data.getSize() + " msg " +  data.getD() + " Thead Name:" + Thread.currentThread().getName());
    }

    public Node<T> get() {
        if (first.getNext().get().getD() != null) {
            first = first.getNext().get();
            System.out.println("get node success:" + first.getD() + " Thead Name:" + Thread.currentThread().getName());
            return first;
        }
        return null;
    }
}

```

CAS虽然高效，但是也存在如下3个问题：
1. ABA问题：CAS在操作时是进行内存值比较，如果发现值相等会进行赋值，如果原来值为A，后来变成了B，然后又变成了A，那么CAS进行检查时会发现值没有发生变化，但是实际上是有变化的。ABA问题的解决思路就是在变量前面添加版本号，每次变量更新的时候都把版本号加一，这样变化过程就从“A－B－A”变成了“1A－2B－3A”。
JDK从1.5开始提供了AtomicStampedReference类来解决ABA问题，具体操作封装在compareAndSet()中。compareAndSet()首先检查当前引用和当前标志与预期引用和预期标志是否相等，如果都相等，则以原子方式将引用值和标志的值设置为给定的更新值。
2. 循环时间长开销大：CAS操作如果长时间不成功，会导致一直自旋，给CPU带来非常大的开销、
3. 只能保证一个共享变量的原子操作。对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。
Java从1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。


## 公平锁 VS 非公平锁
公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。

公平锁和非公平锁实现类：ReentrantLock，ReentrantLock里面有一个内部类Sync，Sync继承AQS（AbstractQueuedSynchronizer），添加锁和释放锁的大部分操作实际上都是在Sync中实现的。它有公平锁FairSync和非公平锁NonfairSync两个子类。ReentrantLock默认使用非公平锁，也可以通过构造器来显示的指定使用公平锁。

```
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs non-fair tryLock.
         */
        @ReservedStackAccess
        final boolean tryLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, 1)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (getExclusiveOwnerThread() == current) {
                if (++c < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(c);
                return true;
            }
            return false;
        }

        /**
         * Checks for reentrancy and acquires if lock immediately
         * available under fair vs nonfair rules. Locking methods
         * perform initialTryLock check before relaying to
         * corresponding AQS acquire methods.
         */
        abstract boolean initialTryLock();

        @ReservedStackAccess
        final void lock() {
            if (!initialTryLock())
                acquire(1);
        }

        @ReservedStackAccess
        final void lockInterruptibly() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            if (!initialTryLock())
                acquireInterruptibly(1);
        }

        @ReservedStackAccess
        final boolean tryLockNanos(long nanos) throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            return initialTryLock() || tryAcquireNanos(1, nanos);
        }

        @ReservedStackAccess
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (getExclusiveOwnerThread() != Thread.currentThread())
                throw new IllegalMonitorStateException();
            boolean free = (c == 0);
            if (free)
                setExclusiveOwnerThread(null);
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final boolean initialTryLock() {
            Thread current = Thread.currentThread();
            if (compareAndSetState(0, 1)) { // first attempt is unguarded
                setExclusiveOwnerThread(current);
                return true;
            } else if (getExclusiveOwnerThread() == current) {
                int c = getState() + 1;
                if (c < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(c);
                return true;
            } else
                return false;
        }

        /**
         * Acquire for non-reentrant cases after initialTryLock prescreen
         */
        protected final boolean tryAcquire(int acquires) {
            if (getState() == 0 && compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
    }

    /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        /**
         * Acquires only if reentrant or queue is empty.
         */
        final boolean initialTryLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedThreads() && compareAndSetState(0, 1)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (getExclusiveOwnerThread() == current) {
                if (++c < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(c);
                return true;
            }
            return false;
        }

        /**
         * Acquires only if thread is first waiter or empty
         */
        protected final boolean tryAcquire(int acquires) {
            if (getState() == 0 && !hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
    }

    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    /**
     * Acquires the lock.
     *
     * <p>Acquires the lock if it is not held by another thread and returns
     * immediately, setting the lock hold count to one.
     *
     * <p>If the current thread already holds the lock then the hold
     * count is incremented by one and the method returns immediately.
     *
     * <p>If the lock is held by another thread then the
     * current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired,
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        sync.lock();
    }
```

公平锁实现核心逻辑：
```
        /**
         * Acquires only if thread is first waiter or empty
         */
        protected final boolean tryAcquire(int acquires) {
            if (getState() == 0 && !hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

```

非公平锁实现核心逻辑：
```
      /**
         * Acquire for non-reentrant cases after initialTryLock prescreen
         */
        protected final boolean tryAcquire(int acquires) {
            if (getState() == 0 && compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
```

两者的区别是非公平锁通过hasQueuedPredecessors判断当前队列是否有在等待的需要唤醒的线程，如果有则排队
非公平锁直接尝试获取锁资源，获取不到在去排队，减少了线程唤醒的消耗。
综上，公平锁就是通过同步队列来实现多个线程按照申请锁的顺序来获取锁，从而实现公平的特性。非公平锁加锁时不考虑排队等待问题，直接尝试获取锁，所以存在后申请却先获得锁的情况。

## 可重入锁 VS 不可重入锁

可重入锁又名递归锁是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。下面用示例代码来进行分析：

```
public class View {
	public synchronized void doOne() {
		doOhters()

	}

	public synchronized void doOhters() {

	}

}
```

在上面的代码中，类中的两个方法都是synchronized修饰的独享锁，在doOne中调用doOthers时，因为syncchronized是一个可重入锁，同一个线程在调用doOthers时可以直接获取到对象锁。
如果是一个非重入锁，当前线程在调用doOthers时需要等待doOne释放掉已经获取的对象锁，实现上对象锁已经被当前线程获取，无法释放，导致了死锁。

可重入锁通过status的引用计数方式对锁的状态进行累加，释放时累减。
当线程尝试获取锁时，可重入锁先尝试获取并更新status值，如果status == 0表示没有其他线程在执行同步代码，则把status置为1，当前线程开始执行。如果status != 0，则判断当前线程是否是获取到这个锁的线程，如果是的话执行status+1，且当前线程可以再次获取锁。而非可重入锁是直接去获取并尝试更新当前status的值，如果status != 0的话会导致其获取锁失败，当前线程阻塞。

释放锁时，可重入锁同样先获取当前status的值，在当前线程是持有锁的线程的前提下。如果status-1 == 0，则表示当前线程所有重复获取锁的操作都已经执行完毕，然后该线程才会真正释放锁。而非可重入锁则是在确定当前线程是持有锁的线程之后，直接将status置为0，将锁释放


## 独享锁 VS 共享锁
独享锁也叫排他锁，是指该锁一次只能被一个线程所持有。如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。JDK中的synchronized和JUC中Lock的实现类就是互斥锁。

共享锁是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

独享锁与共享锁也是通过AQS来实现的，通过实现不同的方法，来实现独享或者共享。

共享锁：
ReentrantReadWriteLock
```
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

  ```

  读锁：
  ```
      public static class ReadLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses.
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        /**
         * Acquires the read lock.
         *
         * <p>Acquires the read lock if the write lock is not held by
         * another thread and returns immediately.
         *
         * <p>If the write lock is held by another thread then
         * the current thread becomes disabled for thread scheduling
         * purposes and lies dormant until the read lock has been acquired.
         */
        public void lock() {
            sync.acquireShared(1);
        }
  ```

  写锁：
  ```
     public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses.
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        /**
         * Acquires the write lock.
         *
         * <p>Acquires the write lock if neither the read nor write lock
         * are held by another thread
         * and returns immediately, setting the write lock hold count to
         * one.
         *
         * <p>If the current thread already holds the write lock then the
         * hold count is incremented by one and the method returns
         * immediately.
         *
         * <p>If the lock is held by another thread then the current
         * thread becomes disabled for thread scheduling purposes and
         * lies dormant until the write lock has been acquired, at which
         * time the write lock hold count is set to one.
         */
        public void lock() {
            sync.acquire(1);
        }
  ```

  核心区别：
  ```
       public void lock() { // 读锁是共享的
            sync.acquireShared(1);
        }

       public void lock() { // 写锁是独享的
            sync.acquire(1);
        }
  ```

  在ReentrantReadWriteLock里面，读锁和写锁的锁主体都是Sync，但读锁和写锁的加锁方式不一样。读锁是共享锁，写锁是独享锁。读锁的共享锁可保证并发读非常高效，而读写、写读、写写的过程互斥，因为读锁和写锁是分离的。所以ReentrantReadWriteLock的并发性相比一般的互斥锁有了很大提升。

那读锁和写锁的具体加锁方式有什么区别呢？在了解源码之前我们需要回顾一下其他知识。 在最开始提及AQS的时候我们也提到了state字段（int类型，32位），该字段用来描述有多少线程获持有锁。

在独享锁中这个值通常是0或者1（如果是重入锁的话state值就是重入的次数），在共享锁中state就是持有锁的数量。但是在ReentrantReadWriteLock中有读、写两把锁，所以需要在一个整型变量state上分别描述读锁和写锁的数量（或者也可以叫状态）。于是将state变量“按位切割”切分成了两个部分，高16位表示读锁状态（读锁个数），低16位表示写锁状态（写锁个数）

```
|---------------32位--------------|
|0000000000000000|0000000000000000|
|--高十六位：读----|--低十六位：写---|
```

## 结语
综上所属，锁从不同的属性有不同的锁：
悲观锁、乐观锁；
可重入锁、不可重入锁；
独享锁、共享锁；
公平锁；不公平锁;
针对不同的场景使用不同的锁，会提高锁切换的性能，在日常工作中需要多思考多线程并发时锁的选择以及性能的问题。

[参考文章](https://tech.meituan.com/2018/11/15/java-lock.html)


