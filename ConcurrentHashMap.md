# ConcurrentHashMap

## 1.简介

HashMap是我们用得非常频繁的一个集合，但是由于它是非线程安全的，在多线程环境下，put操作是有可能产生死循环的，导致CPU利用率接近100%。为了解决该问题，提供了Hashtable和Collections.synchronizedMap(hashMap)两种解决方案，但是这两种方案都是对读写加锁，独占式，一个线程在读时其他线程必须等待，吞吐量较低，性能较为低下。故而Doug Lea大神给我们提供了高性能的线程安全HashMap：ConcurrentHashMap。

ConcurrentHashMap作为Concurrent一族，其有着高效地并发操作，相比Hashtable的笨重，ConcurrentHashMap则更胜一筹了。

在1.8版本以前，ConcurrentHashMap采用分段锁的概念，使锁更加细化，但是1.8已经改变了这种思路，而是利用CAS+Synchronized来保证并发更新的安全，当然底层采用数组+链表+红黑树的存储结构。

关于1.7和1.8的区别请参考占小狼博客：谈谈ConcurrentHashMap1.7和1.8的不同实现:<http://www.jianshu.com/p/e694f1e868ec>

## 2.属性

### 2.1.静态变量

ConcurrentHashMap定义了如下几个常量：


```java
// 最大容量：2^30=1073741824
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认初始值，必须是2的幕数
private static final int DEFAULT_CAPACITY = 16;

//
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

//
private static final float LOAD_FACTOR = 0.75f;

// 链表转红黑树阀值,> 8 链表转换为红黑树
static final int TREEIFY_THRESHOLD = 8;

//树转链表阀值，小于等于6（tranfer时，lc、hc=0两个计数器分别++记录原bin、新binTreeNode数量，<=UNTREEIFY_THRESHOLD 则untreeify(lo)）
static final int UNTREEIFY_THRESHOLD = 6;

//
static final int MIN_TREEIFY_CAPACITY = 64;

//
private static final int MIN_TRANSFER_STRIDE = 16;

//
private static int RESIZE_STAMP_BITS = 16;

// 2^15-1，help resize的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

// 32-16=16，sizeCtl中记录size大小的偏移量
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

// forwarding nodes的hash值
static final int MOVED     = -1;

// 树根节点的hash值
static final int TREEBIN   = -2;

// ReservationNode的hash值
static final int RESERVED  = -3;

// 可用处理器数量
static final int NCPU = Runtime.getRuntime().availableProcessors();
```

* 上面是ConcurrentHashMap定义的常量，简单易懂，就不多阐述了。下面介绍ConcurrentHashMap几个很重要的概念。
* table：用来存放Node节点数据的，默认为null，默认大小为16的数组，每次扩容时大小总是2的幂次方；
* nextTable：扩容时新生成的数据，数组为table的两倍；
* Node：节点，保存key-value的数据结构；
* ForwardingNode：一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动
* sizeCtl：控制标识符，用来控制table初始化和扩容操作的，在不同的地方有不同的用途，其值也不同，所代表的含义也不同
  * 负数代表正在进行初始化或扩容操作
  * -1代表正在初始化
  * -N 表示有N-1个线程正在进行扩容操作
  * 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小

### 2.2.重要内部类

为了实现ConcurrentHashMap，Doug Lea提供了许多内部类来进行辅助实现，如Node，TreeNode,TreeBin等等。下面我们就一起来看看ConcurrentHashMap几个重要的内部类。

#### 2.1.Node


作为ConcurrentHashMap中最核心、最重要的内部类，Node担负着重要角色：key-value键值对。所有插入ConCurrentHashMap的中数据都将会包装在Node中。定义如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;             //带有volatile，保证可见性
    volatile Node<K,V> next;    //下一个节点的指针

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    /** 不允许修改value的值 */
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**  赋值get()方法 */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

在Node内部类中，其属性value、next都是带有volatile的。同时其对value的setter方法进行了特殊处理，不允许直接调用其setter方法来修改value的值。最后Node还提供了find方法来赋值map.get()。

#### 2.2.TreeNode

我们在学习HashMap的时候就知道，HashMap的核心数据结构就是链表。在ConcurrentHashMap中就不一样了，如果链表的数据过长是会转换为红黑树来处理。当它并不是直接转换，而是将这些链表的节点包装成TreeNode放在TreeBin对象中，然后由TreeBin完成红黑树的转换。所以TreeNode也必须是ConcurrentHashMap的一个核心类，其为树节点类，定义如下：

```java
static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }


        Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null);
        }

        //查找hash为h，key为k的节点
        final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
            if (k != null) {
                TreeNode<K,V> p = this;
                do  {
                    int ph, dir; K pk; TreeNode<K,V> q;
                    TreeNode<K,V> pl = p.left, pr = p.right;
                    if ((ph = p.hash) > h)
                        p = pl;
                    else if (ph < h)
                        p = pr;
                    else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                        return p;
                    else if (pl == null)
                        p = pr;
                    else if (pr == null)
                        p = pl;
                    else if ((kc != null ||
                            (kc = comparableClassFor(k)) != null) &&
                            (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.findTreeNode(h, k, kc)) != null)
                        return q;
                    else
                        p = pl;
                } while (p != null);
            }
            return null;
        }
    }
```

#### 2.3.TreeBin

该类并不负责key-value的键值对包装，它用于在链表转换为红黑树时包装TreeNode节点，也就是说ConcurrentHashMap红黑树存放是TreeBin，不是TreeNode。该类封装了一系列的方法，包括putTreeVal、lookRoot、UNlookRoot、remove、balanceInsetion、balanceDeletion。由于TreeBin的代码太长我们这里只展示构造方法（构造方法就是构造红黑树的过程）：

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K, V> root;
    volatile TreeNode<K, V> first;
    volatile Thread waiter;
    volatile int lockState;
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock

    TreeBin(TreeNode<K, V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        TreeNode<K, V> r = null;
        for (TreeNode<K, V> x = b, next; x != null; x = next) {
            next = (TreeNode<K, V>) x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            } else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K, V> p = r; ; ) {
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
                    TreeNode<K, V> xp = p;
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

    /** 省略很多代码 */
}
```

#### 2.4.ForwardingNode

这是一个真正的辅助类，该类仅仅只存活在ConcurrentHashMap扩容操作时。只是一个标志节点，并且指向nextTable，它提供find方法而已。该类也是集成Node节点，其hash为-1，key、value、next均为null。如下：

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
       final Node<K,V>[] nextTable;
       ForwardingNode(Node<K,V>[] tab) {
           super(MOVED, null, null, null);
           this.nextTable = tab;
       }

       Node<K,V> find(int h, Object k) {
           // loop to avoid arbitrarily deep recursion on forwarding nodes
           outer: for (Node<K,V>[] tab = nextTable;;) {
               Node<K,V> e; int n;
               if (k == null || tab == null || (n = tab.length) == 0 ||
                       (e = tabAt(tab, (n - 1) & h)) == null)
                   return null;
               for (;;) {
                   int eh; K ek;
                   if ((eh = e.hash) == h &&
                           ((ek = e.key) == k || (ek != null && k.equals(ek))))
                       return e;
                   if (eh < 0) {
                       if (e instanceof ForwardingNode) {
                           tab = ((ForwardingNode<K,V>)e).nextTable;
                           continue outer;
                       }
                       else
                           return e.find(h, k);
                   }
                   if ((e = e.next) == null)
                       return null;
               }
           }
       }
   }
```

## 3.方法

### 3.1.构造函数

ConcurrentHashMap提供了一系列的构造函数用于创建ConcurrentHashMap对象：

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

### 3.2.初始化： initTable()

ConcurrentHashMap的初始化主要由initTable()方法实现，在上面的构造函数中我们可以看到，其实ConcurrentHashMap在构造函数中并没有做什么事，仅仅只是设置了一些参数而已。其真正的初始化是发生在插入的时候，例如put、merge、compute、computeIfAbsent、computeIfPresent操作时。其方法定义如下：

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //sizeCtl < 0 表示有其他线程在初始化，该线程必须挂起
        if ((sc = sizeCtl) < 0)
            Thread.yield();
        // 如果该线程获取了初始化的权利，则用CAS将sizeCtl设置为-1，表示本线程正在初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                // 进行初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 下次扩容的大小
                    sc = n - (n >>> 2); ///相当于0.75*n 设置一个扩容的阈值
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

初始化方法initTable()的关键就在于sizeCtl，该值默认为0，如果在构造函数时有参数传入该值则为2的幂次方。该值如果 < 0，表示有其他线程正在初始化，则必须暂停该线程。如果线程获得了初始化的权限则先将sizeCtl设置为-1，防止有其他线程进入，最后将sizeCtl设置0.75 * n，表示扩容的阈值。

### 3.3.put操作

ConcurrentHashMap最常用的put、get操作，ConcurrentHashMap的put操作与HashMap并没有多大区别，其核心思想依然是根据hash值计算节点插入在table的位置，如果该位置为空，则直接插入，否则插入到链表或者树中。但是ConcurrentHashMap会涉及到多线程情况就会复杂很多。我们先看源代码，然后根据源代码一步一步分析：

```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }

final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key、value均不能为null
    if (key == null || value == null) throw new NullPointerException();
    //计算hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // table为null，进行初始化工作
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //如果i位置没有节点，则直接插入，不需要加锁
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 有线程正在进行扩容操作，则先帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //对该节点进行加锁处理（hash值相同的链表的头节点），对性能有点儿影响
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //fh > 0 表示为链表，将该节点插入到链表尾部
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //hash 和 key 都一样，替换value
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //putIfAbsent()
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //链表尾部  直接插入
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    //树节点，按照树的插入操作进行插入
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
                // 如果链表长度已经达到临界值8 就需要把链表转换为树结构
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }

    //size + 1
    addCount(1L, binCount);
    return null;
}
```

按照上面的源码，我们可以确定put整个流程如下：

* 判空；ConcurrentHashMap的key、value都不允许为null
* 计算hash。利用方法计算hash值。

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

* 遍历table，进行节点插入操作，过程如下：
  * 如果table为空，则表示ConcurrentHashMap还没有初始化，则进行初始化操作：initTable()
  * 根据hash值获取节点的位置i，若该位置为空，则直接插入，这个过程是不需要加锁的。计算f位置：i=(n - 1) & hash
  * 如果检测到fh = f.hash == -1，则f是ForwardingNode节点，表示有其他线程正在进行扩容操作，则帮助线程一起进行扩容操作
  * 如果f.hash >= 0 表示是链表结构，则遍历链表，如果存在当前key节点则替换value，否则插入到链表尾部。如果f是TreeBin类型节点，则按照红黑树的方法更新或者增加节点
  * 若链表长度 > TREEIFY_THRESHOLD(默认是8)，则将链表转换为红黑树结构
* 调用addCount方法，ConcurrentHashMap的size + 1

这里整个put操作已经完成。


### 3.4.get操作

ConcurrentHashMap的get操作还是挺简单的，无非就是通过hash来找key相同的节点而已，当然需要区分链表和树形两种情况。

```java
 public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        // 搜索到的节点key与传入的key相同且不为null,直接返回这个节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 树
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 链表，遍历
        while ((e = e.next) != null) {
            if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

get操作的整个逻辑非常清楚：

* 计算hash值
* 判断table是否为空，如果为空，直接返回null
* 根据hash值获取table中的Node节点（tabAt(tab, (n - 1) & h)），然后根据链表或者树形方式找到相对应的节点，返回其value值。

### 3.5.size操作

ConcurrentHashMap的size()方法我们虽然用得不是很多，但是我们还是很有必要去了解的。ConcurrentHashMap的size()方法返回的是一个不精确的值，因为在进行统计的时候有其他线程正在进行插入和删除操作。当然为了这个不精确的值，ConcurrentHashMap也是操碎了心。

为了更好地统计size，ConcurrentHashMap提供了baseCount、counterCells两个辅助变量和一个CounterCell辅助内部类。

```java
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

//ConcurrentHashMap中元素个数,但返回的不一定是当前Map的真实元素个数。基于CAS无锁更新
private transient volatile long baseCount;

private transient volatile CounterCell[] counterCells;
```

这里我们需要清楚CounterCell 的定义

size()方法定义如下：

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

内部调用sunmCount()：

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            //遍历，所有counter求和
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

sumCount()就是迭代counterCells来统计sum的过程。我们知道put操作时，肯定会影响size()，我们就来看看CouncurrentHashMap是如何为了这个不和谐的size()操碎了心。

在put()方法最后会调用addCount()方法，该方法主要做两件事，一件更新baseCount的值，第二件检测是否进行扩容，我们只看更新baseCount部分：

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // s = b + x，完成baseCount++操作；
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 在CounterCells未初始化
        // 或尝试通过CAS更新当前线程的CounterCell失败时
        // 调用fullAddCount()，该函数负责初始化CounterCells和更新计数
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //  多线程CAS发生失败时执行
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
// 检查是否进行扩容
}
```

x == 1，如果counterCells == null，则U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)，如果并发竞争比较大可能会导致改过程失败，如果失败则最终会调用fullAddCount()方法。其实为了提高高并发的时候baseCount可见性的失败问题，又避免一直重试，JDK 8 引入了类Striped64,其中LongAdder和DoubleAdder都是基于该类实现的，而CounterCell也是基于Striped64实现的。如果counterCells ！= null，且uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x)也失败了，同样会调用fullAddCount()方法，最后调用sumCount()计算s。

#### 3.5.1 关于 CounterCell

> 以下引用自 https://sylvanassun.github.io/2018/03/16/2018-03-16-map_family/

counterCells是一个元素为CounterCell的数组，该数组的大小与当前机器的CPU数量有关，并且它不会被主动初始化，只有在调用`fullAddCount()`函数时才会进行初始化。

```java
/**
 * Table of counter cells. When non-null, size is a power of 2.
 */
private transient volatile CounterCell[] counterCells;

/**
 * A padded cell for distributing counts.  Adapted from LongAdder
 * and Striped64.  See their internal docs for explanation.
 */
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

注解`@sun.misc.Contended`用于解决伪共享问题。所谓伪共享，即是在同一缓存行（CPU缓存的基本单位）中存储了多个变量，当其中一个变量被修改时，就会影响到同一缓存行内的其他变量，导致它们也要跟着被标记为失效，其他变量的缓存命中率将会受到影响。解决伪共享问题的方法一般是对该变量填充一些无意义的占位数据，从而使它独享一个缓存行。

ConcurrentHashMap的计数设计与LongAdder类似。**在一个低并发的情况下，就只是简单地使用CAS操作来对baseCount进行更新，但只要这个CAS操作失败一次，就代表有多个线程正在竞争，那么就转而使用CounterCell数组进行计数，数组内的每个ConuterCell都是一个独立的计数单元。**

每个线程都会通过`ThreadLocalRandom.getProbe() & m`寻址找到属于它的CounterCell，然后进行计数。ThreadLocalRandom是一个线程私有的伪随机数生成器，每个线程的probe都是不同的（这点基于ThreadLocalRandom的内部实现，它在内部维护了一个probeGenerator，这是一个类型为AtomicInteger的静态常量，每当初始化一个ThreadLocalRandom时probeGenerator都会先自增一个常量然后返回的整数即为当前线程的probe，probe变量被维护在Thread对象中），可以认为每个线程的probe就是它在CounterCell数组中的hash code。

**这种方法将竞争数据按照线程的粒度进行分离**，相比所有竞争线程对一个共享变量使用CAS不断尝试在性能上要效率多了，这也是为什么在高并发环境下LongAdder要优于AtomicInteger的原因。

如果两次CAS失败了就调用 fullAddCount

`fullAddCount()`函数的主要流程如下：

- 首先检查当前线程有没有初始化过ThreadLocalRandom，如果没有则进行初始化。ThreadLocalRandom负责更新线程的probe，而probe又是在数组中进行寻址的关键。
- 检查CounterCell数组是否已经初始化，如果已初始化，那么就根据probe找到对应的CounterCell。
  - 如果这个CounterCell等于null，需要先初始化CounterCell，通过把计数增量传入构造函数，所以初始化只要成功就说明更新计数已经完成了。初始化的过程需要获取自旋锁。
  - 如果不为null，就按上文所说的逻辑对CounterCell实施更新计数。
- CounterCell数组未被初始化，尝试获取自旋锁，进行初始化。数组初始化的过程会附带初始化一个CounterCell来记录计数增量，所以只要初始化成功就表示更新计数完成。
- 如果自旋锁被其他线程占用，无法进行数组的初始化，只好通过CAS更新baseCount。

`fullAddCount()`函数根据当前线程的probe寻找对应的CounterCell进行计数，如果CounterCell数组未被初始化，则初始化CounterCell数组和CounterCell。

> 该函数的实现与Striped64类（LongAdder的父类）的`longAccumulate()`函数是一样的，把CounterCell数组当成一个散列表，每个线程的probe就是hash code，散列函数也仅仅是简单的`(n - 1) & probe`。
>
> CounterCell数组的大小永远是一个2的n次方，初始容量为2，每次扩容的新容量都是之前容量乘以二，处于性能考虑，它的最大容量上限是机器的CPU数量。

所以说CounterCell数组的碰撞冲突是很严重的，因为它的bucket基数太小了。而发生碰撞就代表着一个CounterCell会被多个线程竞争，为了解决这个问题，Doug Lea使用无限循环加上CAS来模拟出一个自旋锁来保证线程安全，自旋锁的实现基于一个被`volatile`修饰的整数变量，该变量只会有两种状态：0和1，当它被设置为0时表示没有加锁，当它被设置为1时表示已被其他线程加锁。这个自旋锁用于保护初始化CounterCell、初始化CounterCell数组以及对CounterCell数组进行扩容时的安全。

CounterCell更新计数是依赖于CAS的，每次循环都会尝试通过CAS进行更新，如果成功就退出无限循环，否则就调用`ThreadLocalRandom.advanceProbe()`函数为当前线程更新probe，然后重新开始循环，以期望下一次寻址到的CounterCell没有被其他线程竞争。

如果连着两次CAS更新都没有成功，那么会对CounterCell数组进行一次扩容，这个扩容操作只会在当前循环中触发一次，而且只能在容量小于上限时触发。

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 当前线程的probe等于0，证明该线程的ThreadLocalRandom还未被初始化
    // 以及当前线程是第一次进入该函数
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        // 初始化ThreadLocalRandom，当前线程会被设置一个probe
        ThreadLocalRandom.localInit();      // force initialization
        // probe用于在CounterCell数组中寻址
        h = ThreadLocalRandom.getProbe();
        // 未竞争标志
        wasUncontended = true;
    }
    // 冲突标志
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // CounterCell数组已初始化
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // 如果寻址到的Cell为空，那么创建一个新的Cell
            if ((a = as[(n - 1) & h]) == null) {
                // cellsBusy是一个只有0和1两个状态的volatile整数
                // 它被当做一个自旋锁，0代表无锁，1代表加锁
                if (cellsBusy == 0) {            // Try to attach new Cell
                    // 将传入的x作为初始值创建一个新的CounterCell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    // 通过CAS尝试对自旋锁加锁
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        // 加锁成功，声明Cell是否创建成功的标志
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            // 再次检查CounterCell数组是否不为空
                            // 并且寻址到的Cell为空
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                // 将之前创建的新Cell放入数组
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            // 释放锁
                            cellsBusy = 0;
                        }
                        // 如果已经创建成功，中断循环
                        // 因为新Cell的初始值就是传入的增量，所以计数已经完毕了
                        if (created)
                            break;
                        // 如果未成功
                        // 代表as[(n - 1) & h]这个位置的Cell已经被其他线程设置
                        // 那么就从循环头重新开始
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            // as[(n - 1) & h]非空
            // 在addCount()函数中通过CAS更新当前线程的Cell进行计数失败
            // 会传入wasUncontended = false，代表已经有其他线程进行竞争
            else if (!wasUncontended)       // CAS already known to fail
                // 设置未竞争标志，之后会重新计算probe，然后重新执行循环
                wasUncontended = true;      // Continue after rehash
            // 尝试进行计数，如果成功，那么就退出循环
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            // 尝试更新失败，检查counterCell数组是否已经扩容
            // 或者容量达到最大值（CPU的数量）
            else if (counterCells != as || n >= NCPU)
                // 设置冲突标志，防止跳入下面的扩容分支
                // 之后会重新计算probe
                collide = false;            // At max size or stale
            // 设置冲突标志，重新执行循环
            // 如果下次循环执行到该分支，并且冲突标志仍然为true
            // 那么会跳过该分支，到下一个分支进行扩容
            else if (!collide)
                collide = true;
            // 尝试加锁，然后对counterCells数组进行扩容
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    // 检查是否已被扩容
                    if (counterCells == as) {// Expand table unless stale
                        // 新数组容量为之前的1倍
                        CounterCell[] rs = new CounterCell[n << 1];
                        // 迁移数据到新数组
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    // 释放锁
                    cellsBusy = 0;
                }
                collide = false;
                // 重新执行循环
                continue;                   // Retry with expanded table
            }
            // 为当前线程重新计算probe
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // CounterCell数组未初始化，尝试获取自旋锁，然后进行初始化
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    // 初始化CounterCell数组，初始容量为2
                    CounterCell[] rs = new CounterCell[2];
                    // 初始化CounterCell
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            // 初始化CounterCell数组成功，退出循环
            if (init)
                break;
        }
        // 如果自旋锁被占用，则只好尝试更新baseCount
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

其实在1.8中，它不推荐size()方法，而是推崇mappingCount()方法，该方法的定义和size()方法基本一致：

```java
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}
```

### 3.6.扩容操作

ConcurrentHashMap的JDK8与JDK7版本的并发实现相比，最大的区别在于JDK8的锁粒度更细，理想情况下talbe数组元素的大小就是其支持并发的最大个数，在JDK7里面最大并发个数就是Segment的个数，默认值是16，可以通过构造函数改变一经创建不可更改，这个值就是并发的粒度，每一个segment下面管理一个table数组，加锁的时候其实锁住的是**整个segment，这样设计的好处在于数组的扩容是不会影响其他的segment**的，简化了并发设计，不足之处在于并发的粒度稍粗，所以在JDK8里面，去掉了分段锁，将锁的级别控制在了更细粒度的table元素级别，也就是说只需要锁住这个链表的head节点，并不会影响其他的table元素的读写，**好处在于并发的粒度更细，影响更小，从而并发效率更好，但不足之处在于并发扩容的时候，由于操作的table都是同一个，不像JDK7中分段控制，所以这里需要等扩容完之后，所有的读写操作才能进行，所以扩容的效率就成为了整个并发的一个瓶颈点**，好在Doug lea大神对扩容做了优化，本来在一个线程扩容的时候，如果影响了其他线程的数据，那么其他的线程的读写操作都应该阻塞，但Doug lea说你们闲着也是闲着，不如来一起参与扩容任务，这样人多力量大，办完事你们该干啥干啥，别浪费时间，于是在JDK8的源码里面就引入了一个ForwardingNode类，在一个线程发起扩容的时候，就会改变sizeCtl这个值，其含义如下：


```
sizeCtl ：默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。
-1 代表table正在初始化
-N 表示有N-1个线程正在进行扩容操作
其余情况：
1、如果table未初始化，表示table需要初始化的大小。
2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍
```

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-092115.png)


transferIndex属性

```java
private transient volatile int transferIndex;
  /**
  扩容线程每次最少要迁移16个hash桶
  */
private static final int MIN_TRANSFER_STRIDE = 16;
```

扩容索引，表示已经分配给扩容线程的table数组索引位置。主要用来协调多个线程，并发安全地
获取迁移任务（hash桶）。

在扩容之前，transferIndex 在数组的最右边 。此时有一个线程发现已经到达扩容阈值，准备开始扩容。

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-092203.png)

扩容线程，在迁移数据之前，首先要将transferIndex右移（以cas的方式修改 transferIndex=transferIndex-stride(要迁移hash桶的个数)），获取迁移任务。每个扩容线程都会通过for循环+CAS的方式设置transferIndex，因此可以确保多线程扩容的并发安全。

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-092230.png)

换个角度，我们可以将待迁移的table数组，看成一个任务队列，transferIndex看成任务队列的头指针。而扩容线程，就是这个队列的消费者。扩容线程通过CAS设置transferIndex索引的过程，就是消费者从任务队列中获取任务的过程。为了性能考虑，我们当然不会每次只获取一个任务（hash桶），因此ConcurrentHashMap规定，每次至少要获取16个迁移任务（迁移16个hash桶，MIN_TRANSFER_STRIDE = 16）

当ConcurrentHashMap中table元素个数达到了容量阈值（sizeCtl）时，则需要进行扩容操作。在put操作时最后一个会调用addCount(long x, int check)，该方法主要做两个工作：1.更新baseCount；2.检测是否需要扩容操作。如下：

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 更新baseCount

    //check >= 0 :则需要进行扩容操作
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) 
            // 扩容标志位
            int rs = resizeStamp(n);
            // 负数代表正在优先成进行扩容
            if (sc < 0) {
                // 扩容结束
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                    break;
                // 进行扩容，扩容线程 + 1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }

            //当前线程是唯一的或是第一个发起扩容的线程  此时nextTable=null
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                    (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}

```

transfer()方法为ConcurrentHashMap扩容操作的核心方法。由于ConcurrentHashMap支持多线程扩容，而且也没有进行加锁，所以实现会变得有点儿复杂。整个扩容操作分为两步：

1. 构建一个nextTable，其大小为原来大小的两倍，这个步骤是在单线程环境下完成的
2. 将原来table里面的内容复制到nextTable中，这个步骤是允许多线程操作的，所以性能得到提升，减少了扩容的时间消耗

我们先来看看源代码，然后再一步一步分析：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
      int n = tab.length, stride;
      // 每核处理的量小于16，则强制赋值16
      if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
          stride = MIN_TRANSFER_STRIDE; // subdivide range
      if (nextTab == null) {            // initiating
          try {
              @SuppressWarnings("unchecked")
              Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];        //构建一个nextTable对象，其容量为原来容量的两倍
              nextTab = nt;
          } catch (Throwable ex) {      // try to cope with OOME
              sizeCtl = Integer.MAX_VALUE;
              return;
          }
          nextTable = nextTab;
          transferIndex = n;
      }
      int nextn = nextTab.length;
      // 连接点指针，用于标志位（fwd的hash值为-1，fwd.nextTable=nextTab）
      ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
      // 当advance == true时，表明该节点已经处理过了
      boolean advance = true;
      boolean finishing = false; // to ensure sweep before committing nextTab
      for (int i = 0, bound = 0;;) {
          Node<K,V> f; int fh;
          // 控制 --i ,遍历原hash表中的节点
          while (advance) {
              int nextIndex, nextBound;
              if (--i >= bound || finishing)
                  advance = false;
              else if ((nextIndex = transferIndex) <= 0) {
                  i = -1;
                  advance = false;
              }
              // 用CAS计算得到的transferIndex
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
              // 已经完成所有节点复制了
              if (finishing) {
                  nextTable = null;
                  table = nextTab;        // table 指向nextTable
                  sizeCtl = (n << 1) - (n >>> 1);     // sizeCtl阈值为原来的1.5倍
                  return;     // 跳出死循环，
              }
              // CAS 更扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
              if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                  if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                      return;
                  finishing = advance = true;
                  i = n; // recheck before commit
              }
          }
          // 遍历的节点为null，则放入到ForwardingNode 指针节点
          else if ((f = tabAt(tab, i)) == null)
              advance = casTabAt(tab, i, null, fwd);
          // f.hash == -1 表示遍历到了ForwardingNode节点，意味着该节点已经处理过了
          // 这里是控制并发扩容的核心
          else if ((fh = f.hash) == MOVED)
              advance = true; // already processed
          else {
              // 节点加锁
              synchronized (f) {
                  // 节点复制工作
                  if (tabAt(tab, i) == f) {
                      Node<K,V> ln, hn;
                      // fh >= 0 ,表示为链表节点
                      if (fh >= 0) {
                          // 构造两个链表  一个是原链表  另一个是原链表的反序排列
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
                          // 在nextTable i 位置处插上链表
                          setTabAt(nextTab, i, ln);
                          // 在nextTable i + n 位置处插上链表
                          setTabAt(nextTab, i + n, hn);
                          // 在table i 位置处插上ForwardingNode 表示该节点已经处理过了
                          setTabAt(tab, i, fwd);
                          // advance = true 可以执行--i动作，遍历节点
                          advance = true;
                      }
                      // 如果是TreeBin，则按照红黑树进行处理，处理逻辑与上面一致
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

                          // 扩容后树节点个数若<=6，将树转链表
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

上面的源码有点儿长，稍微复杂了一些，在这里我们抛弃它多线程环境，我们从单线程角度来看：

* 为每个内核分任务，并保证其不小于16
* 检查nextTable是否为null，如果是，则初始化nextTable，使其容量为table的两倍
* 死循环遍历节点，知道finished：节点从table复制到nextTable中，支持并发，请思路如下：
  * 如果节点 f 为null，则插入ForwardingNode（采用Unsafe.compareAndSwapObjectf方法实现），这个是触发并发扩容的关键
  * 如果f为链表的头节点（fh >= 0）,则先构造一个反序链表（不应该是低位高位分插入？？），然后把他们分别放在nextTable的i和i + n位置，并将ForwardingNode 插入原节点位置，代表已经处理过了
  * 如果f为TreeBin节点，同样也是构造一个反序 ，同时需要判断是否需要进行unTreeify()操作，并把处理的结果分别插入到nextTable的i 和i+nw位置，并插入ForwardingNode 节点
* 所有节点复制完成后，则将table指向nextTable，同时更新sizeCtl = nextTable的0.75倍，完成扩容过程

在多线程环境下，ConcurrentHashMap用两点来保证正确性：ForwardingNode和synchronized。当一个线程遍历到的节点如果是ForwardingNode，则继续往后遍历，如果不是，则将该节点加锁，防止其他线程进入，完成后设置ForwardingNode节点，以便要其他线程可以看到该节点已经处理过了，如此交叉进行，高效而又安全。

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-090054.png)

#### 扩容步骤精简

1. 通过计算 CPU 核心数和 Map 数组的长度得到每个线程（CPU）要帮助处理多少个桶，并且这里每个线程处理都是平均的。默认每个线程处理 16 个桶。因此，如果长度是 16 的时候，扩容的时候只会有一个线程扩容。

2. 初始化临时变量 nextTable。将其在原有基础上扩容两倍。

3. 死循环开始转移。多线程并发转移就是在这个死循环中，根据一个 finishing 变量来判断，该变量为 true 表示扩容结束，否则继续扩容。

   3.1 进入一个 while 循环，分配数组中一个桶的区间给线程，默认是 16. 从大到小进行分配。当拿到分配值后，进行 i-- 递减。这个 i 就是数组下标。（`其中有一个 bound 参数，这个参数指的是该线程此次可以处理的区间的最小下标，超过这个下标，就需要重新领取区间或者结束扩容，还有一个 advance 参数，该参数指的是是否继续递减转移下一个桶，如果为 true，表示可以继续向后推进，反之，说明还没有处理好当前桶，不能推进`)
   3.2 出 while 循环，进 if 判断，判断扩容是否结束，如果扩容结束，清空临时变量，更新 table 变量，更新库容阈值。如果没完成，但已经无法领取区间（没了），该线程退出该方法，并将 sizeCtl 减一，表示扩容的线程少一个了。如果减完这个数以后，sizeCtl 回归了初始状态，表示没有线程再扩容了，该方法所有的线程扩容结束了。（`这里主要是判断扩容任务是否结束，如果结束了就让线程退出该方法，并更新相关变量`）。然后检查所有的桶，防止遗漏。
   3.3 如果没有完成任务，且 i 对应的槽位是空，尝试 CAS 插入占位符，让 putVal 方法的线程感知。
   3.4 如果 i 对应的槽位不是空，且有了占位符，那么该线程跳过这个槽位，处理下一个槽位。
   3.5 如果以上都是不是，说明这个槽位有一个实际的值。开始同步处理这个桶。
   3.6 到这里，都还没有对桶内数据进行转移，只是计算了下标和处理区间，然后一些完成状态判断。同时，如果对应下标内没有数据或已经被占位了，就跳过了。

4. 处理每个桶的行为都是同步的。防止 putVal 的时候向链表插入数据。
   4.1 如果这个桶是链表，那么就将这个链表根据 length 取于拆成两份，取于结果是 0 的放在新表的低位，取于结果是 1 放在新表的高位。
   4.2 如果这个桶是红黑数，那么也拆成 2 份，方式和链表的方式一样，然后，判断拆分过的树的节点数量，如果数量小于等于 6，改造成链表。反之，继续使用红黑树结构。
   4.3 到这里，就完成了一个桶从旧表转移到新表的过程。

1. Cmap 支持并发扩容，实现方式是，将表拆分，让每个线程处理自己的区间。如下图：

![img](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\280044-20190112230518240-1220814738.png)

 

假设总长度是 64 ，每个线程可以分到 16 个桶，各自处理，不会互相影响。

1. 而每个线程在处理自己桶中的数据的时候，是下图这样的：

![img](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\280044-20190112230534628-308549180.png)

 

 

扩容前的状态。

当对 4 号桶或者 10 号桶进行转移的时候，会将链表拆成两份，规则是根据节点的 hash 值取于 length，如果结果是 0，放在低位，否则放在高位。

因此，10 号桶的数据，黑色节点会放在新表的 10 号位置，白色节点会放在新桶的 26 号位置。

下图是循环处理桶中数据的逻辑：
![img](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\280044-20190112230600090-371021265.png)

 

 

处理完之后，新桶的数据是这样的：

 ![img](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\280044-20190112230629967-1417442033.png)

#### 总结

- transfer 方法可以说很牛逼，很精华，内部多线程扩容性能很高，
- 通过给每个线程分配桶区间，避免线程间的争用，通过为每个桶节点加锁，避免 putVal 方法导致数据不一致。同时，在扩容的时候，也会将链表拆成两份，这点和 HashMap 的 resize 方法类似。
- 而如果有新的线程想 put 数据时，也会帮助其扩容。鬼斧神工，令人赞叹。

在put操作时如果发现fh.hash = -1，则表示正在进行扩容操作，则当前线程会协助进行扩容操作。

```java
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
```

helpTransfer()方法为协助扩容方法，当调用该方法的时候，nextTable一定已经创建了，所以该方法主要则是进行复制工作。如下：

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

在扩容时读写操作如何进行

1. 对于get读操作，如果当前节点有数据，还没迁移完成，此时不影响读，能够正常进行。如果当前链表已经迁移完成，那么头节点会被设置成fwd节点，此时get线程会帮助扩容。
2. 对于put/remove写操作，如果当前链表已经迁移完成，那么头节点会被设置成fwd节点，此时写线程会帮助扩容，如果扩容没有完成，当前链表的头节点会被锁住，所以写线程会被阻塞，直到扩容完成。

### 3.7.转换红黑树

在put操作是，如果发现链表结构中的元素超过了TREEIFY_THRESHOLD（默认为8），则会把链表转换为红黑树，已便于提高查询效率。如下：

```java
if (binCount >= TREEIFY_THRESHOLD)
    treeifyBin(tab, i);
```

调用treeifyBin方法用与将链表转换为红黑树。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)//如果table.length<64 就扩大一倍 返回
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    //构造了一个TreeBin对象 把所有Node节点包装成TreeNode放进去
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);//这里只是利用了TreeNode封装 而没有利用TreeNode的next域和parent域
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    //在原来index的位置 用TreeBin替换掉原来的Node对象
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

从上面源码可以看出，构建红黑树的过程是同步的，进入同步后过程如下：

* 根据table中index位置Node链表，重新生成一个hd为头结点的TreeNode
* 根据hd头结点，生成TreeBin树结构，并用TreeBin替换掉原来的Node对象。
* 整个红黑树的构建过程有点儿复杂，关于ConcurrentHashMap 红黑树的构建过程，我们后续分析。

【注】：ConcurrentHashMap的扩容和链表转红黑树稍微复杂点，后续另起博文分析

## 4.红黑树

### 4.1.红黑树

先看红黑树的基本概念：红黑树是一课特殊的平衡二叉树，主要用它存储有序的数据，提供高效的数据检索，时间复杂度为O(lgn)。红黑树每个节点都有一个标识位表示颜色，红色或黑色，具备五种特性：

* 每个节点非红即黑
* 根节点为黑色
* 每个叶子节点为黑色。叶子节点为NIL节点，即空节点
* 如果一个节点为红色，那么它的子节点一定是黑色
* 从一个节点到该节点的子孙节点的所有路径包含相同个数的黑色节点

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-112746.png)

#### 4.1.1.旋转

当对红黑树进行插入和删除操作时可能会破坏红黑树的特性。为了继续保持红黑树的性质，则需要通过对红黑树进行旋转和重新着色处理，其中旋转包括左旋、右旋

左旋
左旋示意图如下:

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2018120822002.gif)

左旋处理过程比较简单，将E的右孩子S调整为E的父节点、S节点的左孩子作为调整后E节点的右孩子。

右旋

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2018120822003.gif)

由于链表转换为红黑树只有添加操作，加上篇幅有限所以这里就只介绍红黑树的插入操作，关于红黑树的详细情况，烦请各位Google。

在分析过程中，我们已下面一颗简单的树为案例，根节点G、有两个子节点P、U，我们新增的节点为N

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-113351.png)

#### 4.1.2.红黑树插入节点


红黑树默认插入的节点为红色，因为如果为黑色，则一定会破坏红黑树的规则5（从一个节点到该节点的子孙节点的所有路径包含相同个数的黑色节点）。尽管默认的节点为红色，插入之后也会导致红黑树失衡。红黑树插入操作导致其失衡的主要原因在于插入的当前节点与其父节点的颜色冲突导致（红红，违背规则4：如果一个节点为红色，那么它的子节点一定是黑色）。

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-113450.png)

要解决这类冲突就靠上面三个操作：左旋、右旋、重新着色。由于是红红冲突，那么其祖父节点一定存在且为黑色，但是叔父节点U颜色不确定，根据叔父节点的颜色则可以做相应的调整。

**叔父U节点是红色**

如果叔父节点为红色，那么处理过程则变得比较简单了：更换G与P、U节点的颜色，下图（一）。

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-113622.png)

当然这样变色可能会导致另外一个问题了，就是父节点G与其父节点GG颜色冲突（上图二），那么这里需要将G节点当做新增节点进行递归处理。


**叔父U节点为黑叔**

如果当前节点的叔父节点U为黑色，则需要根据当前节点N与其父节点P的位置决定，分为四种情况：

* N是P的右子节点、P是G的右子节点
* N是P的左子节点，P是G的左子节点
* N是P的左子节点，P是G的右子节点
* N是P的右子节点，P是G的左子节点

情况1、2称之为外侧插入、情况3、4是内侧插入，之所以这样区分是因为他们的处理方式是相对的。

**外侧插入**

以N是P的右子节点、P是G的右子节点为例，这种情况的处理方式为：以P为支点进行左旋，然后交换P和G的颜色（P设置为黑色，G设置为红色），如下：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-113936.png)

左外侧的情况（N是P的左子节点，P是G的左子节点）和上面的处理方式一样，先右旋，然后重新着色。

**内侧插入**

以N是P的左子节点，P是G的右子节点情况为例。内侧插入的情况稍微复杂些，经过一次旋转、着色是无法调整为红黑树的，处理方法如下：先进行一次右旋，再进行一次左旋，然后重新着色，即可完成调整。注意这里两次右旋都是以新增节点N为支点不是P。这里将N节点的两个NIL节点命名为X、L。如下：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-114408.png)

### 4.2.ConcurrentHashMap链表转红黑树源码分析

ConcurrentHashMap的链表转换为红黑树过程就是一个红黑树增加节点的过程。在put过程中，如果发现链表结构中的元素超过了TREEIFY_THRESHOLD（默认为8），则会把链表转换为红黑树：

```java
if (binCount >= TREEIFY_THRESHOLD)
    treeifyBin(tab, i);
```

treeifyBin主要的功能就是把链表所有的节点Node转换为TreeNode节点，如下：

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

* 先判断当前Node的数组长度是否小于MIN_TREEIFY_CAPACITY（64）
  * 如果小于则调用tryPresize扩容处理以缓解单个链表元素过大的性能问题。
  * 否则则将Node节点的链表转换为TreeNode的节点链表，构建完成之后调用setTabAt()构建红黑树。

TreeNode继承Node，如下：

```java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
            TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }
......
}
```

我们以下面一个链表作为案例，结合源代码来分析ConcurrentHashMap创建红黑树的过程：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-115016.png)

12

12作为跟节点，直接为将红编程黑即可，对应源码：

```java
next = (TreeNode<K,V>)x.next;
x.left = x.right = null;
if (r == null) {
    x.parent = null;
    x.red = false;
    r = x;
}
```

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-115122.png)

1

此时根节点root不为空，则插入节点时需要找到合适的插入位置，源码如下：

```java
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
```

1. 计算节点的hash值 p。dir 表示为往左移还是往右移。x 表示要插入的节点，p 表示带比较的节点。
2. 从根节点出发，计算比较节点p的的hash值 ph ，若ph > h ,则dir = -1 ,表示左移，取p = p.left。若p == null则插入，若 p != null，则继续比较，直到直到一个合适的位置，最后调用balanceInsertion()方法调整红黑树结构。ph < h，右移。
3. 如果ph = h，则表示节点“冲突”（和HashMap冲突一致），那怎么处理呢？首先调用comparableClassFor()方法判断节点的key是否实现了Comparable接口，如果kc ！= null ，则通过compareComparables()方法通过compareTo()比较带下，如果还是返回 0，即dir == 0，则调用tieBreakOrder()方法来比较了。tieBreakOrder如下：

```java
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
        (d = a.getClass().getName().
         compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
             -1 : 1);
    return d;
}
```

tieBreakOrder()方法最终还是通过调用System.identityHashCode()方法来比较。

确定插入位置后，插入，由于插入的节点有可能会打破红黑树的结构，所以插入后调用balanceInsertion()方法来调整红黑树结构。

```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    x.red = true;       // 所有节点默认插入为红
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {

        // x.parent == null，为跟节点，置黑即可
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        // x 父节点为黑色，或者x 的祖父节点为空，直接插入返回
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;

        /*
         * x 的 父节点为红色
         * ---------------------
         * x 的 父节点 为 其祖父节点的左子节点
         */
        if (xp == (xppl = xpp.left)) {
            /*
             * x的叔父节点存在，且为红色，颜色交换即可
             * x的父节点、叔父节点变为黑色，祖父节点变为红色
             */
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                /*
                 * x 为 其父节点的右子节点，则为内侧插入
                 * 则先左旋，然后右旋
                 */
                if (x == xp.right) {
                    // 左旋
                    root = rotateLeft(root, x = xp);
                    // 左旋之后x则会变成xp的父节点
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }

                /**
                 * 这里有两部分。
                 * 第一部分：x 原本就是其父节点的左子节点，则为外侧插入，右旋即可
                 * 第二部分：内侧插入后，先进行左旋，然后右旋
                 */
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }

        /**
         * 与上相对应
         */
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
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
```

回到节点1，其父节点为黑色，即：

```java
else if (!xp.red || (xpp = xp.parent) == null)
    return root;
```

直接插入：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120536.png)

9

9作为1的右子节点插入，但是存在红红冲突，此时9的并没有叔父节点。9的父节点1为12的左子节点，9为其父节点1的右子节点，所以处理逻辑是先左旋，然后右旋，对应代码如下：

```java
if (x == xp.right) {
    root = rotateLeft(root, x = xp);
    xpp = (xp = x.parent) == null ? null : xp.parent;
}
if (xp != null) {
    xp.red = false;
    if (xpp != null) {
        xpp.red = true;
        root = rotateRight(root, xpp);
    }
}
```

图例变化如下：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120618.png)

2

节点2 作为1 的右子节点插入，红红冲突，切其叔父节点为红色，直接变色即可：

```java
if ((xppr = xpp.right) != null && xppr.red) {
    xppr.red = false;
    xp.red = false;
    xpp.red = true;
    x = xpp;
}
```

对应图例为：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120727.png)

0

节点0作为1的左子节点插入，由于其父节点为黑色，不会插入后不会打破红黑树结构，直接插入即可：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120756.png)

11

节点11作为12的左子节点，其父节点12为黑色，和0一样道理，直接插入：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120827.png)

7

节点7作为2右子节点插入，红红冲突，其叔父节点0为红色，变色即可：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120852.png)

19

节点19作为节点12的右子节点，直接插入：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120919.png)

至此，整个过程已经完成了，最终结果如下：

![image](E:\研究生学习\Work\技术笔记\ConcurrentHashMap.assets\2019-03-19-120956.png)

### 4.3.链表转红黑树案例

