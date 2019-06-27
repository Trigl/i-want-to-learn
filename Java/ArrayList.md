ArrayList 可以自动调节长度，每一个 ArrayList 有一个 capacity 属性，用来表示该 ArrayList 可以存储的元素的长度，它的值大于等于 ArrayList 的长度。

当有新的元素添加到 ArrayList 的时候，capacity 就会动态增加，动态增加会带来一定的开销。所以如果可以确定需要给 ArrayList 里面添加大量元素，可以使用 `ensureCapacity` 给定一个初始值，避免多次动态增加大小。但是注意 `ensureCapacity` 方法是 ArrayList 的方法，List 接口是没有这个方法的，所以想要使用这个方法的话使用下面方式 new 一个 ArrayList：

```java
private ArrayList<String> list = new ArrayList<>();
```

ArrayList 是非线程安全的。

```java
package com.bilibili.lancer.gateway.bootstrap;

import java.util.*;

public class ConcurrentModificationExceptionTest {
    private static List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3));
    static List l = Collections.synchronizedList(new ArrayList<>());

    private static class Thread1 extends Thread {
        @Override
        public void run() {
            l.add(1);
            Iterator<Integer> iterator = list.iterator();
            while (iterator.hasNext()) {
                System.out.println(iterator.next());

                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private static class Thread2 extends Thread {
        @Override
        public void run() {
            for (int i = 4; i < 10; i++) {
                list.add(i);
            }
        }
    }

    public static void main(String[] args) {
        Thread1 t1 = new Thread1();
        Thread2 t2 = new Thread2();
        t1.start();
        t2.start();
    }
}
```
