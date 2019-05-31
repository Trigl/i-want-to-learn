通过如下命令列出可以运行时修改的 JVM 参数：

```
java -XX:+PrintFlagsFinal -version | grep manageable
```

然后使用 `jcmd` 获取到进程的 pid。

接着就可以使用 jinfo 命令来动态修改 JVM 虚拟机的参数了，例如在 GC 前后 dump 出堆对象信息：

```
jinfo -flag HeapDumpPath=/dump/path pid
jinfo -flag +HeapDumpBeforeFullGC pid
jinfo -flag +HeapDumpAfterFullGC pid
```

上面的 `+` 就是设置对应的参数为 true，如果你不确定当前这个参数的值，同样可以用 jinfo 命令查看一下其当前值：

```
jinfo -flag HeapDumpBeforeFullGC pid
```

也就是把 `+` 去掉。

可以写一个小程序测试一下：

```java
private static void s2() {
       String name = ManagementFactory.getRuntimeMXBean().getName();
       // get pid
       String pid = name.split("@")[0];
       System.out.println("Pid is:" + pid);

       while(true)
       {
           byte[] b = null;
           for (int i = 0; i < 10; i++)
               b = new byte[1 * 1024 * 1024];

           try {
               Thread.sleep(5000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
   }
```

启动参数设置为：-Xmx20m -Xms20m -Xmn2m
