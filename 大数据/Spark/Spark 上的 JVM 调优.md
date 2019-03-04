开启 GC 监控：

```
-Dcom.sun.management.jmxremote.port=<端口号> -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
```

然后可以通过 JDK 自带的 GUI 工具 `jconsole` 或者 `jstat` 实时监控程序。

以使用 jstat 查看 GC 为例：

```
jstat -gc <进程号>@<主机名>:<端口号>
```
