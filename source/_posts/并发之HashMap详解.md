---
title: 并发之HashMap详解
date: 2021-07-19 11:19:37
tags:
    - 并发
categories:
    - 并发
---
# HashMap

## 数据结构
| 版本 | 结构 |
| :-----| :---- |
|1.7|数组+链表|
|1.8+|数组+链表+红黑树|

## HashMap jdk1.7解析
这里通过下面的例子，来介绍HashMap1.7源码
```java
HashMap<String,String> map = new HashMap(11);
map.put("1","1");
```
### 构造函数
我们看下初始化做了什么事情？见下面代码。初始化容量、加载因子，那这里有个疑问？传进来的初始化容量是11，那数组的容量就是11？这里回答下，数组的长度不是11，那是多少呢？我们接下来看
```java
/** 
*  构造函数
*  @param initialCapacity 初始化容量
*  @param loadFactor 加载因子0.75
**/
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
    threshold = initialCapacity;
    init();
}
```
<!-- more -->

其他属性也需要稍微介绍下
```java
/**
* The default initial capacity - MUST be a power of two.
*/
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
* The maximum capacity, used if a higher value is implicitly specified
* by either of the constructors with arguments.
* MUST be a power of two <= 1<<30.
*/
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
* The load factor used when none specified in constructor.
* 这里要说明下，为什么加载因子是0.75，是在时间和空间上面的一个均衡
* As a general rule, the default load factor (.75) offers a good tradeoff
* between time and space costs
*/
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
* The next size value at which to resize (capacity * load factor).
* @serial
*/
// If table == EMPTY_TABLE then this is the initial capacity at which the
// table will be created when inflated.
int threshold; // 阈值，capacity * load factor
/**
* The load factor for the hash table.
*
* @serial
*/
final float loadFactor;
```

### put操作
```java
/**
* put操作
*
**/
public V put(K key, V value) {
    // 初始化数组
    if (table == EMPTY_TABLE) {
        inflateTable(threshold); // threshold是11
    }
    // HashMap可以接受key为空
    if (key == null)
        return putForNullKey(value);
    // 得到hash值
    int hash = hash(key);
    // 根据hash值得到索引
    int i = indexFor(hash, table.length);
    // 下面是和putForNullKey很像
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
inflateTable方法，这里数组的长度是大于等于size的二的n次幂的数，这里传进来的是11，那么capacity是16，这里第一个问题已经解答。但是还有个问题，为什么数组长度必须是二的n次幂的数？那么我们接下来看
```java
private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize
    int capacity = roundUpToPowerOf2(toSize);
    // 阈值 12
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity);
}
```
putForNullKey方法，如果key为null，则直接取数组的第一个，如果key存在，则覆盖更新，返回旧值，否则不存在的话就加入到链表中
```java
 private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    // 加入到链表中
    addEntry(0, null, value, 0);
    return null;
}
```
hash方法，这里都是位的运算，目的是将hash更散列
```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
indexFor方法，这里是根据hash值得到索引下标，这里可以看下，`h & (length-1)`，如果length是16，则返回的结果就是hash的低4位值，其实这里就是取16的余数，等价于`h % length`，但是`%`取余是非常慢的，`&`更加接近二进制，效率会更快。这里还有一点，为什么是2的n次幂，如果不是的话，通过`&`操作，如果length是17，那得到的结果不是16，就是0，散列不均匀。这个就是第二个问题的答案。
```java
/**
* Returns index for hash code h.
*/
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```
![hashmap索引](/images/pasted-73.png)
![hashmap索引-续](/images/pasted-74.png)

addEntry方法是加入到链表里
```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // 数组元素大于12，扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length); // 数组长度翻倍
        hash = (null != key) ? hash(key) : 0;
        // 重新计算索引
        bucketIndex = indexFor(hash, table.length);
    }
    // 真正将元素放到链表中
    createEntry(hash, key, value, bucketIndex);
}
```
resize扩容，数组也需要迁移，index有可能是不一样的
```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    // 阈值，现在为32 * 0.75 = 24
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
transfer传输元素，逻辑很简单，就是将老的元素迁移到新的数组里面，但是这个迁移的时候是采用头插法的方式
```java
/**
    * Transfers all entries from current table to newTable.
    */
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```
![hashmap头插法](/images/pasted-71.png)

createEntry创建entry，将元素放到数组里面，`new Entry<>(hash, key, value, e)`这里也是头插法
```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e); 
    size++;
}
```

### 引申：1.7HashMap为什么会在put的时候存在并发安全问题？
![hashmap链表环](/images/pasted-72.png)


## HashMap jdk1.8解析
由于HashMap1.8修改了迁移的算法和结构，并发情况下，并没有出现链表环
```java
HashMap<String,String> map = new HashMap(11);
map.put("1","1");
```
### 构造函数
构造函数其他都一样，多了一个`tableSizeFor`
```java
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
```
tableSizeFor方法，其实突然看一下挺复杂的，但是根据注释，还是得到大于size的最小2的n次幂，只是调整了下算法
```java
/**
    * Returns a power of two size for the given target capacity.
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
```
![算法](/images/pasted-75.png)
除了在jdk1.7下的属性，由于HashMap在1.8基础上增加了红黑树，关于红黑树的相关属性也介绍下
```java
/**
* The bin count threshold for using a tree rather than list for a
* bin.  Bins are converted to trees when adding an element to a
* bin with at least this many nodes. The value must be greater
* than 2 and should be at least 8 to mesh with assumptions in
* tree removal about conversion back to plain bins upon
* shrinkage.
*/
static final int TREEIFY_THRESHOLD = 8; //链表长度转红黑树阈值
/**
* The bin count threshold for untreeifying a (split) bin during a
* resize operation. Should be less than TREEIFY_THRESHOLD, and at
* most 6 to mesh with shrinkage detection under removal.
*/
static final int UNTREEIFY_THRESHOLD = 6; // 红黑树退化为链表阈值
/**
* The smallest table capacity for which bins may be treeified.
* (Otherwise the table is resized if too many nodes in a bin.)
* Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
* between resizing and treeification thresholds.
*/
static final int MIN_TREEIFY_CAPACITY = 64; // 链表转红黑树，数组长度


/**
* Basic hash bin node, used for most entries.  (See below for
* TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
*/
static class Node<K,V> implements Map.Entry<K,V> { // 节点结构也是调整了，现在是采用Node节点
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
/**
    * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
    * extends Node) so can be used as extension of either regular or
    * linked node.
    */
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> { // 红黑树节点类型，其间接继承Node
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
}
```
### put操作
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

#### hash方法
这里的hash方法就不太一样了，这里有一定的规则，如果key是空的，就返回0，否则就是key的hashCode值，高16位和低16位异或。
```java
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
*/
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
发现hash方法是有调整的，这里的调整的目的是这样的，由于取index是`(n - 1) & hash`，下面这个例子如果n=16。只要低4位相同，其他位如果不同，也不会有什么影响，还是会bucket冲突，那这么设计是为什么呢？其实注释也说明了下，设计者们在速度，实用性和位扩展质量之间进行权衡后应用了一种变换，通过将 hashcode 的低 16 位与高 16 位异或向下传播较高位的影响。
>
> n = 16
> index = (n - 1) & hash
> Hash：0000 0001 0000 1111
> Hash：0000 0010 1100 1111
> index：0000 0000 0000 1111

#### putVal方法
这里最核心put的方法，这个方法涉及链表、红黑树
1. <font color='red'><b>index相同的节点hash是有可能是相同的</b></font>
2. <font color='red'><b>两个节点相同条件是hash、key相同，就表示相同的节点</b></font>
```java
/**
* Implements Map.put and related methods.
*
* @param hash hash for key
* @param key the key
* @param value the value to put
* @param onlyIfAbsent if true, don't change existing value 如果为true，存在相同的值，不会被覆盖
* @param evict if false, the table is in creation mode. 如果为false，则表在创建中
* @return previous value, or null if none
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) // 数组是空的，初始化数组
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) // 找到的index如果没有节点，插入该节点，注意这里是采用后插法
        tab[i] = newNode(hash, key, value, null);
    else { // 这里表示已经找到index，然后准备插入到链表或者红黑树，其中p是index的第一个节点，k表示index的第一个节点的key
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) // 正好该节点和第一个节点hash相同，key也是相同的，就覆盖了
            e = p;
        else if (p instanceof TreeNode) // 如果该节点是Tree类型，表示是红黑树
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); // 将节点放入到红黑树中
        else { // 这个就是链表
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null); // 该节点加入到链表中了
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st ， TREEIFY_THRESHOLD为8，那binCount为7的时候就树化了，注意binCount是从0开始的，那等于说链表节点有8个元素，注意当前节点也加入到链表中，所以链表长度是9
                        treeifyBin(tab, hash); // 链表转红黑树
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) // 链表节点相同，覆盖
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key , 该节点存在，即更新，返回旧值
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null) // 判断是否覆盖，如果是null值，一定会被覆盖
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict); // 空的，还没有实现
    return null;
}
```
由上面的方法可以发现，数组为空的时候调用了`resize()`，数组元素长度大于阈值就会调用`resize()`，这个方法看起来干了很多事情，接下来我们看一下`resize()`

#### resize方法
```java
/**
* 初始化表格数组、扩容
* Initializes or doubles table size.  If null, allocates in
* accord with initial capacity target held in field threshold.
* Otherwise, because we are using power-of-two expansion, the
* elements from each bin must either stay at same index, or move
* with a power of two offset in the new table.
*
* @return the table
*/
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) { // 迁移数据
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null) // 只存在一个元素
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) // 如果当前数组的根节点就是TreeNode，就表明当前已经是红黑树
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order 链表迁移
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
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
```
![resize流程图](/images/pasted-76.png)
这里再说下链表迁移，链表迁移肯定不会树化，这个跟他的扩容index有关系
![链表迁移](/images/pasted-77.png)

红黑树拆分，代码比较简单，仔细读下，`untreeify`非树化，也就是链表化，这一块是简单的，`treeify`就是树化，比较复杂，下面会详细介绍，这里不过多介绍
```java
/**
    * Splits nodes in a tree bin into lower and upper tree bins,
    * or untreeifies if now too small. Called only from resize;
    * see above discussion about split bits and indices.
    *
    * @param map the map
    * @param tab the table for recording bin heads
    * @param index the index of the table being split
    * @param bit the bit of hash to split on 老的容量
    */
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this; // 当前数组所属的index的第一个节点
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```
非树化
```java
/**
    * Returns a list of non-TreeNodes replacing those linked from
    * this node.
    */
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null); // 将节点转化为Node节点，然后将其链表化
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```
#### treeifyBin树化

##### 红黑树特性
&nbsp;&nbsp;&nbsp;&nbsp;红黑树是HashMap1.8引入的新的结构。红黑树是一种平衡二叉树。红黑树有以下特性
1. 节点是红色或黑色
2. 根节点是黑色
3. 每个叶节点（NIL节点，空节点）是黑色的
4. 每个红色节点的两个子节点都是黑色（从根到每个叶子节点的所有路径上不能有两个连续的红色节点）
5. 从任一节点到该节点的每个叶子节点的所有路径都包含相同数目的黑色节点
> [红黑树：https://www.cs.usfca.edu/~galles/visualization/RedBlack.html](https://www.cs.usfca.edu/~galles/visualization/RedBlack.html)

##### 红黑树演化
![场景](/images/pasted-78.png)
![场景续](/images/pasted-79.png)
根据上面的演化，我们可以得到下面的规则
1. <font color='red'><b>如果插入的父节点是黑色，可以不用动</b></font>
2. <font color='red'><b>如果插入的父节点是红色，叔叔节点是空的，左旋+变色</b></font>
3. <font color='red'><b>如果插入的父节点是红色，叔叔节点是红色，父节点和叔叔节点都变色</b></font>
4. <font color='red'><b>如果插入的父节点是红色，叔叔节点是黑的，变色+左旋+变色</b></font>

> 上面的场景已经覆盖到全部，然后跟着代码走一走

##### 源码介绍
这是树化的入口
```java
/**
    * Replaces all linked nodes in bin at index for given hash unless
    * table is too small, in which case resizes instead.
    */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null); // 创建TreeNode，为树化做准备
            if (tl == null)
                hd = p;
            else {
                p.prev = tl; // 变成双向链表
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab); // 真正树化
    }
}
```
```java
// For treeifyBin
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) { // 当前节点转换成树节点
    return new TreeNode<>(p.hash, p.key, p.value, next); 
}
```
```java
/**
* 树化的核心步骤
* Forms tree of the nodes linked from this node.
*/
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) { // 如果是空的
            x.parent = null;
            x.red = false; // 根节点是黑色的
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h) // 判断hash值大小，然后判断当前节点放入根节点的左边还是右边
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {  // 把元素插进红黑树里面，形成平衡二叉树
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x); // 插入成功后，然后准备平衡红黑树，旋转和着色
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root); //将root移到最前面，然后将root放到数组上
}

```
```java
/**
* 节点已经插入到红黑树中，平衡红黑树
* root 跟节点
* x    插入节点
**/
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                TreeNode<K,V> x) {
    x.red = true; // 插入的节点都是红色
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        if ((xp = x.parent) == null) { // 1. xp 是 x的父节点，根节点
            x.red = false;
            return x;
        }
        else if (!xp.red || (xpp = xp.parent) == null) // 2. xp 是黑色节点，xpp 是 x 爷爷节点
            return root;
        if (xp == (xppl = xpp.left)) { // 3. 父节点是爷爷的左孩子
            if ((xppr = xpp.right) != null && xppr.red) { // 3.1 
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else { // 3.2 
                if (x == xp.right) { // 3.2.1
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) { // 3.2.2
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else { // 4. 父节点是爷爷的右孩子
            if (xppl != null && xppl.red) { // 4.1 
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else { //4.2 
                if (x == xp.left) { // 4.2.1
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) { //4.2.2
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
/**
* 左旋
**/
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        if ((rl = p.right = r.left) != null)
            rl.parent = p;
        if ((pp = r.parent = p.parent) == null)
            (root = r).red = false;
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;
        r.left = p;
        p.parent = r;
    }
    return root;
}
/**
* 右旋
**/
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                        TreeNode<K,V> p) {
    TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) {
        if ((lr = p.left = l.right) != null)
            lr.parent = p;
        if ((pp = l.parent = p.parent) == null)
            (root = l).red = false;
        else if (pp.right == p)
            pp.right = l;
        else
            pp.left = l;
        l.right = p;
        p.parent = l;
    }
    return root;
}
```
> 左旋和右旋中大量使用了lr = p.left = l.right这种三元表达式，那这个表达式是表示什么呢？表示
> lr = l.right
> p.left = l.right

1. xp = x.parent，根节点
2. 父节点是黑色，xpp = xp.parent
![场景一](/images/pasted-80.png)
3. 父节点是爷爷的左孩子，xppl = xpp.left
![场景二](/images/pasted-81.png)
    3.1 叔叔节点是红色的，xppr = xpp.right，是上图的第三、四种场景(第四种比较特殊)，变色即可，通过第二次循环即可解决
    ![场景二-1](/images/pasted-83.png)
    3.2 叔叔节点是null或者黑色的，第一、二场景就是叔叔节点是null，那叔叔节点是黑色的是什么场景？是第四种场景中间变换的节点状态。这个如果当前节点是右节点，先左旋，在考虑右旋
    ![场景二-2](/images/pasted-84.png)
4. 父节点是爷爷的右孩子
![场景三](/images/pasted-82.png)
    4.1 叔叔节点是红色的，是上图的第三、四种场景(第四种比较特殊)，变色即可，通过第二次循环即可解决
    ![场景二-1](/images/pasted-85.png)
    4.2 叔叔节点是null或者黑色的，第一、二场景就是叔叔节点是null，那叔叔节点是黑色的是什么场景？是第四种场景中间变换的节点状态。这个如果当前节点是左节点，先右旋，在考虑左旋
    ![场景二-2](/images/pasted-86.png)

> 这里没有重点讲左旋、右旋，可以对着上面的场景，然后自己走一遍

#### 移动root节点
数组的第一个节点就是root节点
```java
/**
    * Ensures that the given root is the first node of its bin.
    */
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        if (root != first) {
            Node<K,V> rn;
            tab[index] = root;
            TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
                rp.next = rn;
            if (first != null)
                first.prev = root;
            root.next = first;
            root.prev = null;
        }
        assert checkInvariants(root);
    }
}
```
![移动root置前](/images/pasted-87.png)

#### putTreeVal
这个就是把节点放入到红黑树中，还是比较简单的
```java
/**
    * Tree version of putVal.
    */
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;  // 寻找root节点，this就是当前数组索引下标的TreeNode节点，这里有可能其他线程会修改root
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if ((ph = p.hash) > h) // 判断当前节点放在红黑树哪一边
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if ((kc == null &&
                    (kc = comparableClassFor(k)) == null) ||
                    (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                        (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            moveRootToFront(tab, balanceInsertion(root, x)); // 将节点放进红黑树后平衡红黑树，然后将新的root移到最前
            return null;
        }
    }
}
```
```java
/**
    * Returns root of tree containing this node.
    */
final TreeNode<K,V> root() {
    for (TreeNode<K,V> r = this, p;;) {
        if ((p = r.parent) == null)
            return r;
        r = p;
    }
}
```

<p>
```java
/**
 * 还是可以输出，只要不启用断言
 * hashmap assert 断言
 * -ea 启用断言
 * -da 关闭断言
 * 默认是不启用的
 */
public class HashMapAssertTest {

    public static void main(String[] args) {
        assert 1==2;
        System.out.println(1);
    }
}
```
</p>

### get操作
获取操作，就是遍历链表或者红黑树，就不仔细讲了

### delete操作
这个也比较复杂，后续有时间可以补充上来

### 引申：链表转红黑树是条件是节点>=8？
下面条件才会转红黑树
1. <font color='red'><b>数组长度>=64</b></font>
2. <font color='red'><b>链表元素为9个的时候</b></font>