---
title: ConcurrentHashMap详解
date: 2021-07-30 11:12:43
tags:
    - 并发
categories:
    - 并发
---
# ConcurrentHashMap
&nbsp;&nbsp;&nbsp;&nbsp;安全、并发场景集合操作

## 数据结构
| 版本 | 结构 | 实现 |
| :-----| :---- |:---- |
|1.7|分段数组+数组+链表|cas、分段锁（Reentrantlock）|
|1.8+|数组+链表+红黑树|cas、synchronized|

## ConcurrentHashMap jdk1.7解析
```java
ConcurrentHashMap concurrentHashMap = new ConcurrentHashMap();
concurrentHashMap.put(1, 1);
```
在真正去了解的时候我们先看下ConcurrentHashMap有什么特殊参数
```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {
/**
     * The default initial capacity for this table,
     * used when not otherwise specified in a constructor.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 16; // 默认初始化容量

    /**
     * The default load factor for this table, used when not
     * otherwise specified in a constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f; // 默认加载因子

    /**
     * The default concurrency level for this table, used when not
     * otherwise specified in a constructor.
     */
    static final int DEFAULT_CONCURRENCY_LEVEL = 16; // 默认并行度等级，这个是控制冲突量，影响Segment数组大小，Segment数组一旦确定不会更改

    /**
     * The maximum capacity, used if a higher value is implicitly
     * specified by either of the constructors with arguments.  MUST
     * be a power of two <= 1<<30 to ensure that entries are indexable
     * using ints.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30; // 最大HashEntry数组长度大小，必须是2^n的数

    /**
     * The minimum capacity for per-segment tables.  Must be a power
     * of two, at least two to avoid immediate resizing on next use
     * after lazy construction.
     */
    static final int MIN_SEGMENT_TABLE_CAPACITY = 2; // 最小HashEntry数组长度大小，必须是2^n的数

    /**
     * The maximum number of segments to allow; used to bound
     * constructor arguments. Must be power of two less than 1 << 24.
     */
    static final int MAX_SEGMENTS = 1 << 16; // slightly conservative 最大Segment数组长度，也是并行度最大

    /**
     * Number of unsynchronized retries in size and containsValue
     * methods before resorting to locking. This is used to avoid
     * unbounded retries if tables undergo continuous modification
     * which would make it impossible to obtain an accurate result.
     */
    static final int RETRIES_BEFORE_LOCK = 2; // 在加锁之前尝试次数（无锁尝试）
    /**
     * Mask value for indexing into segments. The upper bits of a
     * key's hash code are used to choose the segment.
     */
    final int segmentMask; // segment数组掩码

    /**
     * Shift value for indexing within segments.
     */
    final int segmentShift; // segment数组迁移数

    /**
     * The segments, each of which is a specialized hash table.
     */
    final Segment<K,V>[] segments; // segment数组

    /**
     * Segments are specialized versions of hash tables.  This
     * subclasses from ReentrantLock opportunistically, just to
     * simplify some locking and avoid separate construction.
     */
    static final class Segment<K,V> extends ReentrantLock implements Serializable { // 注意，这里继承ReentrantLock可重入锁，那就是Segment也会有ReentrantLock的特性
        /*
         * Segments maintain a table of entry lists that are always
         * kept in a consistent state, so can be read (via volatile
         * reads of segments and tables) without locking.  This
         * requires replicating nodes when necessary during table
         * resizing, so the old lists can be traversed by readers
         * still using old version of table.
         *
         * This class defines only mutative methods requiring locking.
         * Except as noted, the methods of this class perform the
         * per-segment versions of ConcurrentHashMap methods.  (Other
         * methods are integrated directly into ConcurrentHashMap
         * methods.) These mutative methods use a form of controlled
         * spinning on contention via methods scanAndLock and
         * scanAndLockForPut. These intersperse tryLocks with
         * traversals to locate nodes.  The main benefit is to absorb
         * cache misses (which are very common for hash tables) while
         * obtaining locks so that traversal is faster once
         * acquired. We do not actually use the found nodes since they
         * must be re-acquired under lock anyway to ensure sequential
         * consistency of updates (and in any case may be undetectably
         * stale), but they will normally be much faster to re-locate.
         * Also, scanAndLockForPut speculatively creates a fresh node
         * to use in put if no node is found.
         */

        private static final long serialVersionUID = 2249069246763182397L;

        /**
         * The maximum number of times to tryLock in a prescan before
         * possibly blocking on acquire in preparation for a locked
         * segment operation. On multiprocessors, using a bounded
         * number of retries maintains cache acquired while locating
         * nodes.
         */
        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1; // 最大扫描尝试次数

        /**
         * The per-segment table. Elements are accessed via
         * entryAt/setEntryAt providing volatile semantics.
         */
        transient volatile HashEntry<K,V>[] table; // 每个Segment有HashEntry<K,V>[]元素

        /**
         * The number of elements. Accessed only either within locks
         * or among other volatile reads that maintain visibility.
         */
        transient int count;

        /**
         * The total number of mutative operations in this segment.
         * Even though this may overflows 32 bits, it provides
         * sufficient accuracy for stability checks in CHM isEmpty()
         * and size() methods.  Accessed only either within locks or
         * among other volatile reads that maintain visibility.
         */
        transient int modCount;

        /**
         * The table is rehashed when its size exceeds this threshold.
         * (The value of this field is always <tt>(int)(capacity *
         * loadFactor)</tt>.)
         */
        transient int threshold; // HashTable的阈值
    }
}
```
<!-- more -->
![ConcurrentHashMap结构图](/images/pasted-88.png)
### 构造函数
```java
/**
    * Creates a new, empty map with the specified initial
    * capacity, load factor and concurrency level.
    *
    * @param initialCapacity the initial capacity. The implementation
    * performs internal sizing to accommodate this many elements. 初始化容量
    * @param loadFactor  the load factor threshold, used to control resizing.
    * Resizing may be performed when the average number of elements per
    * bin exceeds this threshold. 阈值
    * @param concurrencyLevel the estimated number of concurrently
    * updating threads. The implementation performs internal sizing
    * to try to accommodate this many threads. 并行等级
    * @throws IllegalArgumentException if the initial capacity is
    * negative or the load factor or concurrencyLevel are
    * nonpositive.
    */
@SuppressWarnings("unchecked")
public ConcurrentHashMap(int initialCapacity,
                            float loadFactor, int concurrencyLevel) { // 默认initialCapacity=16，loadFactor=0.75，concurrencyLevel=16
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) { // 注意这是while，我看的时候一直当做for，需要注意下，这个就是找到比concurrencyLevel大的2^n的数，默认的ssize=16，sshift=4
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift; // 28，移动数
    this.segmentMask = ssize - 1; // 15，掩码
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize; // 1
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY; // 2，这里默认是2是有讲究的，2 * 0.75 = 1.5，第一个元素是不会扩容的，第二个元素才会扩容
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                            (HashEntry<K,V>[])new HashEntry[cap]); // 初始化Segment内部变量
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize]; // 初始化Segment数组
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0] 注释，就是初始化segments[0]第一个元素
    this.segments = ss;
}
```

### put操作
自己看下，这是调用ConcurrentHashMap.put，然后里面调用Segment.put操作的
```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null) // key可以为null，value不能为null
        throw new NullPointerException();
    int hash = hash(key); // 取hash值
    int j = (hash >>> segmentShift) & segmentMask; // 自己读下，取hash（32位），无符号右移28位，然后&15，就是取最高位的4位当Segment索引下标
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck  不是getObjectVolatile，这个是取j下标下的Segment，UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)获取当前下标有没有初始化，默认只有数组[0]初始化，其他都没有初始化
            (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j); // 初始化
    return s.put(key, hash, value, false); // 主要是调用Segment里的put
}
```
初始化Segment当前key所属的索引为空
```java
/**
    * Returns the segment for the given index, creating it and
    * recording in segment table (via CAS) if not already present.
    *
    * @param k the index
    * @return the segment
    */
@SuppressWarnings("unchecked")
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset 偏移量
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) { // 注意这个采用的volatile，这个防止并发情况下其他线程初始化了
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype(原型) 默认segment[0]是初始化的，其他都可以用segment[0]来初始化
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck 重复检查
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                    == null) { // 仔细看下，代码很严谨，反复检查
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s)) // cas
                    break;
            }
        }
    }
    return seg;
}
```
Segment.put
```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value); // 尝试获取锁，获取成功返回null，失败的话scanAndLockForPut方法，这里有一点注意，这个是Segment的独占锁
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash; // HashEntry 数组里的索引，也是根据hash算出来的，这里也有一点注意，HashEntry[]和Segment[] 里定位索引的方法是不同的，这是为了防止hash碰撞
        HashEntry<K,V> first = entryAt(tab, index); // 根据index，cas取HashEntry对象
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) { // key存在替换
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else { // e 为null || 走到链表最后一个元素
                if (node != null) // 节点不为空，就直接头插法关联链表first
                    node.setNext(first);
                else // node 为null，采用头插法关联现在链表，注意这个是没有放进到数组元素中的
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY) // 注意这是HashEntry数组判断，扩容，扩容逻辑就是元素个数大于阈值
                    rehash(node); // 扩容
                else
                    setEntryAt(tab, index, node); // 把node放到数组中
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally { // 释放锁
        unlock();
    }
    return oldValue;
}

```
`scanAndLockForPut`这个是未获取锁，我们看下他是想做什么？
```java
/**
    * Scans for a node containing given key while trying to
    * acquire lock, creating and returning one if not found. Upon
    * return, guarantees that lock is held. UNlike in most
    * methods, calls to method equals are not screened: Since
    * traversal speed doesn't matter, we might as well help warm
    * up the associated code and accesses as well.
    *
    * @return a new node if key not found, else null
    */
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash); // 根据hash来获取HashEntry[]中下标的HashEntry
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    while (!tryLock()) { // 再次尝试获取锁，获取失败while内容体
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) { // 未初始化
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null); // 该节点初始化，然后返回出去
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) { // 尝试次数超过64，入队列，阻塞，被唤醒后直接返回
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                    (f = entryForHash(this, hash)) != first) { // retries不可能为0，只要retries是偶数(retries & 1) == 0成立，并发情况下，其他线程已经初始化该索引位置元素、或者加入元素了，这里是重新走一遍scanAndLockForPut方法
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}

```
`rehash`扩容
```java
/**
* Doubles size of table and repacks entries, also adding the
* given node to new table
*/
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
    /*
        * Reclassify nodes in each list to new table.  Because we
        * are using power-of-two expansion, the elements from
        * each bin must either stay at same index, or move with a
        * power of two offset. We eliminate unnecessary node
        * creation by catching cases where old nodes can be
        * reused because their next fields won't change.
        * Statistically, at the default threshold, only about
        * one-sixth of them need cloning when a table
        * doubles. The nodes they replace will be garbage
        * collectable as soon as they are no longer referenced by
        * any reader thread that may be in the midst of
        * concurrently traversing table. Entry accesses use plain
        * array indexing because they are followed by volatile
        * table write.
        */
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1; // 老的长度 * n
    threshold = (int)(newCapacity * loadFactor); 
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1; // 新的掩码
    for (int i = 0; i < oldCapacity ; i++) { // 老的数组
        HashEntry<K,V> e = oldTable[i];
        if (e != null) { // 链表元素不为空
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask; // 根据hash重新计算新的索引
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                        last != null;
                        last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) { // 仔细看下，这个是确定迁移后的尾结点
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun; // 先确定尾结点
                // Clone remaining nodes
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) { // 循环链表
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node; // 将当前节点以头插法插入到链表中
    table = newTable;
}
```
分析完上面的代码会遇到下面这种情况，<font color='red'><b>导致节点顺序不固定，但是由于rehash是在线程获取到锁后执行，不会有线程问题</b></font>
场景一
![rehash](/images/pasted-89.png)
场景二
![rehash续](/images/pasted-90.png)

### size操作
```java

/**
* Returns the number of key-value mappings in this map.  If the
* map contains more than <tt>Integer.MAX_VALUE</tt> elements, returns
* <tt>Integer.MAX_VALUE</tt>.
*
* @return the number of key-value mappings in this map
*/
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry 注意这个注释，第一次迭代不是重试
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) { // RETRIES_BEFORE_LOCK == 2 if条件满足retries=3，先判断==再判断++，先尝试了无锁状态三次，三次都没有成功，再加锁。但是第一次不是重试，所以尝试了2次
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation 这个时候无法进行put
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0) // size = size + c
                        overflow = true;
                }
            }
            if (sum == last) 
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) { // 调用lock()方法，retries=3
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```
> <font color='red'><b>这里要注意一下，并发场景下size会尝试获取独占锁</b></font>

### get操作
```java
/**
    * Returns the value to which the specified key is mapped,
    * or {@code null} if this map contains no mapping for the key.
    *
    * <p>More formally, if this map contains a mapping from a key
    * {@code k} to a value {@code v} such that {@code key.equals(k)},
    * then this method returns {@code v}; otherwise it returns
    * {@code null}.  (There can be at most one such mapping.)
    *
    * @throws NullPointerException if the specified key is null
    */
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
> 发现整个get过程中使用了大量的volatile关键字,其实就是保证了可见性、有序性，所以我们只需要保证读取的是最新的数据即可.

### 总结
jdk 1.7 ConcurrentHashMap保证线程安全（注意Volatile读取的是内存中的最新的值，这个是弱一致性，最终一致性）
1. getObjectVolatile
2. compareAndSwapObject
3. putOrderedObject
4. ReentrantLock

> <font color='red'><b>并发场景下put和size操作是会相互影响的，都在尝试获取独占锁，性能还是会影响</b></font>

## ConcurrentHashMap jdk1.8解析
前言，jdk1.8版本的ConcurrentHashMap融合了很多其他特性，采用了HashMap1.8的红黑树特性，也有ConcurrentHashMap并发特性。需要仔细看下，而且代码难度还是非常高，建议静心去看
```java
ConcurrentHashMap concurrentHashMap = new ConcurrentHashMap();
concurrentHashMap.put(1, 1);
```
在真正去了解的时候我们先看下ConcurrentHashMap有什么特殊参数
```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    /**
     * The largest possible table capacity.  This value must be
     * exactly 1<<30 to stay within Java array allocation and indexing
     * bounds for power of two table sizes, and is further required
     * because the top two bits of 32bit hash fields are used for
     * control purposes.
     */
    private static final int MAXIMUM_CAPACITY = 1 << 30; // 数组最大容量，但是数组容量一定是2^n次幂的数

    /**
     * The default initial table capacity.  Must be a power of 2
     * (i.e., at least 1) and at most MAXIMUM_CAPACITY.
     */
    private static final int DEFAULT_CAPACITY = 16; // 默认数组大小

    /**
     * The largest possible (non-power of two) array size.
     * Needed by toArray and related methods.
     */
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; // toArray最大数组大小

    /**
     * The default concurrency level for this table. Unused but
     * defined for compatibility with previous versions of this class.
     */
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16; // 默认并行度16

    /**
     * The load factor for this table. Overrides of this value in
     * constructors affect only the initial table capacity.  The
     * actual floating point value isn't normally used -- it is
     * simpler to use expressions such as {@code n - (n >>> 2)} for
     * the associated resizing threshold.
     */
    private static final float LOAD_FACTOR = 0.75f; // 默认加载因子

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2, and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8; // 链表转红黑树阈值

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6; // 红黑树退化链表阈值

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * The value should be at least 4 * TREEIFY_THRESHOLD to avoid
     * conflicts between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64; // 最小转红黑树数组长度

    /**
     * Minimum number of rebinnings per transfer step. Ranges are
     * subdivided to allow multiple resizer threads.  This value
     * serves as a lower bound to avoid resizers encountering
     * excessive memory contention.  The value should be at least
     * DEFAULT_CAPACITY.
     */
    private static final int MIN_TRANSFER_STRIDE = 16; // 最小转移数据步长

    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16; // 扩容中使用的戳

    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1; // 最大的帮助迁移数据线程数

    /**
     * The bit shift for recording size stamp in sizeCtl.
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS; // 扩容中使用的迁移位

    /*
     * Encodings for Node hash fields. See above for explanation.
     */
    static final int MOVED     = -1; // hash for forwarding nodes 状态在迁移数据中
    static final int TREEBIN   = -2; // hash for roots of trees   状态在树中
    static final int RESERVED  = -3; // hash for transient reservations 
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

    /** Number of CPUS, to place bounds on some sizings */
    static final int NCPU = Runtime.getRuntime().availableProcessors(); // 逻辑核

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
    }
    /**
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table; // 数组

    /**
     * The next table to use; non-null only while resizing.
     */
    private transient volatile Node<K,V>[] nextTable; // 扩容中扩容数组

    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     */
    private transient volatile long baseCount; // 基础数量，这个是计算size()使用

    /**
     * Table initialization and resizing control.  When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads).  Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     */
    private transient volatile int sizeCtl; // 这个变量有非常大的作用的

    /**
     * The next table index (plus one) to split while resizing.
     */
    private transient volatile int transferIndex; // 扩容中转移的下标

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
    private transient volatile int cellsBusy; // 计算单元的信号

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells; // size里帮组计算的计算单元

    /**
     * Nodes for use in TreeBins
     */
    static final class TreeNode<K,V> extends Node<K,V> { // TreeNode
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        // ...
    }

    /**
     * TreeNodes used at the heads of bins. TreeBins do not hold user
     * keys or values, but instead point to list of TreeNodes and
     * their root. They also maintain a parasitic read-write lock
     * forcing writers (who hold bin lock) to wait for readers (who do
     * not) to complete before tree restructuring operations.
     */
    static final class TreeBin<K,V> extends Node<K,V> { // 树本
        TreeNode<K,V> root; // 跟节点
        volatile TreeNode<K,V> first; // 第一个节点
        volatile Thread waiter; 
        volatile int lockState; // 锁状态
        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock

        /**
         * Creates bin with initial set of nodes headed by b.
         */
        TreeBin(TreeNode<K,V> b) {
            super(TREEBIN, null, null, null); // hash值是TREEBIN（-2）
            this.first = b;
            //... 省略代码
        }

        // ...
    }
    /**
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> { // 转移节点
        final Node<K,V>[] nextTable; // 下一次数组
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null); // hash为MOVED（-1）
            this.nextTable = tab;
        }
        // ...
    }

    /**
     * A place-holder node used in computeIfAbsent and compute
     */
    static final class ReservationNode<K,V> extends Node<K,V> {// 翻转节点

        ReservationNode() {
            super(RESERVED, null, null, null); // hash为RESERVED（-3）
        }
        // ...
    }

}

```
### 构造函数
```java
public ConcurrentHashMap() { // 默认情况下，没有内容，发现相比较其他版本是比较少
}
// initialCapacity=16,loadFactor=0.75f,concurrencyLevel=16得到的cap=32，这里和之前是不一样的，但是还是得到2^n的数
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) { // 最全参数，这里可以借鉴下1.7CoucurrentHashMap
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads 16
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap; // sizeCtl=32 (初始化容量)
}
```

### put操作
看一下源码，我们大致已经知道其过程，那比较复杂就是`helpTransfer`、`treeifyBin`、`addCount`，这些还是先看下`treeifyBin`，然后再看一下`addCount`和`helpTransfer`（为什么这么看？一脸懵）
```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException(); // key、value不为null
    int hash = spread(key.hashCode()); // 取hash值
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0) 
            tab = initTable(); // 数组为null，初始化数组
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // index 还是 (n - 1) & hash
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED) // 这个是标志位，hash为MOVED，表示在迁移中
            tab = helpTransfer(tab, f); // 看字面意思是帮助转移，具体再看
        else {
            V oldVal = null;
            synchronized (f) { // synchronized 获取对象锁，一个线程put只能操作这个索引位置的节点
                if (tabAt(tab, i) == f) { // 双重检查，再次判断，是否节点以及修改
                    if (fh >= 0) { // 链表，这个判断很尴尬，看内容体是表示链表，但是有个问题是链表的hash都大于等于0（这个也是对的，可以看下spread方法），但是也可能是红黑树节点，那这里先打个预防针，除了链表fh都是<0，可以持续看后续节点
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) { // 循环链表
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
                            if ((e = e.next) == null) { // 尾插法
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 这个表示红黑树，这个和1.8HashMap TreeNode是不一样的，这个是TreeBin
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) { // putTreeVal
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) { // 默认是0，当索引位置有元素就不可能为0
                if (binCount >= TREEIFY_THRESHOLD) 
                    treeifyBin(tab, i); // 树化
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount); // 增加节点，看到这里发现有扩容的代码，那这里的代码addCount也包括了扩容的逻辑
    return null;
}
```
#### spread方法
这个方法就是取hash值
```java
/**
    * Spreads (XORs) higher bits of hash to lower and also forces top
    * bit to 0. Because the table uses power-of-two masking, sets of
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
    */
static final int spread(int h) { // 取hash值，虽然和1.8hashmap大致相同，同样是高低16位异或，但是多了个与HASH_BITS，HASH_BITS是0x7fffffff;，保持最高位是0，就是符号位是0，是正数，这也应对了后续的判断条件
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
#### initTable
初始化表格方法，这里是有并发情况，但是只能一个线程初始化能成功，那我们看下如果线程安全的初始化，这里要标注下
sizeCtl就是阈值
```java
/**
    * Initializes table, using the size recorded in sizeCtl.
    */
private final Node<K,V>[] initTable() { // 初始化表格，使用sizeCtl方法
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) { 
        if ((sc = sizeCtl) < 0)// sizeCtl默认是0，根据下面的代码，并发情况下只有一个线程将sizeCtl改为-1，其他都会来到这里
            Thread.yield(); // lost initialization race; just spin // 让出cpu使用权
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 设置为-1
            try {
                if ((tab = table) == null || tab.length == 0) { // 再次验证
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY; // DEFAULT_CAPACITY=16
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2); // sc = 3/4 * n，就是阈值
                }
            } finally {
                sizeCtl = sc; // sizeCtl阈值
            }
            break;
        }
    }
    return tab;
}
```
#### treeifyBin
树化，<font color='red'><b>通过synchronized锁住b，然后去树化，不像1.7ConcurrentHashMap锁住Segment来树化，发现粒度更小，树化后直接把TreeBin放到的数组下标，这里TreeBin内部如何调整，只要TreeBin没有变化，可以干更多的事情</b></font>
```java
/**
    * Replaces all linked nodes in bin at given index unless table is
    * too small, in which case resizes instead.
    */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) // MIN_TREEIFY_CAPACITY=64，数组长度，ConcurrentHashMap1.7也是如此，扩容
            tryPresize(n << 1); // 2 * n
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) { // 树化，tabAt是cas操作，取出数组索引下的第一个节点
            synchronized (b) {
                if (tabAt(tab, index) == b) { // dcl，双重验证，有可能其他线程修改第一个元素
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) { // 修改成双向链表，和1.7是一样的逻辑
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                                null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd)); // 这才是正在树化的过程，然后把TreeBin放在数组下标
                }
            }
        }
    }
}
```

```java
/*
    * Volatile access methods are used for table elements as well as
    * elements of in-progress next table while resizing.  All uses of
    * the tab arguments must be null checked by callers.  All callers
    * also paranoically precheck that tab's length is not zero (or an
    * equivalent check), thus ensuring that any index argument taking
    * the form of a hash value anded with (length - 1) is a valid
    * index.  Note that, to be correct wrt arbitrary concurrency
    * errors by users, these checks must operate on local variables,
    * which accounts for some odd-looking inline assignments below.
    * Note that calls to setTabAt always occur within locked regions,
    * and so in principle require only release ordering, not
    * full volatile semantics, but are currently coded as volatile
    * writes to be conservative.
    */

@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) { // cas 操作
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) { // cas操作
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) { // cas操作
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```
##### tryPresize
扩容，随便者迁移数据
```java
/**
    * Tries to presize table to accommodate the given number of elements.
    *
    * @param size number of elements (doesn't need to be perfectly accurate)
    */
private final void tryPresize(int size) { // size是 2* n，默认的话size就是32
        // size c
        // 1 2 ====>1
        // 2 4 ====>10
        // 3 8
        // 4 8
        // 5 8 ====>101
        // 6 16
        // 8 16
        // 10 16
        // 21 32 ====>10101(c是size最接近2^n最大的数，不会翻倍)
        // 22 64
        // 85 128 ====>1010101
        // j      ====>101010101
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1); // tableSizeFor 就是找出比传入值大于等于2^n次幂的数，如果是默认的话，c=64
    int sc;
    while ((sc = sizeCtl) >= 0) { // sc 这个时候是阈值
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) { // putAll操作，就会走到这里，这里的n就是16，老数组的长度
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) { 
            int rs = resizeStamp(n); // 这个获取一个扩容的戳，n一样，返回的值是一样的，具体实现可以不用管，rs是正数
            if (sc < 0) { // TODO 这里什么时候的sc < 0?
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0) // 这个条件需要好好看下
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) // 这里会把sc+1
                    transfer(tab, nt); // 转移，nt是新的数组
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2)) // sizeCtl = (rs << RESIZE_STAMP_SHIFT) + 2 这个必须是负数
                transfer(tab, null); // 转移
        }
    }
}
```
```java
/**
    * Returns the stamp bits for resizing a table of size n.
    * Must be negative when shifted left by RESIZE_STAMP_SHIFT.
    */
static final int resizeStamp(int n) { // 时间戳，看下注释，很清晰，这里这个方法没有必要去深入
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
```java
/**
    * Moves and/or copies the nodes in each bin to new table. See
    * above for explanation.
    */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) { // 转移
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range stride步长，固定取16
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1]; // 2 * n = 32
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true; // advance 表示推进
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) { // 
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

```
##### TreeBin
真正树化，下面的代码已经很熟悉了，1.8HashMap 已经深入介绍了，不多讲，这里着重说一下这里的hash值是TREEBIN，
```java
/**
    * Creates bin with initial set of nodes headed by b.
    */
TreeBin(TreeNode<K,V> b) {
    super(TREEBIN, null, null, null); // hash为TREEBIN，就是在这里操作的
    this.first = b; // 第一个节点就是b
    TreeNode<K,V> r = null;
    for (TreeNode<K,V> x = b, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (r == null) {
            x.parent = null;
            x.red = false;
            r = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = r;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    r = balanceInsertion(r, x);
                    break;
                }
            }
        }
    }
    this.root = r;
    assert checkInvariants(root);
}
```