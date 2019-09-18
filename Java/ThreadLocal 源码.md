ThreadLocal 类提供了 thread-local 类型变量，这种变量与普通的通过 `get` 或者 `set` 方法访问的变量相比，每个线程会对应一个独立的变量值，它是与线程相关的一个类的私有静态变量，例如一个用户 ID 或事务 ID 就会被设置成一个 ThreadLocal 类型的变量。

例如下面这个例子，通过 ThreadLocal 变量可以给每个线程指定 ID。

```java
public class ThreadId {
    // ID 生成器
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // 包含线程 ID 的 ThreadLocal 变量，使用 withInitial 方法进行初始化
    private static final ThreadLocal<Integer> threadId = ThreadLocal.withInitial(() -> nextId.getAndIncrement());

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


## Refer
https://zhuanlan.zhihu.com/p/53698490

https://zhuanlan.zhihu.com/p/26713362

https://stackoverflow.com/questions/817856/when-and-how-should-i-use-a-threadlocal-variable
