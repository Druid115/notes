## HashMap

**Hash** 也称散列、哈希，基本原理就是把任意长度的输入，通过 Hash 算法变成固定长度的输出。原始数据映射后的二进制串就是哈希值。Hash 的特点：

- 从 Hash 值不可以反向推导出原始的数据；
- 哈希算法的执行效率要高效，长的文本也能快速地计算出哈希值；
- 哈希算法的冲突概率要小。

冲突：由于 Hash 的原理是将输入空间的值映射成 Hash 空间内，而 Hash 空间远小于输入空间，根据抽屉原理，一定会存在将不同输入映射成相同输出的情况。



### 静态常量

~~~java
/**
 * The default initial capacity - MUST be a power of two.
 *
 * 数组默认长度
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 *
 * 数组最大长度
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor.
 *
 * 默认负载因子
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2 and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 *
 * 树化阈值
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 *
 * 树降级成为链表的阈值
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 *
 * 当哈希表中所有元素超过该值，才会允许树化
 */
static final int MIN_TREEIFY_CAPACITY = 64;
~~~



### 核心属性

~~~java
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 *
 * 哈希表
 */
transient Node<K,V>[] table;

/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 *
 * 键值对集合
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * The number of key-value mappings contained in this map.
 *
 * 当前元素个数
 */
transient int size;

/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 *
 * 当前哈希表结构修改次数
 */
transient int modCount;

/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 *
 * 扩容阈值
 */
// (The javadoc description is true upon serialization.
// Additionally, if the table array has not been allocated, this
// field holds the initial array capacity, or zero signifying
// DEFAULT_INITIAL_CAPACITY.)
int threshold;

/**
 * The load factor for the hash table.
 *
 * @serial
 *
 * 负载因子
 */
final float loadFactor;
~~~



### 构造函数

~~~java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and the default load factor (0.75).
 *
 * @param  initialCapacity the initial capacity.
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
~~~



### 静态工具方法

~~~java
/**
 * Computes key.hashCode() and spreads (XORs) higher bits of hash
 * to lower.  Because the table uses power-of-two masking, sets of
 * hashes that vary only in bits above the current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact of higher bits
 * downward. There is a tradeoff between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way to reduce systematic lossage, as well as
 * to incorporate impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 *
 * 让 hash 值的高 16 位也参与路由运算，减少 hash 碰撞，散列更加均匀
 *
 * 0b 1100 0010 0010 1101 1111 0101 1010 1111
 * ^
 * 0b 0000 0000 0000 0000 1100 0010 0010 1101
 * => 1100 0010 0010 1101 0011 0111 1000 0010
 *
 * 寻址公式为（length - 1）& hash，在数组容量比较低的时候，高位没有参与运算，增大了碰撞的可能性
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**
 * Returns a power of two size for the given target capacity.
 *
 * 返回一个大于等于当前值的数字，该数字一定是 2 的次方
 *
 * 0001 0110 1100 => 0001 1111 1111 + 1 => 0010 0000 0000
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
~~~



### put 方法

~~~java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

/**
 * Implements Map.put and related methods.
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    /**
     * tab 引用当前散列表数组
     * p 散列表数组当前的元素
     * n 散列表数组的长度
     * i 路由寻址结果
     */
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 延迟初始化，第一次调用 putVal 方法时会初始化 HashMap 对象中最耗内存的散列表
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    
    else {
        /**
         * e 定位与要插入元素的键值一样的元素
         * k 临时 key
         */
        Node<K,V> e; K k;
        
        // 桶位中的元素的键值与要插入元素的键值相等
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 桶位中存放的是红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // 桶位中为链表，且头元素键值与插入元素键值不一致
        else {
            for (int binCount = 0; ; ++binCount) {
                // 尾插
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 检查链表长度是否需要树化
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
        
        // 找到与要插入元素的键值一样的元素，替换其值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 记录结构修改次数，替换元素的值不纳入计数
    ++modCount;
    // 检查是否需要扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
~~~



### resize 方法

~~~java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 *
 * 为了解决哈希冲突导致的链化影响查询效率的问题
 */
final HashMap.Node<K,V>[] resize() {
    // 扩容前的 hash 表
    HashMap.Node<K,V>[] oldTab = table;
    // 扩容前的数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 扩容前的扩容阈值
    int oldThr = threshold;
    
    int newCap, newThr = 0;
    // Hash 已经初始化过，是一次正常扩容
    if (oldCap > 0) {
        // 数组大小已达到最大阈值，无法再次扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 1. oldCap 左移一位实现扩容，当 newCap 小于最大阈值且扩容前数组大小大于 16，下次扩容阈值为当前阈值两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    
    // 2. HashMap 通过有参构造方法构造（new HashMap(map) map 不为空也可），并未初始化
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    
    // HashMap 通过无参构造方法构造
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // 1、2 两种情况需要通过计算得到新的扩容阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    @SuppressWarnings({"rawtypes","unchecked"})
    HashMap.Node<K,V>[] newTab = (HashMap.Node<K,V>[])new HashMap.Node[newCap];
    table = newTab;
    // 扩容前数组不为 null
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            // e 当前节点
            HashMap.Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                
                // 释放原数组空间，方便 JVM GC
                oldTab[j] = null;
                
                // 当前桶位只有一个元素，直接计算出元素在新数组的位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                
                // 当前桶位已经树化
                else if (e instanceof HashMap.TreeNode)
                    ((HashMap.TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                
                // 当前桶位存储的是链表
                else { // preserve order
                    /**
                     * 扩容前：
                     * oldCap = 1000
                     * hash & (oldCap - 1) = 1111 (15)
                     * hash 可以确定 。。。?1111 ? 只有 1 或 0
                     *
                     * 扩容后：
                     * newCap = 10000
                     * hash & (oldCap - 1) = 01111 (15) 或 11111 (31) 
                     * 
                     * 新的位置由高位决定
                     **/
                    // 低位链表：扩容后存储的下标位置与原下标位置一致
                    HashMap.Node<K,V> loHead = null, loTail = null;
                    // 高位链表：扩容后存储的下标位置为原下标位置 + 扩容前数组的长度
                    HashMap.Node<K,V> hiHead = null, hiTail = null;
                    HashMap.Node<K,V> next;
                    do {
                        next = e.next;
                        // 区分高低链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
~~~
