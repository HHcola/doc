## ThreadLocal

### ThreadLocal
这个类可以使线程绑定的对象关联起来，可以实现同线程的数据的访问，通过get方法可以返回当前线程绑定的值，而不会获取到其他线程的值。
可以做到线程的隔离，不同的线程都会存储一个副本。


### 源码解析
具体实现逻辑是使用ThreadLocalMap存储
```
     static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

这里的key是一个弱引用的threadLocal对象，在线程对象被回收后这个map对象也会被回收。

#### set
```
   public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

使用set时，如果map不存在，则创建Thread想关联的map

```
   void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

这里的set使用的是散列hash值，线性探测的方式存储，当hash值相同时引用的对象一致时替换原来的值，如果原来引用的对象被回收，则刷新引用的对象。
如果引用的对象不一致时，则通过线性探测的方式寻找下一个位置

```
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {

                // Android-changed: Use refersTo() (twice).
                // ThreadLocal<?> k = e.get();
                // if (k == key) { ... } if (k == null) { ... }
                if (e.refersTo(key)) {
                    e.value = value;
                    return;
                }

                if (e.refersTo(null)) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

寻找下一个存储的位置：
```
  private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

```
如果i +1 < len，则直接使用i + 1，否则使用0


### get

通过当前线程获取到线程的成员变量：threadLocals
在获取对应的值，如果threadLocals没有初始化，则初始化threadLocals并返回null
```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

### 扩容

扩容为原来的2倍：newLen = oldLen * 2;

```
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```