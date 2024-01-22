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
public class Node<T> {
	private AtomicReference<Node<T>> next;
	private T d;
	private int size;

}

public MessageList {
	Node<String> first;
	Node<String> end;

	public void add(Node<String> data) {
		AtomicReference<Node<String>> dataRef = new AtomicReference<>(data);
		while(true) {
			AtomicReference<Node<String>> oldValue = end.next.get();
			if (data.ref.compareAndSet(oldValue, dataRef)) {
				data.size = oldValue.size +1;
				break;
			}

		}
	}
}
```

