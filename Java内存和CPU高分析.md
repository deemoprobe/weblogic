# Java内存和CPU高分析

## 垃圾回收机制（GC）

### GC机制作用

1. JVM自动检测和释放不再使用的对象内存
2. Java运行时JVM会执行GC, 不再需要显示释放对象,例如System.gc()

### Java 堆内存3代分布

![20201020163737](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201020163737.png)

- 年轻代: Young <Eden+From+To>
- 老年代: Old
- 持久代: Permanent

```shell
# 查看堆内存分布,3代都能看到
./jmap -heap pid
```

![20201021162413](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201021162413.png)

### GC

GC分类

1. Young GC (Minor GC)

    收集生命周期短的区域(Young)

2. Full GC (Major GC)

    收集生命周期短的区域(Young)和生命周期较长的区域(Old),对整个堆内存进行垃圾回收
    有时也会回收持久区(Perm)

GC过程

1. 新生成的对象在Eden区完成内存分配
2. 当Eden区满,再创建对象,会因为申请不到空间而出发YoungGC,进行Young区的垃圾回收
3. YGC时,Eden不能被回收的对象被放入到空的survivor,另一个survivor（From Survivor）里不能被GC回收的对象也会被放入To Survivor,始终保证一个survivor是空的
4. YGC结束后,若存放对象的survivor满,则这些对象被copy到old区,或者survivor区没有满,但是有些对象已经足够Old（超过XX:MaxTenuringThreshold）,也被放入Old区
5. 当Old区被放满的之后,进行完整的垃圾回收,即 FGC
6. FGC后,若Survivor及old区仍然无法存放从Eden复制过来的部分对象,导致JVM无法在Eden区为新对象创建内存区域,则出现OOM错误

堆内存:

存放由 new 创建的对象和数组,在堆中分配的内存,由 Java 虚拟机的自动垃圾回收器来管理

栈内存:

在函数中定义的一些基本类型的变量和对象的引用变量,都是在函数的栈内存中分配

## Java内存过高问题排查

### 内存过高的两种情况

- 内存溢出:程序分配的内存超过物理机内存大小,导致无法分配内存,出现OOM
- 内存泄漏:不再使用的对象一直占用内存不释放,长时间堆积造成OOM

### 检测和分析

1. 使用JVM自带工具(JDK里面)查看基本信息

    jstat [-options] pid [interval]  

    支持查询的options有如下:

    - class 查看类加载信息
    - compile 编译统计信息
    - gc 垃圾回收信息
    - gcXXX 各区域GC的详细信息,如gcold

    interval  每多少毫秒输出一次记录

    ```shell
    # 假设java进程pid是1234
    cd jdk/bin
    ./jstat -gc 1234
    ```

2. 导出dump文件

    ```shell
    # 如果需要进一步定位问题代码,打一下heap dump
    ./jmap -dump:live,format=b,file=<导出目录+文件名.hprof> <进程号>
    # 例如
    ./jmap -dump:live,format=b,file=/applog/java_dump_1234.hprof 1234
    ```

3. 分析dump

   使用IBM HeapAnalyzer或者Eclipse MAT分析

## Java进程CPU过高问题排查

### 分析

可以参考链接<https://www.cnblogs.com/wuchanming/p/7766994.html>

查看并处理CPU高的Java进程

```shell
# top命令查看
top
# 查看线程信息
top -Hp 1234
# 导出JVM线程信息(thread dump)
./jstack -l 1234 > /applog/1234.txt
# 将有问题的线程号转化为16进制(10进制转化为16进制)
echo "ibase=10; obase=16; 1234" | bc
or
printf "%x\n" 1234
# thread dump文件中 nid 代表线程号(小写), 十六进制前加 0x, 如0xfc10
# 去/applog/1234.txt文件里查到对应的十六进制线程号,分析问题
```

在dump中,线程一般存在如下几种状态:

1. RUNNABLE,线程处于执行中
2. BLOCKED,线程被阻塞
3. WAITING,线程正在等待

具体实例如图
![20201021161756](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20201021161756.png)

`PS:` 扩展一下bc进行进制转换

```shell
# 十六进制要大写,否则会有语法错误
# 二进制转为十进制
echo "ibase=2; 10101100" | bc
172
# 十进制转为二进制
echo "obase=2; 172" | bc
10101100
# 十进制转为十六进制
echo "obase=16; 172" | bc
AC
# 十六进制转为十进制
echo "ibase=16; AC" | bc
172
echo "ibase=16; obase=1010; AC" | bc
0172
echo "ibase=16; obase=101010; FC10" | bc
0064528 # 十进制对应的是64528
# ibase和obase默认是十进制,所以有时可省略不写
```
