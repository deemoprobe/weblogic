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
3. YGC时

## Java内存过高问题排查

### 内存过高的两种情况

- 内存溢出：程序分配的内存超过物理机内存大小，导致无法分配内存，出现OOM
- 内存泄漏：不再使用的对象一直占用内存不释放，长时间堆积造成OOM

### 检测和分析

1. 使用JVM自带工具(JDK里面)查看基本信息

    jstat [-options] pid [interval]  

    支持查询的options有如下：

    - class 查看类加载信息
    - compile 编译统计信息
    - gc 垃圾回收信息
    - gcXXX 各区域GC的详细信息，如gcold

    interval  每多少毫秒输出一次记录

    ```shell
    # 假设java进程pid是1234
    cd jdk/bin
    ./jstat -gc 1234
    ```

2. 导出dump文件

    ```shell
    # 如果需要进一步定位问题代码，那么就需要把Java程序的内存镜像导出
    ./jmap -dump:live,format=b,file=<导出目录+文件名.hprof> <进程号>
    # 例如
    ./jmap -dump:live,format=b,file=/applog/java_dump_1234.hprof 1234
    ```

3. 分析dump

   使用IBM HeapAnalyzer或者Eclipse MAT分析

## Java进程CPU过高问题排查

### 分析

查看并处理CPU高的Java进程

```shell
# top命令查看
top
# 查看线程信息
top -Hp 1234
# 导出JVM堆栈信息
./jstack -l 1234 > /applog/1234.hprof
# 将有问题的线程号转化为16进制(10进制转化为16进制)
echo "ibase=10; obase=16; 1234" | bc
or
printf "%x\n" 1234
```
