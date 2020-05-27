# Java HashMap 相关

<img src="https://pic2.zhimg.com/80/26341ef9fe5caf66ba0b7c40bba264a5_hd.png" style="zoom:50%;" />

## HashMap概述

- HashMap是一个散列桶(直观的说出来HashMap的数据结构特性：数组和链表)，存储的是key-value(键值对)映射
- HashMap采用了数组和链表的数据结构，能在查询和修改便于继承了数组的线性查找和链表的寻址修改提高效率
- HashMap是非安全的(非synchronized)，所以不保证安全的情况下，速度很快
- HashMap可以是key和value都是null，而Hashtable则不能(原因是Hashtable使用equal方法会产生空指针异常，而HashMap经过API处理过的，不会出现在这种情况)
  

## HashMap 的数据结构

### JDK 1.8 之前

JDK1.8 之前 `HashMap` 底层是 **数组和链表** 结合在一起使用也就是 **链表散列**。

* **HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash 值**
* **通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的是数组的长度）**
* **如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法（将冲突的值插入链表中）解决冲突。**

所谓扰动函数指的就是 HashMap 的 hash 方法。使用 hash 方法也就是扰动函数是为了防止一些实现比较差的 hashCode() 方法 换句话说使用扰动函数之后可以减少碰撞。

**JDK 1.8 HashMap 的 hash 方法源码:**

```java
    static final int hash(Object key) {
      int h;
      // key.hashCode()：返回散列值也就是hashcode
      // ^ ：按位异或
      // >>>:无符号右移，忽略符号位，空位都以0补齐
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```

> 解决Hash冲突的三种方法：
>
> 对于一个数组，我们在O(1)时间复杂度完成可以完成其索引的查找，现在我们想要存储一个key value的键值对，我们可以根据key生成一个数组的索引，然后在索引存储这个key value键值对。我们把根据key生成索引这个函数叫做散列函数。显而易见，不同的key有可能产生相同的索引，这就叫做哈希碰撞，也叫哈希冲突。一个优秀的哈希函数应该尽可能避免哈希碰撞
> 
>
> * **拉链法**：我们采用链表（jdk1.8之后采用链表+红黑树）的数据结构来去存取发生哈希冲突的输入域的关键字
>
> * **开放地址法**：所有输入的元素全部存放在哈希表里。发生哈希冲突，就以当前地址为基准，根据再寻址的方法（探查序列），去寻找下一个地址，若发生冲突再去寻找，直至找到一个为空的地址为止。所以这种方法又称为再散列法。
>
>   * 线性探查
>
>     dii=1，2，3，…，m-1；这种方法的特点是：冲突发生时，顺序查看表中下一单元，直到找出一个空单元或查遍全表。
>
>     （使用例子：ThreadLocal里面的ThreadLocalMap）
>
>   * 二次探查
>
>     di=12，-12，22，-22，…，k2，-k2    ( k<=m/2 )；这种方法的特点是：冲突发生时，在表的左右进行跳跃式探测，比较灵活。
>
>   * 伪随机探测
>
>   ​         di=伪随机数序列；具体实现时，应建立一个伪随机数发生器，（如i=(i+p) %m），生成一个位随机   序列，并给定一个随机数做起点，每次去加上这个伪随机数++就可以了。
>
> * **再散列**：发生冲突时再次计算散列值



### JDK1.8 源码

JDK1.8之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间

> 红黑树是一种含有红黑结点并能自平衡的二叉查找树。它必须满足下面性质：
>
> - 性质1：每个节点要么是黑色，要么是红色。
> - 性质2：根节点是黑色。
> - 性质3：每个叶子节点（NIL）是黑色。
> - 性质4：每个红色结点的两个子结点一定都是黑色。
> - **性质5：任意一结点到每个叶子结点的路径都包含数量相同的黑结点。**
>
> 从性质5又可以推出：
>
> - 性质5.1：如果一个结点存在黑子结点，那么该结点肯定有两个子结点



```java
Node[] table = new Node[16];    //散列桶初始化，table

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;    //用来定位数组索引位置
    final K key;
    V value;
    Node<K,V> next;   //链表的下一个node

    Node(int hash, K key, V value, Node<K,V> next) { ... }
    public final K getKey(){ ... }
    public final V getValue() { ... }
    public final String toString() { ... }
    public final int hashCode() { ... }
    public final V setValue(V newValue) { ... }
    public final boolean equals(Object o) { ... }
}
```



## HashMap 的 Hash 方法

理论上来说，哈希桶越大，存储的值就越松散，碰撞次数越少。哈希桶越大，即使再好hash方法也存在较多的碰撞问题。

不管增加、删除、查找键值对，定位到哈希桶数组的位置都是很关键的第一步。前面说过HashMap的数据结构是数组和链表的结合，所以我们当然希望这个HashMap里面的元素位置尽量分布均匀些，尽量使得每个位置上的元素数量只有一个，那么当我们用hash算法求得这个位置的时候，马上就可以知道对应位置的元素就是我们要的，不用遍历链表，大大优化了查询的效率。HashMap定位数组索引位置，直接决定了hash方法的离散性能

```java
方法一：
static final int hash(Object key) {   //jdk1.8 & jdk1.7
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     // ^ ：按位异或
     // >>>:无符号右移，忽略符号位，空位都以0补齐
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
方法二：1.7
static int indexFor(int h, int length) {  
     return h & (length - 1);  //第三步 取模运算
}
```

JDK 1.8 优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销

右位移16位，正好是32bit的一半，自己的高半区和低半区做异或，就是为了混合原始哈希码的高位和低位，以此来加大低位的随机性。而且混合后的低位掺杂了高位的部分特征，这样高位的信息也被变相保留下来。



<img src="https://pic2.zhimg.com/80/8e8203c1b51be6446cda4026eaaccf19_hd.png" style="zoom: 67%;" />

#### HashMap 的长度为什么是2的幂次方

* 为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。
* Hash 值的范围值-2147483648到2147483647，前后加起来大概40亿的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。
* 但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ `(n - 1) & hash`”。（n代表数组长度）。这也就解释了 HashMap 的长度为什么是2的幂次方。

**这个算法应该如何设计呢？**

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：**“取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash % length == hash & (length - 1)的前提是 length 是2的 n 次方；）。”** 并且 **采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方**



## HashMap 的 put 方法

1. **预判断是否扩容**：判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；
2. **计算存储的hash值和索引**：根据键值key计算hash值得到插入的数组索引i，
   1. 如果table[i]==null，直接新建节点添加，转向⑥，
   2. 如果table[i]不为空，转向③；
3. **处理hash冲突：**判断table[i]的首个元素是否和key一样（hashCode以及equals）
   * 如果相同直接覆盖value，
   * 如果不相同否则转向④，
4. **判断是要冲突位置的结点否是红黑树**：判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；
5. **遍历链表判断是否要转化为红黑树**：遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；
6. **插入成功后根据threshold再判断是否扩容**插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。



## HashMap 的 get 方法

```java 
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```



当我们调用get方法时，HashMap会使用键对象的hashcode找到bucket的位置，找到bucket位置后，会调用keys.equal()方法去找到链表中正确节点，最终找到值对象



## HashMap 的 resize 方法 -- 扩容机制

###  与扩容有关的字段

```java 
     int threshold;             // 所能容纳的key-value对极限 
     final float loadFactor;    // 负载因子
     int modCount;              // 记录HashMap内部结构变化的次数（put键值对覆盖不算）
     int size;					// 实际的键值对的数量
```

`Load factor`为负载因子(默认值是0.75)，threshold是HashMap所能容纳的最大数据量的Node(键值对)个数。`threshold = length * LoadFactor`。也就是说，在数组定义好长度之后，负载因子越大，所能容纳的键值对个数越多。 

threshold就是在此LoadFactor和length(数组长度)对应下允许的最大元素数目，超过这个数目就重新resize(扩容)，扩容后的HashMap容量是之前容量的两倍。

### JDK 1.7 扩容：

```java
 void resize(int newCapacity) {   //传入新的容量
     Entry[] oldTable = table;    //引用扩容前的Entry数组
     int oldCapacity = oldTable.length;         
     if (oldCapacity == MAXIMUM_CAPACITY) {  //扩容前的数组大小如果已经达到最大(2^30)了
        threshold = Integer.MAX_VALUE; //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
           return;
      }
  
     Entry[] newTable = new Entry[newCapacity];  //初始化一个新的Entry数组
     transfer(newTable);                         //！！将数据转移到新的Entry数组里
     table = newTable;                           //HashMap的table属性引用新的Entry数组
     threshold = (int)(newCapacity * loadFactor);//修改阈值
 }
```

原数组中的数据必须重新计算其在新数组中的位置，并放进去，这就是resize。

* 到超过一定的`threshold` 时，会调用 `resize()`  方法进行扩容，将哈希桶的容量扩大至原来的两倍
* 然后遍历所有的结点再重新计算位置 （rehash）
  * `index = node.hash & (table.length-1)`
  * 遍历数组中所有的结点，并计算出结点的Hash值，并将Hash值与数组的长度减1进行&运算得出数组下标
  * 一般位置有两种情况，第一种是原位置，第二种是原位置+新的位置

![](E:\研究生学习\Typora图片\e5aa99e811d1814e010afa7779b759d4_hd.png)

这里假设负载因子 loadFactor=1，即当键值对的实际大小 size 大于 table的实际大小时进行扩容。接下来的三个步骤是哈希桶数组 resize成4，然后所有的Node重新rehash的过程。



### JDK  1.8 扩容：

在扩容之后，一个元素的新索引要么是在原位置，要么就是在原位置加上扩容前的容量。这个方法的巧妙之处全在于&运算，之前提到过&运算只会关注n - 1（n = 数组长度）的有效位，当扩容之后，n的有效位相比之前会多增加一位（n会变成之前的二倍，所以确保数组长度永远是2次幂很重要），然后**只需要判断hash在新增的有效位的位置是0还是1**就可以算出新的索引位置，如果是0，那么索引没有发生变化，如果是1，索引就为原索引加上扩容前的容量。 e.hash & oldCap

* 如果计算出来的值为0，将旧数组当前位置的Node链表赋值到新数组相同位置，即`newTab[j] = loHead`
* 如果计算出来的值不为0，此时将当前位置的Node链表赋值给新数组当前位置加上旧数组长度的位置，即`newTab[j + oldCap] = hiHead` (原位置 + 旧的数组长度)
* 相当于平移了2的倍数的位置

如下图 （a）n = 16 扩容前 key 1 和 key 2 的 hash 值和其对应的运算结果（b） 扩容 n = 32 

![](E:\研究生学习\Typora图片\a285d9b2da279a18b052fe5eed69afe9_hd.png)

![](E:\研究生学习\Typora图片\b2cb057773e3d67976c535d6ef547d51_hd.png)

**Hash值与旧数组长度的&运算不为0**

```
node.hash & oldTable.length != 0;
```

我们假设Node结点的Hash值的二进制是1000010101，旧数组长度为16，二进制即10000。此时计算出来的index为5。

```
Hash     ：1000010101
length-1 ：0000001111
           ——————————
index    : 0000000101
```


当我们对数组进行扩容，数组的长度变成了32，Node结点的Hash值依然为1000010101。此时计算出来的index为21。

```
Hash     ：1000010101
length-1 ：0000011111
           ——————————
index    : 0000010101
```


结点平移后，此时计算出来的存放结点的新数组下标为旧数组下标加上旧数组的长度，即`newIndex = oldIndex+oldLength`

**Hash值与旧数组长度的&运算为0**

```
node.hash & oldTable.length == 0;
```

我们假设Node结点的Hash值的二进制是1000000101，旧数组长度为16，二进制即10000。此时计算出来的index为5。

```
Hash     ：1000010101
length-1 ：0000001111
           ——————————
index    : 0000000101
```


当我们对数组进行扩容，数组的长度变成了32，Node几点的Hash值依然为1000000101。此时计算出来的index仍为5。

```
Hash     ：1000000101
length-1 ：0000011111
           ——————————
index    : 0000000101
```


结点平移后，此时计算存放结点的新数组下标与旧数组下标相等，即newIndex = oldIndex。


**优点**：

* 省去了遍历Node的时候重新计算hash值的时间
* 由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了
* JDK1.7中rehash的时候，旧链表迁移新链表，如果在新表的数组索引位置相同，则链表元素会倒置。因为移动机制时通过头插法移动元素，遍历时也会从最新插入的元素开始遍历重新计算hash值。1.8则不会倒置
* 在JDK1.8前，**在多线程的情况下，使用HashMap进行put操作会造成死循环**。这是因为多次put操作会引发HashMap的扩容机制，HashMap的扩容机制采用头插法的方式移动元素，这样会造成链表闭环，形成死循环。JDK1.8中HashMap使用高低位来平移元素，这样保证效率的同时避免了多线程情况下扩容造成死循环的问题



## 多线程下 HashMap （JDK1.7）造成的死循环问题

```java
public class HashMapInfiniteLoop {  

    private static HashMap<Integer,String> map = new HashMap<Integer,String>(2，0.75f);  
    public static void main(String[] args) {  
        map.put(5， "C");  

        new Thread("Thread1") {  
            public void run() {  
                map.put(7, "B");  
                System.out.println(map);  
            };  
        }.start();  
        new Thread("Thread2") {  
            public void run() {  
                map.put(3, "A);  
                System.out.println(map);  
            };  
        }.start();        
    }  
}
                        
```

```java
void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}

```

其中，map初始化为一个长度为2的数组，loadFactor=0.75，threshold=2*0.75=1，也就是说当put第二个key的时候，map就需要进行resize。

使线程1在第九行进行挂起，先不进行扩容，此时注意，Thread1的 e 指向了key(3)，而next指向了key(7)。线程2进行resize后，**e 和 next 指向 指向了线程2 重组后哈希表中对应的链表中的 3 和 7**

![img](E:\研究生学习\Typora图片\fa10635a66de637fe3cbd894882ff0c7_hd.png)

(e  = key(3), next = key(7))

线程一被调度回来执行以下这一段代码:

```java
do {
    Entry<K,V> next = e.next;
    int i = indexFor(e.hash, newCapacity);
    e.next = newTable[i];
    newTable[i] = e;
    e = next;
} while (e != null);
```



1. 先是执行` newTalbe[i] = e`（e = key(3))

2.  然后是`e = next` 导致了e指向了key(7) `(next = key(7)，e = next = key(7))`

3. `（e ！= null）`进入下一次循环，`next = e.next `导致了 next 指向了key(3)

   执行结果如下图

![img](E:\研究生学习\Typora图片\d39d7eff6e8e04f98f5b53bebe2d4d7f_hd.png)

4. `e.next = newTable[i]` 导致 key(3).next 指向了 key(7)，而key(7).next 已经指向了key(3) (在线程2扩容时形成的）， 环形链表就这样出现了。

![img](E:\研究生学习\Typora图片\5f3cf5300f041c771a736b40590fd7b1_hd.png)





## ConcurrentHashMap 和 Hashtable 的区别

ConcurrentHashMap 和 Hashtable 的区别主要体现在实现线程安全的方式上不同。

- **底层数据结构：** JDK1.7的 ConcurrentHashMap 底层采用 **分段的数组+链表** 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 **数组+链表** 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；
- **实现线程安全的方式：**
  -  ① **在JDK1.7的时候，ConcurrentHashMap（分段锁）** 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 **到了 JDK1.8 的时候已经摒弃了Segment的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作，synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash不冲突，就不会产生并发** 
  - ② **Hashtable(同一把锁)** :使用全表锁。使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

> **什么是CAS？**
>
> CAS：Compare and Swap，即比较再交换。
>
> jdk5增加了并发包java.util.concurrent.*,其下面的类使用CAS算法实现了区别于synchronouse同步锁的一种乐观锁。JDK 5之前Java语言是靠synchronized关键字保证同步的，这是一种独占锁，也是是悲观锁。
>
> **CAS算法理解**
>
> 对CAS的理解，CAS是一种无锁算法，CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。
>
> CAS比较与交换的伪代码可以表示为：
>
> ```java
>do{
> 	备份旧数据；
>	基于旧数据构造新数据；
> } while(!CAS( 内存地址，备份的旧数据，新数据 ))
>```
> 
>CPU去更新一个值，但如果想改的值不再是原来的值，操作就失败，因为很明显，有其它操作先改变了这个值。
> 
>就是指当两者进行比较时，如果相等，则证明共享数据没有被修改，替换成新值，然后继续往下运行；如果不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作。容易看出 CAS 操作是基于共享数据不会被修改的假设，采用了类似于数据库的commit-retry 的模式。当同步冲突出现的机会很少时，这种假设能带来较大的性能提升



## HashMap中为什么初始化的大小为 16， 链表长度为8转化为红黑树，为什么为6的时候又变回链表？

### 初始容量 16

` int index =key.hashCode()&(length-1);`

因为HashMap的index计算是按位与，16 -1 = 15 --> 二进制 1111 末位是1，这样也能保证计算后的index既能是奇数也能是偶数，只要key比较分散，那么就能有效的减少hash碰撞，为什么一定是16？其实2的正数倍也可以，但是分配过小容易频繁扩容，分配太容易浪费资源，因此16是一个这种的选择

* **减少Hash碰撞**
* **2的倍数可以按位与，提高查找效率**
* **减少分配过小频繁扩容**
* **分配过大浪费资源**

### 为什么链表的长度为8是变成红黑树？为什么为6时又变成链表？

首先 链表的平均查找长度 为 n/2, 红黑树的查找长度为 log(length)。当链表长度为6是，链表平均查找长度为3，红黑树的长处为2.6。 为8时一个是4另一个是3。具体的值是由下面的值确定的。

`static final int TREEIFY_THRESHOLD = 8;`

理论上来说当 hashCode离散型越好时，数据越均匀分布在bin中。理想情况下hashCode算法下所有的bin结点分布会遵循泊松分布，当阈值为8的时候，一个bin中长度达到8的时候概率非常的小，从而转化为红黑树的概率小

也是为了一种时空的权衡





## ConcurrentHashMap
（从源码中就能看出不允许null key null value）所有操作都是线程安全的，put、get、remove都能同时进⾏，不会阻塞，迭代过程中，即使map的结构发⽣改变，也不会抛出异常，map是数组+链表+红⿊树，**ConcurrentHashMap在这个基础上加了个转移节点，来保证扩容时线程安全**。
ConcurrentHashMap和map的区别是，ConcurrentHashMap中的红⿊树，TreeNode仅仅是维护属性和查找的功能，**这⾥⾯加了个TreeBin来维护红⿊树的结构**，主要负责root的加锁、解锁。同时，ConcurrentHashMap中加了个**转移节点，如果转移节点有值，表示正在扩容，那么put操作就要等待，等扩容完才能继续放**，那么加
⼊了这个转移节点，就是为了维护线程的安全。

### put的步骤

1、来了⼀个key value，先判断是不是null的，是null，直接抛出异常。
2、不是null，那就继续，如果table是空的，**先要初始化**，然后先得到 key 的 hash，找到它在哪个槽中，然后进⼊⾃旋死循环。
3、进⼊⾃旋死循环后，如果这个位置上没有任何键值对，那就初始化，通过cas来创建，结束⾃旋，如果这个位置上有值了，我先看看当**前索引位置是不是转移节点，如果是转移节点，表示正在扩容，继续⾃旋，等扩容完再操作**。

4、如果不是转移节点，我先把这个位置**锁住，被我⼀个线程独占**，然后，判断是链表结构还是红⿊树结构，链
表结构的新增就和map⾥的新增是⼀样的，遍历链表，碰到⼀样的值直接break，否则就插到尾部，退出⾃旋

5、红⿊树的新增就和map的不⼀样了，它没有使⽤treenode，⽽是使⽤了**TreeBin，TreeBin持有红黑树的引用，会对红黑树的根节点加锁**，保证这棵树现在只有⼀个线程能旋转，保证线程的安全。在链表和红⿊树的新增都会改变

### CHM保证线程安全的三个方面：

**第⼀个：初始化**

是刚进来，**table还没初始化的时候。它先通过⾃旋来保证初始化⼀定成功**。初始化的时候⾥⾯有个sizeCtl，也就是size的偏移量，初始尾0，如果⼩于0，说明有线程正在初始化，其他线程你放开时间⽚吧，yield⼀下，如果⼤于0，就说明已经初始化好了。sizectl的值也是通过cas来设置的，这样保证了同⼀时刻只有⼀个线程在设置这个值，。cas完后，还会再 double check⼀下，所以 初始化的时候它是通过 ⾃旋+cas+double check来保证线程安全的。

**第二个：新增结点**

第⼆个是新增槽点的时候是这样的。它先看槽是不是空的，**如果是空的，就通过cas新增**（不能直接赋值，可能
期间被⼈插队了，⼀定要通过cas）。如果槽点**不是空的，它先会锁住槽点，再操作**，如果是链表，就链表新增如果是红⿊树，就红⿊树新增，**红⿊树新增也会通过TreeBin锁住根**，保证这棵树现在只有⼀个线程能旋转。

**第三个 ： 扩容**

第三个是扩容时的时候。它的扩容时机和map是⼀样的，都是在put完的时候扩容，但是扩容过程有区别，**map直**
**接在老数组上扩容**，⽽CHM通过addCount 来扩容，⾥⾯有⼀个transfer ⽅法，他就是主要的扩容⽅法。它要
**把⽼数组中的节点都拷⻉到新的数组中去，拷⻉的时候，先把原数组中的槽点锁住，这样原数组中的值就不能被操**
**作了，然后这个槽拷⻉完，它就成为了转移节点，继续扩容下⼀个**。它是从**链表尾部拷⻉到头部的，每拷⻉成功⼀**
**次，这个节点就变为转移节点，直到全部拷⻉完，那么就是得到新的容器**。

### get操作

整体思路和hashmap的思路就是很想的，先对得到key的hashcode，然后计算它在数组中的下标，得到在哪个槽
中，然后去这个槽的节点开始遍历，如果是链表就链表⽅式找，否则就是红⿊树的⽅式找，红⿊树还是根据
hashcode找吧，左⼩右⼤。

## 参考 

* https://blog.csdn.net/dgxin_605/article/details/86249771
* https://zhuanlan.zhihu.com/p/21673805
* https://blog.csdn.net/dgxin_605/article/details/86249771
* https://blog.csdn.net/weixin_41884010/article/details/100699047
* https://blog.csdn.net/qq_27409289/article/details/92759730



