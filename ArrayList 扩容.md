# ArrayList 扩容

## ArrayList 初始化

**ArrayList有三种方式来初始化，构造方法源码如下：**

```java
   /**
     * 默认初始容量大小
     */
    private static final int DEFAULT_CAPACITY = 10;
    

    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     *默认构造函数，使用初始容量10构造一个空列表(无参数构造)
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    /**
     * 带初始容量参数的构造函数。（用户自己指定容量）
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {//初始容量大于0
            //创建initialCapacity大小的数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {//初始容量等于0
            //创建空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {//初始容量小于0，抛出异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }


   /**
    *构造包含指定collection元素的列表，这些元素利用该集合的迭代器按顺序返回
    *如果指定的集合为null，throws NullPointerException。 
    */
     public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

**以无参数构造方法创建 ArrayList 时，实际上初始化赋值的是具有默认容量为10一个空数组**，



## ArrayList扩容机制

### `add` 方法

```java
 /**
  * 将指定的元素追加到此列表的末尾。 
   */
    public boolean add(E e) {
        //添加元素之前，先调用ensureCapacityInternal方法
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //这里看到ArrayList添加元素的实质就相当于为数组赋值
        elementData[size++] = e;
        return true;
    }
```

**加入时 minCapacity 为 数组的 size + 1**



### ` ensureCapacityInternal` 方法

```java
//得到最小扩容量
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
              // 获取默认的容量和传入参数的较大值
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

**当add第一个元素时，minCapacity为1，此时和默认的相比取最大值，minCapacity为10**



###  `ensureExplicitCapacity()` 方法

```java
//判断是否需要扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            //调用grow方法进行扩容，调用此方法代表已经开始扩容了
            grow(minCapacity);
    }
```



#### ArrayList何时开始扩容？

* 当加入（add）第1个元素时，因为无参数构造函数创建空的 list，此时 `elementData.length == 0` ，`minCapacity == 1`

* 执行 `ensureCapacityInternal(1)`  ，判断 `minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity)`  minCapacity 取默认值和现在数组容量的最大值，因此该方法执行完后，minCapacity = 10

* 执行`ensureExplicitCapacity(10)` 方法，`minCapacity - elementData.length > 0 `成立，执行 `grow(10)` 开始扩容，扩容到10

* 当加入 （add）第二个元素时， `minCapacity == 2`，此时e lementData.length(容量)在添加第一个元素后扩容成 10 了。此时，`minCapacity - elementData.length > 0 `不成立，所以不会进入 （执行）`grow(minCapacity)` 方法。

* 加入 3、4 到 10 个元素时都不会扩容

* 加入第 11 元素时， minCapacity 为 11， 比 elementData.length（为10） 大。进入grow扩容

  

### `grow()` 方法

```java
    /**
     * 要分配的最大数组大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * ArrayList扩容的核心方法。
     */
    private void grow(int minCapacity) {
        // oldCapacity为旧容量，newCapacity为新容量
        // oldCapacity等于现在元素的长度
        int oldCapacity = elementData.length;
        
        //将oldCapacity 右移一位，其效果相当于oldCapacity /2，
        //我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍，
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        
        //然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量，
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
       // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```



#### ArrayList 如何 扩容

* `oldCapacity =  elementData.length ` 旧容量是数组本身的容量，先按照数组本身的大小进行扩容

* **`int newCapacity = oldCapacity + (oldCapacity >> 1)` ,所以 ArrayList 每次扩容之后容量都会变为原来的 1.5 倍！（JDK1.6版本以后）** JDk1.6版本时，扩容之后容量为 1.5 倍+1！详情请参考源码
* **两次 if 判断**
  * 判断新扩容的容量（newCapacity ）是否大于最小需要容量（minCapacity）基于原来的数组容量扩容是否满足最小需要容量
  * 判断新扩容的容量（newCapacity ）是否大于 MAX_ARRAY_SIZE（MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8)

> ">>"（移位运算符）：>>1 右移一位相当于除2，右移n位相当于除以 2 的 n 次方。这里 oldCapacity 明显右移了1位所以相当于oldCapacity /2。对于大数据的2进制运算,位移运算符比那些普通运算符的运算要快很多,因为程序仅仅移动一下而已,不去计算,这样提高了效率,节省了资源 　

- 当add第1个元素时，oldCapacity 为0，经比较后第一个if判断成立，newCapacity = minCapacity(为10)。但是第二个if判断不会成立，即newCapacity 不比 MAX_ARRAY_SIZE大，则不会进入 `hugeCapacity` 方法。数组容量为10，add方法中 return true,size增为1。
- 当add第11个元素进入grow方法时，newCapacity为15，比minCapacity（为11）大，第一个if判断不成立。新容量没有大于数组最大size，不会进入hugeCapacity方法。数组容量扩为15，add方法中return true,size增为11。
- 以此类推······



### `hugeCapacity()` 方法。

从上面 `grow()` 方法源码我们知道： 如果新容量大于 MAX_ARRAY_SIZE,进入(执行)`hugeCapacity()` 方法

* 比较 minCapacity 和 MAX_ARRAY_SIZE
* 如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`
* 否则，新容量大小则为 MAX_ARRAY_SIZE 即为 `Integer.MAX_VALUE - 8`。

```java 
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
      
        return 
            (minCapacity > MAX_ARRAY_SIZE) ? 
            Integer.MAX_VALUE : MAX_ARRAY_SIZE;
    }
```



### 扩容机制中的 System.arraycopy() 和 Arrays.copyOf()

**联系：**

看两者源代码可以发现 copyOf() 内部实际调用了 `System.arraycopy()` 方法

**区别：**

`arraycopy()` 需要目标数组，将原数组拷贝到你自己定义的数组里或者原数组，而且可以选择拷贝的起点和长度以及放入新数组中的位置 

`copyOf()` 是系统自动在内部新建一个数组，并返回该数组。

