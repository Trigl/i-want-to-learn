1、`$#`：传入的参数的个数。

2、`cut`：可以理解成剪切，例如：

```
TIME=`date  +"%Y%m%d%H" -d  "-1 hours"`
TIMEDAY=`echo $TIME|cut -c1-8`
echo $TIME
echo $TIMEDAY
```
