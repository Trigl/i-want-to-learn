## 入门例子
ThreadLocal 类提供了 thread-local 类型变量，这种变量与普通的通过 `get` 或者 `set` 方法访问的变量相比，每个线程会对应一个独立的变量值，它是与线程相关的一个类的私有静态变量，例如一个用户 ID 或事务 ID 就会被设置成一个 ThreadLocal 类型的变量。

例如下面这个例子，通过 ThreadLocal 变量可以给每个线程指定 ID。

```java
public class ThreadId {
    // ID 生成器
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // 包含线程 ID 的 ThreadLocal 变量
    private static final ThreadLocal<Integer> threadId = ThreadLocal.withInitial(() -> nextId.getAndIncrement());
    // 另一种创建 ThreadLocal 实例的实现：new 一个匿名内部类，当然有了 lambda 表达式就不要用这种方式了
//    private static final ThreadLocal<Integer> threadId = new ThreadLocal<Integer>() {
//        @Override
//        protected Integer initialValue() {
//            return nextId.getAndIncrement();
//        }
//    };

    // 获取当前线程 ID
    public static int get() {
        return threadId.get();
    }

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(() -> {
                System.out.println("线程:" + Thread.currentThread().getName() + ", ID:" + ThreadId.get());
            });

            t.start();
        }
    }
}
```

为什么需要 ThreadLocal 这种线程私有的实例呢？

在 Java 中，避免并发问题最简单有效的方法就是不要引入并发，也就是多个线程之间不共享变量，也就不会存在并发问题。

当**线程存活**并且 `ThreadLocal` 实例可以被访问时，每个线程都会有一个该 ThreadLocal 变量引用，这些引用都不相同，可以看作是该 ThreadLocal 变量的多个副本。一旦线程不存在，这些复制体就被 GC 掉了。

## API
ThreadLocal 提供的方法如下：

![](/resource/public.png)

#### 公共方法
首先来看一下创建 ThreadLocal 对象的方法，API 提供了两种方法，一种是直接通过构造函数创建，另一种是通过 `withInitial` 方法，这里我们讲一下 `withInitial` 方法的具体实现，因为其使用到了 Java 8 的函数式编程接口。

###### `withIntial` 方法
`withIntial` 的源码如下：

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
```

可以看到这里的实现是通过给定一个 `Supplier` 类型的参数，new 了一个 `SuppliedThreadLocal` 的内部类，`Supplier` 是其构造函数的一个参数，所以关键是搞懂 `SuppliedThreadLocal`。

`SuppliedThreadLocal` 继承自 `ThreadLocal` 类，定义这样一个内部类的作用其实就是为了传入 Java8 的 Lambda 表达式，它的实现如下：

```java
/**
 * An extension of ThreadLocal that obtains its initial value from
 * the specified {@code Supplier}.
 */
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

这个内部类 override 了其父类 `ThreadLocal` 的 `initialValue` 方法，这个方法就是 `ThreadLocal` 实例拿到初始化值时必然会调用到的一个方法：

```java
protected T initialValue() {
    return null;
}
```

这个方法的用法如下：

1. 该方法返回当前 `ThreadLocal` 实例的初始化值
2. 当一个线程第一次访问 `ThreadLocal.get()` 方法式，该方法将被调用
3. 该方法在一种情况下不会被调用，那就是线程访问 `ThreadLocal.get()` 之前先调用了 `set` 方法
4. 该方法在大部分情况下最多会被调用一次，除了一种情况：线程先调用 `ThreadLocal.remove()`，然后调用 `ThreadLocal.get()`
5. 开发者如何使用这个方法呢？一种就是实现 `ThreadLocal` 的一个子类，子类里面重载这个方法；另一种就是 new 一个匿名内部类，直接得到其匿名内部类的实例，正如上面入门例子注释掉的部分。


回到 `SuppliedThreadLocal`，这里其实就是继承了 `ThreadLocal` 然后重载了 `initialValue()`。重载以后的值就是 `supplier.get()`，让我们看一下 `Supplier` 到底是个什么东东：

```java
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

这个接口加上了 `@FunctionalInterface` 注解，是一个典型的函数式编程接口，关于函数式编程接口可以专门再开一篇来讲了，这里可以通过 [函数式接口 Functional Interface](https://www.cnblogs.com/chenpi/p/5890144.html) 简单了解一下，总之，这里 `Supplier` 的作用其实就是一个我们输入 Lambda 函数式参数，然后它提供给我们需要的对象（T）。

注意在 `java.util.function` 包下面有很多类似于 `Supplier` 的类，可以帮助我们快乐的使用函数式编程接口：

|接口|参数类型|描述|
|-|-|-|
|Consumer|Consumer<T>|接收 T 对象，不返回值|
|Predicate|Predicate<T>|接收 T 对象并返回 boolean|
|Function|Function<T, R>|接收 T 对象，返回 R 对象|
|Supplier|Supplier<T>|提供 T 对象，不接收值|

## Refer
函数式接口 Functional Interface：https://www.cnblogs.com/chenpi/p/5890144.html

https://zhuanlan.zhihu.com/p/53698490

https://zhuanlan.zhihu.com/p/26713362

https://stackoverflow.com/questions/817856/when-and-how-should-i-use-a-threadlocal-variable

https://www.throwable.club/2019/02/17/java-currency-threadlocal/#%E9%BB%84%E9%87%91%E5%88%86%E5%89%B2%E6%95%B0%E7%9A%84%E5%BA%94%E7%94%A8
