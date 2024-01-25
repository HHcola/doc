#Java 集合

Java集合是一个庞大的家族，包含很多类，但是从根源去梳理，也可以很清晰的梳理清楚。集合顶层分为Collection和Map，然后是派生类，具体如下：
Collection家族：

```
                                         Collection

                     List               |          Set            |                     Queue

Vector      ArrayList     LinkedList    |    HashSet    TreeSet   |   ConcurrentLinkQueue     BlockingQueue

Stack                                   |     LinkedHashSet       |  
```

Map家族：
```
                  Map
       HashMap           TreeMap
    LinkedHashMap
```


## List集合

### 1.1 ArrayList

ArrayList是最基础的集合，底层基于数组实现：transient Object[] elementData;
需要讨论几个：
1. ArrayList是线程安全的吗？
2. ArrayList默认容量是多少？如何扩容
3. ArrayList的优势有那些？

#### ArrayList的线程安全性
ArrayList本身不具有线程安全性的，ArrayList的实现源码里并没有对读写加锁。
如果要实现ArrayList的线程安全，要怎么做？
1. 使用Collections.synchronizedList包装成线程安全list
2. 使用Vertor替代ArrayList
3. 使用CopyOnWriteArrayList
JDK1.5版本实现了CopyOnWriteArrayList，对与elementData的写和操作进行了加锁

```
    public E set(int index, E element) {
        synchronized (lock) {
            Object[] es = getArray();
            E oldValue = elementAt(es, index);

            if (oldValue != element) {
                es = es.clone();
                es[index] = element;
            }
            // Ensure volatile write semantics even when oldvalue == element
            setArray(es);
            return oldValue;
        }
    }
```

synchronizedList会对方法加可重入锁，这种操作线程安全，但是性能不高：
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

        public int indexOf(Object o) {
            synchronized (mutex) {return list.indexOf(o);}
        }
        public int lastIndexOf(Object o) {
            synchronized (mutex) {return list.lastIndexOf(o);}
        }

        public boolean addAll(int index, Collection<? extends E> c) {
            synchronized (mutex) {return list.addAll(index, c);}
        }
```

CopyOnWriteArrayList:在添加数据时，使用对add方式使用lock锁，然后重新copy一份列表，把数据添加到末尾，在设置回去。
```
    public boolean add(E e) {
        synchronized (lock) {
            Object[] es = getArray();
            int len = es.length;
            es = Arrays.copyOf(es, len + 1);
            es[len] = e;
            setArray(es);
            return true;
        }
    }
```

这种实现方式的优势在哪里？
CopyOnWriteArrayList对于读并没有加锁，如果使用SynchronizedList会对读也加锁，那么对于频繁读的ArrayList的场景可以使用CopyOnWriteArrayList，但是对于频繁写的操作就不适合。
```
  public E get(int index) {
        return elementAt(getArray(), index);
    }
```

CopyOnWriteArrayList的缺点：
1. 内存占用高
2. 实时性不高：在数据进行copy时，并不能保证多个线程获取的array是一致性的


### 1.2 LinkedList
LinkedList双向链表实现的队列：
1. LinkedList是双向链表实现的List
2. LinkedList是非线程安全的：需要使用SynchronizedList实现线程安全
3. LinkedList元素允许为null，允许重复元素
4. LinkedList是基于链表实现的，因此插入删除效率高，查找效率低(虽然有一个加速动作)
5. LinkedList是基于链表实现的，因此不存在容量不足的问题，所以没有扩容的方法
6. LinkedList还实现了栈和队列的操作方法，因此也可以作为栈、队列和双端队列来使用

```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    transient int size = 0;

    /**
     * Pointer to first node.
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     */
    transient Node<E> last;
```


## Map

### 1.1 HashMap
问题：
1. HashMap底层原理
2. 为什么使用数组+链表
3. 可以用LinkedList代替数组吗？为何不使用
4. 如何扩容？

```
默认初始容量(数组默认大小):16，2的整数次方
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;


默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
装载因子用来衡量HashMap满的程度，表示当map集合中存储的数据达到当前数组大小的75%则需要进行扩容

 

链表转红黑树边界
static final int TREEIFY_THRESHOLD = 8;

 
红黑树转离链表边界
static final int UNTREEIFY_THRESHOLD = 6;


哈希桶数组
transient Node<K,V>[] table;

 
实际存储的元素个数
transient int size;


当map里面的数据大于这个threshold就会进行扩容
int threshold   阈值 = table.length * loadFactor
```
可以用LinkedList代替数组吗？是可以使用的，为何不使用？
LinkedList是链表结构，使用数组查找效率更高，HashMap使用key的hash值对数组长度取模，此时已经得到了节点的位置，查找效率更高。
但是为何不用ArrayList，而是直接使用了数组，ArrayList查询效率也很高，因为HashMap的扩容刚好是2的幂，做取模效率高，但是ArrayList扩容机制是1.5倍。


### 1.2 ConcurrentHashMap
HashMap是非线程安全的，如果需要线程安全，Collections的synchronizedMap或者使用ConcurrentHashMap,但是ConcurrentHashMap和Collections的synchronizedMap有什么区别

JDK1.7版本：ConcurrentHashMap使用Segment数组结构和 HashEntry 数组结构组成,采用Segment分段加锁机制保证了数据的线程安全。
JDK1.8版本：ConcurrentHashMap使用synchronized 和 CAS 和 HashEntry 和红黑树，取消了Segment分段锁的数据结构，取而代之的是Node数组+链表+红黑树的结构
```
//第一次put 初始化 Node 数组
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

```
public V put(K key, V value) {
        return putVal(key, value, false);
    }
    /** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value))) // 通过CAS对于空的tab加锁
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
                V oldVal = null;
                synchronized (f) { // 通过synchronized进行加锁
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
```
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSetReference(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```

### 1.3 LinkedHashMap
LinkedHashMap是HashMap的一个子类，保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。
实现原理：HashMap+双向链表


### 1.4 TreeMap
TreeMap实现SortedMap接口，底层是用红黑树实现,能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。

```
static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;

        /**
         * Make a new cell with given key, value, and parent, and with
         * {@code null} child links, and BLACK color.
         */
        Entry(K key, V value, Entry<K,V> parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        /**
         * Returns the key.
         *
         * @return the key
         */
        public K getKey() {
            return key;
        }

        /**
         * Returns the value associated with the key.
         *
         * @return the value associated with the key
         */
        public V getValue() {
            return value;
        }

        /**
         * Replaces the value currently associated with the key with the given
         * value.
         *
         * @return the value associated with the key before this method was
         *         called
         */
        public V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }
```

put()方法：遍历树的操作，根据排序器的规则遍历，找到对应的位置插入
```
private V put(K key, V value, boolean replaceOld) {
        Entry<K,V> t = root;
        if (t == null) {
            addEntryToEmptyMap(key, value);
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else {
                    V oldValue = t.value;
                    if (replaceOld || oldValue == null) {
                        t.value = value;
                    }
                    return oldValue;
                }
            } while (t != null);
        } else {
            Objects.requireNonNull(key);
            @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else {
                    V oldValue = t.value;
                    if (replaceOld || oldValue == null) {
                        t.value = value;
                    }
                    return oldValue;
                }
            } while (t != null);
        }
        addEntry(key, value, parent, cmp < 0);
        return null;
    }
```
### 1.5 Hashtable
Hashtable是遗留类，很多映射的常用功能与HashMap类似，不同的是它承自Dictionary类，并且是线程安全的，任一时间只有一个线程能写Hashtable，并发性不如ConcurrentHashMap，因为ConcurrentHashMap引入了分段锁。Hashtable不建议在新代码中使用，不需要线程安全的场合可以用HashMap替换，需要线程安全的场合可以用ConcurrentHashMap替换。



## Set
### 1.1 HashSet
HashSet底层原理完全就是包装了一下HashMap
HashSet的唯一性保证是依赖与hashCode()和equals()两个方法，所以存入对象的时候一定要自己重写这两个方法来设置去重的规则。
因为底层是HashMap，而存储的又是key，所以没有get()方法来直接获取，只能遍历获取
```
  private transient HashMap<E,Object> map;

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

### 1.2 LinkedHashSet
LinkedHashSet实现是利用LinkedHashMap实现保证顺序的：
```
 HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }

  public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }
```

### 1.3 TreeSet
TreeSet也完全依赖于TreeMap来实现

## Queue
### 1.1 ConcurrentLinkedQueue
ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，它会添加到队列的尾部，当我们获取一个元素时，它会返回队列头部的元素

### 1.2 BlockingQueue