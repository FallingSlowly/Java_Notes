## ConcurrentHashMap

#### 1.7

- segmant和hashentry组成。

- put方法

  - ``` java
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
    ```

  - 首先是通过 key 定位到 Segment，之后在对应的 Segment 中进行具体的 put。

  - ``` java
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;
            HashEntry<K,V> first = entryAt(tab, index);
            for (HashEntry<K,V> e = first;;) {
                if (e != null) {
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    e = e.next;
                }
                else {
                    if (node != null)
                        node.setNext(first);
                    else
                        node = new HashEntry<K,V>(hash, key, value, first);
                    int c = count + 1;
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                        setEntryAt(tab, index, node);
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();
        }
        return oldValue;
    }
    ```

  - 将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry。

  - 遍历该 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。

  - 不为空则需要新建一个 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容。

  - 最后会解除所获取当前 Segment 的锁。



- get方法

  - ``` java
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
    ```

  - 只需要将 Key 通过 Hash 之后定位到具体的 Segment ，再通过一次 Hash 定位到具体的元素上。

  - 由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。

  - ConcurrentHashMap 的 get 方法是非常高效的，因为整个过程都不需要加锁。



#### 1.8

- 1.7问题：那就是查询遍历链表效率太低。

- 1.8抛弃了原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性。

- put方法

  - ``` java
    public V put(K key, V value) {
            return putVal(key, value, false);
        }
     
        /** Implementation for put and putIfAbsent */
        final V putVal(K key, V value, boolean onlyIfAbsent) {
        		//不允许 key或value为null
            if (key == null || value == null) throw new NullPointerException();
            //计算hash值
            int hash = spread(key.hashCode());
            int binCount = 0;
            //死循环 何时插入成功 何时跳出
            for (Node<K,V>[] tab = table;;) {
                Node<K,V> f; int n, i, fh;
                //如果table为空的话，初始化table
                if (tab == null || (n = tab.length) == 0)
                    tab = initTable();
                //根据hash值计算出在table里面的位置 
                else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                	//如果这个位置没有值 ，直接放进去，不需要加锁
                    if (casTabAt(tab, i, null,
                                 new Node<K,V>(hash, key, value, null)))
                        break;                   // no lock when adding to empty bin
                }
                //当遇到表连接点时，需要进行整合表的操作
                else if ((fh = f.hash) == MOVED)
                    tab = helpTransfer(tab, f);
                else {
                    V oldVal = null;
                    //结点上锁  这里的结点可以理解为hash值相同组成的链表的头结点
                    synchronized (f) {
                        if (tabAt(tab, i) == f) {
                            //fh〉0 说明这个节点是一个链表的节点 不是树的节点
                            if (fh >= 0) {
                                binCount = 1;
                                //在这里遍历链表所有的结点
                                for (Node<K,V> e = f;; ++binCount) {
                                    K ek;
                                    //如果hash值和key值相同  则修改对应结点的value值
                                    if (e.hash == hash &&
                                        ((ek = e.key) == key ||
                                         (ek != null && key.equals(ek)))) {
                                        oldVal = e.val;
                                        if (!onlyIfAbsent)
                                            e.val = value;
                                        break;
                                    }
                                    Node<K,V> pred = e;
                                    //如果遍历到了最后一个结点，那么就证明新的节点需要插入 就把它插入在链表尾部
                                    if ((e = e.next) == null) {
                                        pred.next = new Node<K,V>(hash, key,
                                                                  value, null);
                                        break;
                                    }
                                }
                            }
                            //如果这个节点是树节点，就按照树的方式插入值
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
                        }
                    }
                    if (binCount != 0) {
                    	//如果链表长度已经达到临界值8 就需要把链表转换为树结构
                        if (binCount >= TREEIFY_THRESHOLD)
                            treeifyBin(tab, i);
                        if (oldVal != null)
                            return oldVal;
                        break;
                    }
                }
            }
            //将当前ConcurrentHashMap的元素数量+1
            addCount(1L, binCount);
            return null;
        }
    ```

  - 根据 key 计算出 hashcode 。

  - 判断是否需要进行初始化。

  - f 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。

  - 如果当前位置的 hashcode == MOVED == -1,则需要进行扩容。

  - 如果都不满足，则利用 synchronized 锁写入数据。

  - 如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树。



- get方法

  - ``` java
    public V get(Object key) {
            Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
            //计算hash值
            int h = spread(key.hashCode());
            //根据hash值确定节点位置
            if ((tab = table) != null && (n = tab.length) > 0 &&
                (e = tabAt(tab, (n - 1) & h)) != null) {
                //如果搜索到的节点key与传入的key相同且不为null,直接返回这个节点	
                if ((eh = e.hash) == h) {
                    if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                        return e.val;
                }
                //如果eh<0 说明这个节点在树上 直接寻找
                else if (eh < 0)
                    return (p = e.find(h, key)) != null ? p.val : null;
                 //否则遍历链表 找到对应的值并返回
                while ((e = e.next) != null) {
                    if (e.hash == h &&
                        ((ek = e.key) == key || (ek != null && key.equals(ek))))
                        return e.val;
                }
            }
            return null;
        }
    
    ```

  - 根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。

  - 如果是红黑树那就按照树的方式获取值。

  - 就不满足那就按照链表的方式遍历获取值。



#### 面试常见问题

- 谈谈你理解的 HashMap，讲讲其中的 get put 过程。
- 1.8 做了什么优化？
- 是线程安全的嘛？
- 不安全会导致哪些问题？
- 如何解决？有没有线程安全的并发容器？
- ConcurrentHashMap 是如何实现的？ 1.7、1.8 实现有何不同？为什么这么做？