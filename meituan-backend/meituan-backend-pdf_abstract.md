# meituan-backend-pdf_abstract
## 2018
### netty堆外内存泄漏（netty-socketio）  
1. 一次 Connect 和 Disconnect 为一次连接的建立与关闭  
2. 在 Disconnect事件前后申请的内存并没有释放(DIRECT_MEMORY_COUNTER堆外统计字段)  
3. 断点打在client.send() 这行， 然后关闭客户端连接，之后直接进入到这个方法，有个逻辑 encoder.allocateBuffer申请堆外内存  
4. handleWebsocket ：调用 encoder 分配了一段内存，调用完之后，我们的控制台立马就彪了 256B（怀疑肯定是这里申请的内存没有释放）  
5. encoder.encodePacket() 方法，把 packet 里面一个字段的值转换为一个 char（这里报NPE）  
6. 跟踪到NPE之前的代码，看看为啥没有赋值进来，给附上值 *解决*  

### 不可不说的Java“锁”事  
1. 乐观锁 VS 悲观锁(synchronized关键字和Lock的实现类都是悲观锁)  
2. 自旋锁 VS 适应性自旋锁(自旋锁的实现原理同样也是CAS，AtomicInteger中调用unsafe进行自增操作的源码中的do-while循环就是一个自旋操作)  
3. 无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁(Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息)  
4. 公平锁 VS 非公平锁(AQS AbstractQueuedSynchronizer,hasQueuedPredecessors()）  
5. 可重入锁 VS 非可重入锁(ReentrantLock和synchronized都是可重入锁，NonReentrantLock)  
6. 独享锁 VS 共享锁(JDK中的synchronized和JUC中Lock的实现类就是互斥锁。ReentrantReadWriteLock有两把锁：ReadLock和WriteLock，StampedLock 提供了一种乐观读锁的实现)  

## 2019
### Java Unsafe 
1. 提升程序 I/O 操作的性能。通常在 I/O 通信过程中，会存在堆内内存到堆外内存的数据拷贝操作，对于需要频繁进行内存间数据拷贝且生命周期较短的暂存数据，都建议存储到堆外内存。  
2. 创 建 DirectByteBuffer 的时候， 通过Unsafe.allocateMemory 分配内存、Unsafe.setMemory 进行内存初始化，而后构建 Cleaner 对象用于跟踪 DirectByteBuffer 对象的垃圾回收，以实现当 DirectByteBuffer 被垃圾回收时，分配的堆外内存一起被释放。（Cleaner 继承自 Java 四大引用类型之一的虚引用 PhantomReference（众所周知，无法通过虚引用获取与之关联的对象实例，且当对象仅被虚引用引用时，在任何发生 GC 的时候，其均可被回收），）  
3. 这部分，包括线程挂起、恢复、锁机制等方法。  
4. allocateInstance 在 java.lang.invoke、Objenesis（提供绕过类构造器的对象生成方式）、Gson（反序列化时用到）中都有相应的应用。  
5. 在 Java 8 中引入，用于定义内存屏障（也称内存栅栏，内存栅障，屏障指令等，是一类同步屏障指令，是 CPU 或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可  

### Java动态追踪技术 
1. java.lang.instrument.Instrumentation替换已经存在的 class 文件，运行时直接替换类很不安全。比如新的 class 文件引用了一个不存在的类，或者把某个类的一个 field 给删除了等等  
2. 因为有 BTrace 的存在，我们不必自己写一套ASM这样的工具了，BTrace 最终借 Instruments 实现 class 的替换  

### 字节码增强技术探索
1. 如果每次查看反编译后的字节码都使用 javap 命令的话，好非常繁琐。这里推荐一个 Idea 插件：jclasslib。  
2. 利用 Javassist 实现字节码增强时，可以无须关注字节码刻板的结构，其优点就在于编程简单。  
3. Attach API 的作用是提供 JVM 进程间通信的能力，比如说我们为了让另外一个 JVM 进程把线上服务的线程 Dump 出来，会运行 jstack 或 jmap 的进程，并传递 pid 的参数，告诉它要对哪个进程进行线程 Dump，这就是 Attach API 做的事情  
4. 热部署：不部署服务而对线上服务做修改，可以做打点、增加日志等操作，Mock：测试时候对某些服务做 Mock，性能诊断工具

### JVM Profiler技术原理和源码探索
1. Instrumentation 方式对几乎所有方法添加了额外的 AOP 逻辑，这会导致对线上服务造成巨额的性能影响，但其优势是：绝对精准的方法调用次数、调用时间统计。  
2. Sampling 方式基于无侵入的额外线程对所有线程的调用栈快照进行固定频率抽样，相对前者来说它的性能开销很低。（典型开源实现有 Async-Profiler 和 Honest-Profiler）  
3. FlameGraph 项目的核心只是一个 Perl 脚本  

### Java动态调试技术原理
1. Java-debug-tool 的同类产品主要是 greys，其他类似的工具大部分都是基于greys 进行的二次开发，所以直接选择 greys 来和 Java-debug-tool 进行对比。  

### 从ReentrantLOck的实现看AQS
1. 某个线程获取锁失败的后续流程是什么呢？存在某种排队等候机制，线程继续等待，仍然保留获取锁的可能，获取锁流程仍在继续。  
2. 如果处于排队等候机制中的线程一直无法获取锁，需要一直等待么？：线程所在节点的状态会变成取消状态，取消状态的节点会从队列中释放  
3. 

### springboot堆外内存排查
1. Native Code 所引起，而 Java 层面的工具不便于排查此类问题，只能使用系统层面的工具gperftools去定位问题  
2. 使用命令“strace -f -e”brk,mmap,munmap”-p pid”追踪向 OS 申请内存请求  
3. 想着看看内存中的情况使用命令 gdp -pid pid 进入 GDB 之后，然后使用命令 dump memory mem.bin startAddress endAddressdump 内存
其中 startAddress 和 endAddress 可以从 /proc/pid/smaps 中查找。然后使用 strings mem.bin 查看 dump 的内容  

## 2020
### 线程池实现原理
1. RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED  
2. 不用线程池的框架：Disruptor、Actor、协程  
3. 动态化线程池设计：简化线程池配置、参数可动态修改、增加线程池监控

### 美团亿万级KV存储
1. Squirrel 官方提供的方案，任何一个节点从宕机到被标记为 FAIL 摘除，一般需要经过 30 秒。更新 ZooKeeper，通知客户端、添加新node  
2. 持久化机制，写请求会先写到 DB 里，然后写到内存Backlog，这跟官方是一样的。同时它会把请求发给异步线程，异步线程负责把变更刷到硬盘的 Backlog 里。当硬盘 Backlog 过多时，我们会主动在业务低峰期做一次RDB ，然后把 RDB 之前生成的 Backlog 删除  
3. 如果有热点，监控服务会把热点 Key 所在 Slot 上报到我们的迁移服务。迁移服务这时会把热点主从节点加入到这个集群中，然后把热点 Slot 迁移到这个热点主从上。因为热点主从上只有热点 Slot 的请求，所以热点 Key 的处理能力得到了大幅提升  
4. Cellar 跟阿里开源的 Tair 主要有两个架构上的不同。第一个是 OB，第二个是 ZooKeeper  
5. Cellar 快慢列队，Cellar 智能迁移，Cellar 强一致  
6. 如果这个 Key 是一个热点，那么它会在做集群内复制的同时，还会把这个数据复制有热点区域的节点，同时，存储节点在返回结果给客户端时，会告诉客户端，这个 Key 是热点，这时客户端内会缓存这个热点 Key。  
7. 

### Java常见的9种CMS GC问题分析和解决
1. 分代收集器：ParNew：一款多线程的收集器，采用复制算法，主要工作在 Young 区；CMS：以获取最短回收停顿时间为目标，采用“标记 - 清除”算法
2. 分区收集器：G1：一种服务器端的垃圾收集器，应用在多处理器和大容量内存环境中；ZGC：JDK11 中推出的一款低延迟垃圾回收器，适用于大内存低延迟服务的内存管理和回收；
3. 读懂 GC Cause: System.gc()：手动触发 GC 操作；CMS：CMS GC 在执行过程中的一些动作；Promotion Failure：Old 区没有足够的空间分配给 Young 区；Concurrent Mode Failure：CMS GC 运行期间Old 区预留的空间不足；GCLocker Initiated GC：如果线程执行在 JNI 临界区时，刚好需要进行GC
4. MetaSpace 区 OOM: 经常会出问题的几个点有 Orika的 classMap、JSON 的 ASMSerializer、Groovy 动态加载类  
5. 过早晋升：分配速率接近于晋升速率，对象晋升年龄较小。原因：Young/Eden 区过小，分配速率过大  
6. CMS Old GC 频繁：判 断 当前 Old 区使用率是否大于阈值，则触发 CMS GC，默认为 92%。  
7. 内存泄漏：Dump Diff 和 Leak Suspects 比较直观就

### 堆外内存泄漏排查
```
-XX:NativeMemoryTracking=detail JVM 参数后重启项目
jcmd 272662 VM.native_memory detail
如果 total 中的 committed 和 top 中的 RES 相差不大，则应为主动申请的堆外内存
未释放造成的，如果相差较大，则基本可以确定是 JNI 调用造成的
```

## linux查看哪个进程占用磁盘IO  
$ iotop -oP  
命令的含义：只显示有I/O行为的进程  

$ pidstat -d 1  
命令的含义：展示I/O统计，每秒更新一次  

/Library/Java/JavaVirtualMachines/jdk-16.0.2.jdk/Contents/Home
/Library/Java/JavaVirtualMachines/jdk-15.0.2.jdk/Contents/Home
/Library/Java/JavaVirtualMachines/jdk-9.0.4.jdk/Contents/Home

PUT _all/_settings
{
"index.translog.durability" : "async",
"index.translog.flush_threshold_size" : "1024mb",
"index.translog.sync_interval" : "60s",
"index.refresh_interval" : "60s"
}

PUT _cluster/settings
{
  "transient" : {
    "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
  }
}


/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 172.16.1.15:9092,172.16.1.16:9092 --topic xueqiu-push-req --from-beginning --property print.key=true|grep 39171676469

/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.10.106.3:9092,10.10.106.4:9092 --topic usercenter_auth_sep --from-beginning --property print.key=true

/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.10.163.10:9092,10.10.163.11:9092 --topic xueqiu_push_user_auth_xy --from-beginning --property print.key=true

/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.10.55.2:9092,10.10.55.3:9092,10.10.56.2:9092,10.10.56.3:9092 --topic snowball_analysis_prod --offset 0  --partition 0 --property print.key=true |grep new_symbol  > partition0.txt
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.10.55.2:9092,10.10.55.3:9092,10.10.56.2:9092,10.10.56.3:9092 --topic snowball_analysis_prod --offset 0  --partition 1 --property print.key=true |grep new_symbol  > partition1.txt
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.10.55.2:9092,10.10.55.3:9092,10.10.56.2:9092,10.10.56.3:9092 --topic snowball_analysis_prod --offset 0  --partition 2 --property print.key=true |grep new_symbol  > partition2.txt
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.10.55.2:9092,10.10.55.3:9092,10.10.56.2:9092,10.10.56.3:9092 --topic snowball_analysis_prod --offset 0  --partition 3 --property print.key=true |grep new_symbol  > partition3.txt

/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.10.22.7:9092,10.10.22.8:9092,10.10.23.7:9092,10.10.23.8:9092 --group mirror-maker --describe

/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.10.55.7:9092,10.10.56.7:9092,10.10.58.7:9092 --group logging_logstash_ES --reset-offsets --all-topics --to-current --execute
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.10.55.7:9092,10.10.56.7:9092,10.10.58.7:9092 --group logging_logstash_ES --reset-offsets --topic logging_snowflake-usercenter_production --to-latest --execute

/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.10.55.2:9092 --describe --group status_release
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.10.55.2:9092,10.10.55.3:9092,10.10.56.2:9092,10.10.56.3:9092 --list
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 10.10.55.2:9092,10.10.55.3:9092,10.10.56.2:9092,10.10.56.3:9092 --delete --group rc.screener.option

/home/op/kafka_2.13-2.8.0/bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 10.10.22.7:9092 --topic stock_view_recently --time -1
/home/op/kafka_2.13-2.8.0/bin/kafka-consumer-offset-checker.sh --zookeeper 10.10.31.9:2181 --topic stock_view_recently  --group stock_follower
cat ./grpc.log |sed -n  '/2021-03-19 14:00:00.*/,/2021-03-19 14:10:00.*/p' |grep prePay | awk -F '|' '{if ($6>2000) print $6}'

java -Xms512M -Xmx512M -Xss1024K -XX:PermSize=256m -XX:MaxPermSize=512m  -cp KafkaOffsetMonitor-assembly-0.2.0.jar com.quantifind.kafka.offsetapp.OffsetGetterWeb \
--port 8088 \
--zk 10.10.31.9:2181,10.10.36.7:2181,10.10.37.7:2181 \
--refresh 5.minutes \
--retain 1.day >/dev/null 2>&1;

nohup /home/op/KafkaOffsetMonitor/kafka-monitor-start.sh &

jcmd 239312 GC.class_stats|awk '{print$13}'|sed 's/\(.*\)\.\(.*\)/\1/g'|sort |uniq -c|sort -nrk1

