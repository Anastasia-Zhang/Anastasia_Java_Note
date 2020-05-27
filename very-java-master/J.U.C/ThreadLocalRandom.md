### 前言

学习 `ThreadLocalRandom` 的时候遇到一些疑惑，为何使用它在多线程下会产生相同的随机数？

阅读源码后终于稍微了解了一些它的运行机制，总结出它在多线程下正确的用法，特此记录。

### ThreadLocalRandom的用处

在多线程下，使用 `java.util.Random` 产生的实例来产生随机数是线程安全的，但深挖 `Random` 的实现过程，会发现多个线程会竞争同一 `seed` 而造成性能降低。

其原因在于：

`Random` 生成新的随机数需要两步：

- 根据老的 `seed` 生成新的 `seed`
- 由新的 `seed` 计算出新的随机数

其中，第二步的算法是固定的，如果每个线程并发地获取同样的 `seed`，那么得到的随机数也是一样的。为了避免这种情况，`Random` 使用 CAS 操作保证每次只有一个线程可以获取并更新 seed，失败的线程则需要自旋重试。

因此，在多线程下用 `Random` 不太合适，为了解决这个问题，出现了 `ThreadLocalRandom`，在多线程下，它为每个线程维护一个 `seed` 变量，这样就不用竞争了。

但是我在使用的时候，发现 `ThreadLocalRandom` 在多线程下产生了相同的随机数，这是怎么回事呢？

### ThreadLocalRandom多线程下产生相同随机数

来看一下产生相同随机数的示例代码：



```java
import java.util.concurrent.ThreadLocalRandom;

public class ThreadLocalRandomDemo {

    private static final ThreadLocalRandom RANDOM =
            ThreadLocalRandom.current();

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Player().start();
        }
    }

    private static class Player extends Thread {
        @Override
        public void run() {
            System.out.println(getName() + ": " + RANDOM.nextInt(100));
        }
    }
}
```

运行该代码，结果如下：



```undefined
Thread-0: 4
Thread-1: 4
Thread-2: 4
Thread-3: 4
Thread-4: 4
Thread-5: 4
Thread-6: 4
Thread-7: 4
Thread-8: 4
Thread-9: 4
```

为此，我阅读了 `ThreadLocalRandom` 的源码，从中找到了端倪。

先是静态 current() 方法：



```cpp
public static ThreadLocalRandom current() {
    //如果线程第一次调用 current() 方法，执行 localInit()方法
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```

初始化方法 `localInit()`中，为线程初始化了 `seed`，并保存在 `UNSAFE` 里，这里 `UNSAFE` 的方法是 `native` 方法，我不太了解，但并不影响理解。可以把这里的操作看作是初始化了 `seed`，把线程和 `seed` 以键值对的形式保存起来。



```java
static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}
```

当要生成随机数的时候，调用 `nextInt()` 方法：



```cpp
public int nextInt(int bound) {
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);
    //第一处
    int r = mix32(nextSeed());
    int m = bound - 1;
    if ((bound & m) == 0) // power of two
        r &= m;
    else { // reject over-represented candidates
        for (int u = r >>> 1;
             u + m - (r = u % bound) < 0;
             u = mix32(nextSeed()) >>> 1)
            ;
    }
    return r;
}
```

这里主要关注 `第一处` 的 `nextSeed()` 方法：



```java
final long nextSeed() {
    Thread t; long r; // read and update per-thread seed
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                   r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
}
```

好了，问题来了！这里返回的值是 `r = UNSAFE.getLong(t, SEED) + GAMMA`，是从 `UNSAFE` 里取出来的。但问题是，这里取出来的值对不对？或者说，能否取出来？

回到示例代码，我们在主线程调用了 `TreadLocalRandom` 的 `current()` 方法，该方法把主线程和主线程的 `seed` 存入了 `UNSAFE`。

接下来，我们在非主线程调用 `nextInt()`，但非主线程和 `seed` 的键值对之前并没有存入 `UNSAFE` 。但我们却从 `UNSAFE` 里取非主线程的 `seed` 值，虽然我不知道取出来的 `seed` 到底是什么，但肯定不是多线程下想要的结果，而这也导致了多线程下产生的随机数是重复的。

那么在多线程下如何正确地使用 `ThreadLocalRandom` 呢？

### ThreadLocalRandom多线程下正确用法

结合上述分析，正确地使用 `ThreadLocalRandom`，肯定需要给每个线程初始化一个 `seed`，那就需要调用 `ThreadLocalRandom.current()` 方法。

那么有个疑问，在每个线程里都调用 `ThreadLocalRandom.current()`，会产生多个 `ThreadLocalRandom` 实例吗？

不会的，见源码：



```php
/** The common ThreadLocalRandom */
static final ThreadLocalRandom instance = new ThreadLocalRandom();

/**
 * Returns the current thread's {@code ThreadLocalRandom}.
 *
 * @return the current thread's {@code ThreadLocalRandom}
 */
public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```

放心大胆地使用。

于是示例代码改动如下：



```java
import java.util.concurrent.ThreadLocalRandom;

public class ThreadLocalRandomDemo {

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Player().start();
        }
    }

    private static class Player extends Thread {
        @Override
        public void run() {
            System.out.println(getName() + ": " + ThreadLocalRandom.current().nextInt(100));
        }
    }
}
```

运行一下，可以得到想要的结果：



```undefined
Thread-0: 90
Thread-3: 77
Thread-2: 97
Thread-5: 96
Thread-4: 42
Thread-1: 3
Thread-6: 4
Thread-7: 6
Thread-8: 52
Thread-9: 39
```

总结一下，在多线程下使用 `ThreadLocalRandom` 产生随机数时，直接使用 `ThreadLocalRandom.current().xxxx`



作者：Murray66
链接：https://www.jianshu.com/p/89dfe990295c
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。