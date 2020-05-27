# JDK 动态代理

## 动态代理

**静态代理的缺点：**

1. 代理类和被代理类实现了相同的接口。导致代码的重复，如过接口新增加一方法，代理类和被代理类都要实现这个方法，增加代码维护的难度
2. 代理对象只服务于一种类型的对象，如果要服务多种类型的对象就要为每个对象都进行代理。

静态代理只能指定针对某个类，那么如果想在这个类的任意方法上都加上同样的逻辑（实现同样的代理）该如何做，可以通过JDK的动态代理，JDK动态代理可以通过反射机制来获取class和Method

### JDK自带方法

#### 步骤

   ①创建被代理的接口和类；
   ②创建InvocationHandler接口的实现类，在invoke方法中实现代理逻辑；
   ③通过Proxy的静态方法`newProxyInstance( ClassLoaderloader, Class[] interfaces, InvocationHandler h)`创建一个代理对象
   ④使用代理对象

首先介绍一下最核心的一个接口和一个方法：

首先是`java.lang.reflect`包里的`InvocationHandler`接口：

```java
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

我们对于**被代理的类的操作都会由该接口中的invoke方法实现**，其中的参数的含义分别是：

- proxy：被代理的类的实例
- method：调用被代理的类的方法 （通过反射机制获得）
- args：该方法需要的参数

使用方法首先是需要实现该接口，并且我们可以在invoke方法中调用被代理类的方法并获得返回值，自然也可以在调用该方法的前后去做一些额外的事情，从而实现动态代理，下面的例子会详细写到。

另外一个很重要的静态方法是 `java.lang.reflect` 包中的 `Proxy` 类的 `newProxyInstance ` 方法：

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
```

其中的参数含义如下：

- loader：指定代理类的类加载器
- interfaces：被代理类实现的接口数组，当传入代理类实现的接口，这样生成的代理类和被代理类实现了相同的接口）
- invocationHandler：就是刚刚介绍的调用处理器类的对象实例

该方法会返回一个被修改过的类的实例，从而可以自由的调用该实例的方法。

#### Demo 演示

Fruit接口：

```java
public interface Fruit {
    public void show();
}
```

Apple实现Fruit接口：

```java
public class Apple implements Fruit{
    @Override
    public void show() {
        System.out.println("<<<<show method is invoked");
    }
}
```

代理类Agent.java：

```java
public class DynamicAgent {

    //实现InvocationHandler接口，并且可以初始化被代理类的对象
    static class MyHandler implements InvocationHandler {
        private Object proxy;
        public MyHandler(Object proxy) {
            this.proxy = proxy;
        }
            
        //自定义invoke方法
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println(">>>>before invoking");
            //真正调用方法的地方
            Object ret = method.invoke(this.proxy, args);
            System.out.println(">>>>after invoking");
            return ret;
        }
    }

    //返回一个被修改过的对象
    public static Object agent(Class interfaceClazz, Object proxy) {
        return Proxy.newProxyInstance(interfaceClazz.getClassLoader(), new Class[]{interfaceClazz},
                new MyHandler(proxy));
    }    
}
```

测试类：

```java
public class ReflectTest {
    public static void main(String[] args) throws InvocationTargetException, IllegalAccessException {
        //注意一定要返回接口，不能返回实现类否则会报错
        Fruit fruit = (Fruit) DynamicAgent.agent(Fruit.class, new Apple());
        // 如果光将代理的内部类拿出来，那么当前类就为代理类，直接在main方法内返回代理对象
        // Friut fruit = Proxy.newProxyInstance(Reflect.class.getClassLoader(), new class[] {Friut}, new MyHandler(new Apple()))
        fruit.show();
    }
}
```



## JDK 动态代理源码解析

### 1. `Proxy.newProxyInstance` 方法

上述Demo代码中可以看到是由 `Proxy` 方法内的 `newProxyInstance` 方法产生代理类并返回

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
{
    // 检验 InvocationHandler 是否为空
    Objects.requireNonNull(h);
	// 接口的类对象拷贝
    final Class<?>[] intfs = interfaces.clone();
    // 安全解析
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    /*
      * Look up or generate the designated proxy class.
      * 查询（在缓存中已经有）或生成指定代理类的class对象
      */
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
      * Invoke its constructor with the designated invocation handler.
      */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
		
        // 得到代理类对象的构造函数，这个构造函数的参数由 constructorParams 指定
        // constructorParams 常量值
        // private static final Class<?>[] constructorParams = { InvocationHandler.class };
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        // 生成代理类对象
        return cons.newInstance(new Object[]{h});
        
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

上述代码去掉一些校验规则之后，其核心代码就是 

1. ```java
   Class<?> cl = getProxyClass0(loader, intfs);
   ```

    通过代理对象的类加载器和接口数组`ClassLoader` 和 `Class<?>[] interfaces` 查看缓存里是否存在或生成指定代理类的class对象

2. ```java
   private static final Class<?>[] constructorParams = { InvocationHandler.class };
   final Constructor<?> cons = cl.getConstructor(constructorParams);
   ```

   得到代理类对象的构造函数，构造函数的参数由 `constructorParams` 参数指定，

3. ```java
   return cons.newInstance(new Object[]{h});
   ```

   根据这个构造函数生成一个对象

接下来主要对以下代码进行分析

### 2. `getProxyClass0(loader, intfs)` 方法

```java
//此方法也是Proxy类下的方法
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    //意思是：如果代理类被指定的类加载器loader定义了，并实现了给定的接口interfaces，
    //那么就返回缓存的代理类对象，否则使用ProxyClassFactory创建代理类。
    return proxyClassCache.get(loader, interfaces);
}
```

这段代码的注释：如果代理类被指定的类加载器loader定义了，并实现了给定的接口interfaces，那么就返回缓存的代理类对象，否则使用**ProxyClassFactory**创建代理类。

接下来看一下 **ProxyClassFactory** 是什么

###  3. `proxyClassCache` 

```java
/**
 * a cache of proxy classes
 */
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

proxyClassCache 其实是个 **WeakCache** 对象，proxyClassCache 对象的 get 方法其实是 WeakCache 类的 get 方法。

注意对象内的泛型 `<ClassLoader, Class[?], Class<?>` 

* ClassLoader ： 代理对象的类加载器
* Class[]<?> : 代理对象实现的参数数组
* Class<?>：需要的代理类对象

构造方法的参数 `new KeyFactory(), new ProxyClassFactory()`

到这里可以看出代理对象的实例化与 **WeakCache** 对象密切相关

### 4 .`WeakCache` 对象的参数与构造参数

这里先看 这个对象的核心参数和构造参数

```Java
//K代表key的类型，P代表参数的类型，V代表value的类型。
// WeakCache<ClassLoader, Class<?>[], Class<?>>  proxyClassCache  说明proxyClassCache存的值是Class<?>对象，正是我们需要的代理类对象。
final class WeakCache<K, P, V> {

    private final ReferenceQueue<K> refQueue
        = new ReferenceQueue<>();
    // the key type is Object for supporting null key
    private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
    private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
        = new ConcurrentHashMap<>();
    
    private final BiFunction<K, P, ?> subKeyFactory;
    private final BiFunction<K, P, V> valueFactory;
	
     /**
     * Construct an instance of {@code WeakCache}
     *
     * @param subKeyFactory a function mapping a pair of
     *                      {@code (key, parameter) -> sub-key}
     * @param valueFactory  a function mapping a pair of
     *                      {@code (key, parameter) -> value}
     * @throws NullPointerException if {@code subKeyFactory} or
     *                              {@code valueFactory} is null.
     */
  
    public WeakCache(BiFunction<K, P, ?> subKeyFactory,
                     BiFunction<K, P, V> valueFactory) {
        this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
        this.valueFactory = Objects.requireNonNull(valueFactory);
    }
```

再回顾一下 proxyClassCache 是如何利用 WeakCache 创建的

```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

通过指定的两个参数来构建 一个指定分别指定  K P V 泛型的  WeakCache 类，这个指定类型 K P V 和两个构造参数贯穿了整个代理对像的创建流程。这两个参数一定要对应好：

**两个参数**

* **BiFunction<K, P, ?> subKeyFactory - > new KeyFactory()**
* **BiFunction<K, P, V> valueFactory -> new ProxyClassFactory()**

**K P V**

* **K key 值 ->ClassLoader ： 代理对象的类加载器** 
* **P 参数值 ->Class[]<?> : 代理对象实现的接口数组**
* **V value值 -> Class<?>：需要的代理类对象**

**构造函数**

创建 ProcyClassCache 变量对应的 WeakCache 构造函数的参数有两个，结合注释可以知道这两个参数的作用：

* **new KeyFactory()**：通过 key 和 p 生成 subKey
* **new ProxyClassFactory()** ：通过 key 和 p 生成 value

key 传进来的是ClassLoader进行包装后的对象，p 是代理对象实现的参数数组，最终生成的 value 就是我们需要的代理类对象

```java
private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
```

结合这个二级缓存可以推测：

* 通过构造函数传进来的 `keyFactory` 根据代理对象的类加载器和接口数据生成 subKey
* 根据 subKey 查找 Suppler 类，即代理对象，如果有则返回；否则根据 代理对象的ClassLoader 和 interfaces 生成代理类

前面说了，proxyClassCache 对象的 get 方法其实是 WeakCache 类的 get 方法，接下来看一下 WeakCache 对象的 get 方法。

### 5.`WeakCache` 对象的get方法

Proxy 类 下`getProxyClass0` 的方法最后  `return proxyClassCache.get(loader, interfaces);`

由前面的分析可以得到 其实这个 get 方法就是 WeakCache 的 get 方法

```java
 /**
  * Look-up the value through the cache. This always evaluates the
  * {@code subKeyFactory} function and optionally evaluates
  * {@code valueFactory} function if there is no entry in the cache for given pair of (key, subKey) or the entry has already been cleared.
  *
  * @param key       possibly null key
  * @param parameter parameter used together with key to create sub-key and
  *                  value (should not be null)
  * @return the cached value (never null)
  * @throws NullPointerException if {@code parameter} passed in or
  *                              {@code sub-key} calculated by
  *                              {@code subKeyFactory} or {@code value}
  *                              calculated by {@code valueFactory} is null.
  */
public V get(K key, P parameter) {
    // 检查p 不为空，这里的p 就是类接口数组
    Objects.requireNonNull(parameter);
	// 清除无效的缓存
    expungeStaleEntries();
	// cacheKey就是(key, sub-key) -> value里的一级key，
    Object cacheKey = CacheKey.valueOf(key, refQueue);

    // lazily install the 2nd level valuesMap for the particular cacheKey
    // 根据一级key得到 ConcurrentMap<Object, Supplier<V>>对象。如果之前不存在，则新建一个ConcurrentMap<Object, Supplier<V>>和cacheKey（一级key）一起放到map中。

    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // create subKey and retrieve the possible Supplier<V> stored by that
    // subKey from valuesMap
    // subKeyFactory.apply 方法生成 sub-key
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    // 根据二级索引的sub-key查到 supper ，其实就是 factory
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;

    while (true) {
        // 如果这个 二级map里面有 suppier，直接通过 get 方法获得代理对象返回
        if (supplier != null) {
            // supplier might be a Factory or a CacheValue<V> instance
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        // else no supplier in cache
        // or a supplier that returned null (could be a cleared CacheValue
        // or a Factory that wasn't successful in installing the CacheValue)
		
        // 若缓存中没有supplier，则创建一个Factory对象，把factory对象在多线程安全的环境下安全的赋值给supplier因为是while(true)当中，赋值成功后又get方法，返回才结束
        // lazily construct a Factory
        if (factory == null) {
            factory = new Factory(key, parameter, subKey, valuesMap);
        }

        if (supplier == null) {
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // successfully installed Factory
                supplier = factory;
            }
            // else retry with winning supplier
        } else {
            if (valuesMap.replace(subKey, supplier, factory)) {
                // successfully replaced
                // cleared CacheEntry / unsuccessful Factory
                // with our Factory
                supplier = factory;
            } else {
                // retry with current supplier
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```

看着有点乱，这段代码的核心逻辑：

此前的核心变量： 二级缓存 

`ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>>` 中的第一个 Object 就相当于一个一级 key，第二个Object 相当于 sub-key，Supplier<V> 相当于 value

```java
private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
```

1. **生成一级key** ：根据传入的key（代理对象的类加载器）和引用队列生成一级 key

```java
 Object cacheKey = CacheKey.valueOf(key, refQueue);
```

2. 根据一级key得到 `ConcurrentMap<Object, Supplier<V>>`对象。如果之前不存在，则新建一个`ConcurrentMap<Object, Supplier<V>>` 和 `cacheKey`（一级key）一起放到map中

```java
ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
```

3. **生成sub -key**： 通过传入的构造函数中`keyFactory`生成 sub-key，**通过 sub-key 得到 `Supplier<V>`**

```java
Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
Supplier<V> supplier = valuesMap.get(subKey);
```

生成 sub-key 的代码（了解）

```java
private static final class KeyFactory implements BiFunction<ClassLoader, Class<?>[], Object>
{
    @Override
    public Object apply(ClassLoader classLoader, Class<?>[] interfaces){
        switch (interfaces.length) {
            case 1: return new Key1(interfaces[0]); // the most frequent
            case 2: return new Key2(interfaces[0], interfaces[1]);
            case 0: return key0;
            default: return new KeyX(interfaces);
        }
    }
}
```

4. **如果`Suplier` 不为空，则直接调用 get 方法，得到代理对象**，返回结束。否则创建一个 Factory 对象，把factory对象在多线程的环境下安全的赋给supplier

```java
if (supplier != null) {
    // supplier might be a Factory or a CacheValue<V> instance
    V value = supplier.get();
	if (value != null) {
		return value;
	}
}
```

### 6. `Factory ` 类的 get 方法

```Java
        public synchronized V get() { // serialize access
            // re-check
            Supplier<V> supplier = valuesMap.get(subKey);
            //重新检查得到的supplier是不是当前对象
            if (supplier != this) {
                // something changed while we were waiting:
                // might be that we were replaced by a CacheValue
                // or were removed because of failure ->
                // return null to signal WeakCache.get() to retry
                // the loop
                return null;
            }
            // else still us (supplier == this)

            // create new value
            V value = null;
            try {
                 //代理类就是在这个位置调用valueFactory生成的
                 //valueFactory就是我们传入的 new ProxyClassFactory()
                
                value = Objects.requireNonNull(valueFactory.apply(key, parameter));
            } finally {
                if (value == null) { // remove us on failure
                    valuesMap.remove(subKey, this);
                }
            }
            // the only path to reach here is with non-null value
            assert value != null;

            // wrap value with CacheValue (WeakReference)
            //把value包装成弱引用
            CacheValue<V> cacheValue = new CacheValue<>(value);

            // put into reverseMap
            // reverseMap是用来实现缓存的有效性
            reverseMap.put(cacheValue, Boolean.TRUE);

            // try replacing us with CacheValue (this should always succeed)
            if (!valuesMap.replace(subKey, this, cacheValue)) {
                throw new AssertionError("Should not reach here");
            }

            // successfully replaced us with new CacheValue -> return the value
            // wrapped by it
            return value;
        }
    }
```

其实最终的代理对象就是这句代码生成的： `valueFactory` 就是我们最开始在构造参数里传入的 `new ProxyClassFactory()`

```java
value = Objects.requireNonNull(valueFactory.apply(key, parameter));
```

### 7 . `ProxyClassFactory` 的 apply 方法

```java
 //这里的BiFunction<T, U, R>是个函数式接口，可以理解为用T，U两种类型做参数，得到R类型的返回值
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        //所有代理类名字的前缀
        private static final String proxyClassNamePrefix = "$Proxy";
        
        // next number to use for generation of unique proxy class names
        //用于生成代理类名字的计数器
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
              
            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            //验证代理接口，可不看
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }
            //生成的代理类的包名 
            String proxyPkg = null;     // package to define proxy class in
            //代理类访问控制符: public ,final
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
            //验证所有非公共的接口在同一个包内；公共的就无需处理
            //生成包名和类名的逻辑，包名默认是com.sun.proxy，类名默认是$Proxy 加上一个自增的整数值
            //如果被代理类是 non-public proxy interface ，则用和被代理类接口一样的包名
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            //代理类的完全限定名，如com.sun.proxy.$Proxy0.calss
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            //核心部分，生成代理类的字节码
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                //把代理类加载到JVM中，至此动态代理过程基本结束了
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```

这里主要的代码逻辑：

1. 验证所有非公共的接口在同一个包内；公共的就无需处理。
2. 生成包名和类名的逻辑，包名默认是com.sun.proxy，类名默认是$Proxy 加上一个自增的整数值。如果被代理类是 non-public proxy interface ，则用和被代理类接口一样的包名

```java
/*
	* Choose a name for the proxy class to generate.
 */
long num = nextUniqueNumber.getAndIncrement();
//代理类的完全限定名，如com.sun.proxy.$Proxy0.calss
String proxyName = proxyPkg + proxyClassNamePrefix + num;
```

3. 生成类的字节码

```java
/*
 	* Generate the specified proxy class.
*/
    //核心部分，生成代理类的字节码
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
        proxyName, interfaces, accessFlags);
    try {
        //把代理类加载到JVM中，至此动态代理过程基本结束了
        return defineClass0(loader, proxyName,
                            proxyClassFile, 0, proxyClassFile.length);
```



## JDK 生成的动态代理字节码

```java
public static void main(String[] args) throws InvocationTargetException, IllegalAccessException {
        //注意一定要返回接口，不能返回实现类否则会报错
        Fruit fruit = (Fruit) DynamicAgent.agent(Fruit.class, new Apple());
        // 如果光将代理的内部类拿出来，那么当前类就为代理类，直接在main方法内返回代理对象
        // Friut fruit = Proxy.newProxyInstance(Reflect.class.getClassLoader(), new class[] {Friut}, new MyHandler(new ProxyPattern.Apple()));
        fruit.show();
        String path = "E:/$Proxy0.class";
        // 生成的代理字节码文件
        byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", Fruit.class.getInterfaces());
        FileOutputStream out = null;

        try {
            out = new FileOutputStream(path);
            out.write(classFile);
            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

字节码文件 .Class

```java 
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy {
    private static Method m1;
    private static Method m2;
    private static Method m0;

    // 构造函数，参数正是 InvocationHandler
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }
	
    // Object 的三个方法 equals， toString， hashCode
    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```

```java
return cons.newInstance(new Object[]{h});
```

最后的返回语句正是把 `InvocationHandler` 当做代理类的构造函数来初始化代理类，在字节码文件中也可以看到。

## 参考

https://www.jianshu.com/p/269afd0a52e6

https://www.cnblogs.com/puyangsky/p/6218925.html