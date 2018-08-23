## HashMap

#### 1.7

- 为什么需要优化：当 Hash 冲突严重时，在桶上形成的链表会变的越来越长，这样在查询时的效率就会越来越低；时间复杂度为 O(N)。

- HashMap是由数组和链表组成

- put方法

  ```java
  public V put(K key, V value) {
      if (table == EMPTY_TABLE) {
          inflateTable(threshold);
      }
      if (key == null)
          return putForNullKey(value);
      int hash = hash(key);
      int i = indexFor(hash, table.length);
      for (Entry<K,V> e = table[i]; e != null; e = e.next) {
          Object k;
          if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
              V oldValue = e.value;
              e.value = value;
              e.recordAccess(this);
              return oldValue;
          }
      }
      modCount++;
      addEntry(hash, key, value, i);
      return null;
  }
  ```

  - 判断当前数组是否需要初始化。

  - 如果 key 为空，则 put 一个空值进去。

  - 根据 key 计算出 hashcode。

  - 根据计算出的hashcode定位出所在桶。

  - 如果桶是一个链表则需要遍历判断里面的 hashcode、key 是否和传入 key 相等，如果相等则进行覆盖，并返回原来的值。

  - 如果桶是空的，说明当前位置没有数据存入；新增一个 Entry 对象写入当前位置。

  - ``` java
    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        createEntry(hash, key, value, bucketIndex);
    }
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
    ```

  - 如果需要就进行两倍扩充，并将当前的 key 重新 hash 并定位。



- get方法

  - ``` java
    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);
        return null == entry ? null : entry.getValue();
    }
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
    ```

  - 首先也是根据 key 计算出 hashcode，然后定位到具体的桶中。

  - 判断该位置是否为链表。

  - 不是链表就根据 key、key 的 hashcode 是否相等来返回值。

  - 为链表则需要遍历直到 key 及 hashcode 相等时候就返回值。

  - 啥都没取到就直接返回 null。



#### 1.8

- 与1.7的区别：TREEIFY_THRESHOLD 用于判断是否需要将链表转换为红黑树的阈值、HashEntry 修改为 Node。

- put方法

  - ``` java
    public V put(K key, V value) {
            return putVal(hash(key), key, value, false, true);
        }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
            Node<K,V>[] tab; Node<K,V> p; int n, i;
            //1
            if ((tab = table) == null || (n = tab.length) == 0)
                n = (tab = resize()).length;
            //2
            if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
            else {
                Node<K,V> e; K k;
                //3 
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                    e = p;
                //4 
                else if (p instanceof TreeNode)
                    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                //5
                else {
                    for (int binCount = 0; ; ++binCount) {
                        if ((e = p.next) == null) {
                            p.next = newNode(hash, key, value, null);
                            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                                treeifyBin(tab, hash);
                            break;
                        }
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            break;
                        p = e;
                    }
                }
                //6 
                if (e != null) { 
                    V oldValue = e.value;
                    if (!onlyIfAbsent || oldValue == null)
                        e.value = value;
                    afterNodeAccess(e);
                    return oldValue;
                }
            }
            ++modCount;
            //7 
            if (++size > threshold)
                resize();
            afterNodeInsertion(evict);
            return null;
        }
    ```

  - 判断当前桶是否为空，空的就需要初始化（resize 中会判断是否进行初始化）。

  - 根据当前 key 的 hashcode 定位到具体的桶中并判断是否为空，为空表明没有 Hash 冲突就直接在当前位置创建一个新桶即可。

  - 如果当前桶有值（ Hash 冲突），那么就要比较当前桶中的 key、key 的 hashcode 与写入的 key 是否相等，相等就赋值给 e

  - 如果当前桶为红黑树，那就要按照红黑树的方式写入数据。

  - 如果是个链表，就需要将当前的 key、value 封装成一个新节点写入到当前桶的后面（形成链表）。

  - 接着判断当前链表的大小是否大于预设的阈值，大于时就要转换为红黑树。

  - 如果在遍历过程中找到 key 相同时直接退出遍历。

  - 如果 e != null 就相当于存在相同的 key,那就需要将值覆盖。

  - 最后判断是否需要进行扩容。



- get方法

  - ``` java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
    ```

  - 首先将 key hash 之后取得所定位的桶。

  - 如果桶为空则直接返回 null 。

  - 否则判断桶的第一个位置(有可能是链表、红黑树)的 key 是否为查询的 key，是就直接返回 value。

  - 如果第一个不匹配，则判断它的下一个是红黑树还是链表。

  - 红黑树就按照树的查找方式返回值。

  - 不然就按照链表的方式遍历匹配返回值。



- 并发场景下使用时容易出现死循环

  - ``` java
    final HashMap<String, String> map = new HashMap<String, String>();
    for (int i = 0; i < 1000; i++) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                map.put(UUID.randomUUID().toString(), "");
            }
        }).start();
    }
    ```

  - HashMap 扩容的时候会调用 resize() 方法，就是这里的并发操作容易在一个桶上形成环形链表；这样当获取一个不存在的 key 时，计算出的 index 正好是环形链表的下标就会出现死循环。