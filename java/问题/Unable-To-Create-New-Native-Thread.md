# Unable to Create New Native Thread问题

高并发场景下经常会出现 `java.lang.OutOfMemoryError`。在所有的场景中 `java.lang.OutOfMemoryError`: `unable to create new native thread` 是最常见的场景之一。当应用程序无法创建新线程时会生成这种类型。出现此错误，一般都是如下两个原因导致：

- 内存中没有空间容纳新线程。

- 线程数超过操作系统限制。

**两种类型的Out of Memory**

java.lang.OutOfMemoryError: Java heap space error
当JVM尝试在堆中分配对象，堆中空间不足时抛出。一般通过设定JAVA启动参数-Xmx最小可用内存解决。

java.lang.OutOfMemoryError: Unable to create new native thread
当JVM向OS申请创建线程，而OS不能分配一个本地线程时抛出。

## 无法创建native thread场景复现

搜索下日志，会发现海量日志系统中存在此类异常。

```go
java.lang.OutOfMemoryError: Unable to create new native thread .....
```

此异常并不会导致服务宕机，当次请求一定5xx。出现该问题一定会经过如下几个阶段：

![img](http://img.tianwangchong.com/blog/e8021f3b6179293f3eb855c372d4602820220722234041G3tCfc.png)

- 运行在 JVM 中的应用程序收到一个新的 Java 请求创建线程；

- JVM 系统会把创建新线程的请求转到操作系统；

- 操作系统尝试创建新线程，并为该线程分配内存；

- 如果已经超过操作系统的最大线程数限制，或者堆外内存不足，操作系统会拒绝创建线程，紧接着报 `java.lang.OutOfMemoryError: Unable to create new native thread error is thrown`。

通过如下代码可以验证自身系统可以创建的最大线程数量：

```java
public class TestThread extends Thread {  
    private static final AtomicInteger count = new AtomicInteger();  
  
    public static void main(String[] args) {  
        while (true)  
            (new TestThread()).start();  
  
    }  
  
    @Override  
    public void run() {  
        System.out.println(count.incrementAndGet());  
  
        while (true)  
            try {  
                Thread.sleep(Integer.MAX_VALUE);  
            } catch (InterruptedException e) {  
                break;  
            }  
    }  
} 
```

执行如下命令，到达线程创建上限后，则会抛出异常。

```shell
javac TestThread.java
java TestThread
```

## 解决方法

该类问题很难杜绝，除非你在上线之前做好万全的准备，根据自身经验说说，如何才能在一定程度上避免该问题的出现。

### 修改操作系统线程限制

操作系统可以创建的线程数存在限制。可以通过 `ulimit –u` 命令找到限制数。在某些服务器上，这个值设置较低，例如 1024。这意味着在这台机器上总共只能创建 1024 个线程。因此，如果您的应用程序正在创建超过 1024 个线程，它将遇到 `java.lang.OutOfMemoryError: unable to create new native thread.`在这种情况下，可以修改此限制。

```
查看线程限制数
ulimit -u

# 通过ulimit -u，可见用户默认的最大进程(含线程)数是1024
# 通过ulimit -s，可见进程默认的最大栈大小是10M
# ulimit -u 32000，更改用户默认的最大进程(含线程)数为32000

查看线程数
pstree -p | wc -l

查看用户当前创建的线程数【不准确吧】
ps hH p [pid] | wc -l

查看进程下的线程数
cat /proc/[pid]/status

查看服务器资源限制
ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 30631
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 65535
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 4096
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

open files (-n) 65535 是系统最多打开文件数量
max user processes (-u) 4096 是用户最多打开文件数量

大致的位置有这几个：/etc/security/limits.conf、/etc/security/limits.d/90-nproc.conf、/etc/security/limits.d/20-nproc.conf（我这边测试了，是在/etc/security/limits.conf，如果设置无效，其他2个位置也设置看看）
描述：
soft   xxx    : 代表警告的设定，可以超过这个设定值，但是超过后会有警告。
hard  xxx   : 代表严格的设定，不允许超过这个设定的值。
如：soft 设为1024，hard设为2048 ，则当你使用数在1~1024之间时可以随便使用，1024~2048时会出现警告信息，大于2048时，就会报错。
nproc  : 是操作系统级别对每个用户创建的进程数的限制
nofile  : 是每个进程可以打开的文件数的限制

配置如下(*代表所有用户，但是不包含root，root需要的话，就要单独配置)：
*          hard    nproc     800
*          soft    nproc     800
```



如果使用了K8s pod的话，那么需要修改容器 `pids-limit`限制，具体可以参考：`https://cloud.tencent.com/developer/article/1428964`, 可以调大，要进行评估，建议不要无限大，因为该物理机不一定只运行一个Java进程。

### 为机器分配更多的内存

线程不是在 JVM 堆中创建的。它们是在 JVM 堆之外创建的。所以如果 RAM 中剩余的空间较少，在 JVM 堆分配完成内存后，应用程序将遇到 `java.lang.OutOfMemoryError: unable to create new native thread.` 可能性更大。

例如：

整体内存大小：6 GB

堆大小（即 –Xms 和 –Xmx）：5 GB

Perm Gen 大小（即 -XX:MaxPermSize 和 -XX:MaxPermSize）：512 MB

根据此配置，JVM 堆使用 5.5 GB（即 5 GB 堆 + 512 MB Perm Gen）并且只留下 0.5 GB（即 6 GB – 5.5 GB）空间。注意这 0.5 GB 空间 - 内核进程、其他用户进程和线程必须运行。一般情况下Java线程大小配置为1Mb.如果您的应用程序有 500 个线程，那么仅线程就将占用 500mb 的空间。为了缓解这个问题，您可以考虑将堆大小从 5GB 减少到 4GB（如果您的应用程序可以容纳它而不会遇到其他内存瓶颈）；另外一种方式就是使用 java 系统属性 –Xss 来设置线程的内存大小。使用此属性，您可以减少内存大小。例如，如果您配置-Xss256k，您的线程将仅消耗 125mb 的空间。

堆外内存计算线程大小的公式:还没整理

如果使用k8s进行部署，一般会在编排文件层面限制容器内存或CPU大小，所以尽量不要使用 xms,xmx 参数，而要使用JVM内存参数新增了MaxRAMPercentage、InitialRAMPercentage、MinRAMPercentage，较灵活设定JVM 大小。例如：如上POD为1G内存，通用的启动脚本中指定80%（-XX:MaxRAMPercentage=80.0 -XX:InitialRAMPercentage=80.0 -XX:MinRAMPercentage=80.0）。那么服务就相当于设置了-Xmx819m -Xms819m。