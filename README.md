H1. JDK17 Memroy Leak 问题排查

问题说明


1. 服务升级JDK1.7 LTS版本后, 内存会持续增长, 12G的规格, 一天约增长1%. 持续约1个月POD会OOMKiller
2. JVM堆配置如下 -Xms8G -Xmx8G -Xss256K -Dio.netty.maxDirectMemory=512000000
3. 堆+非堆内存实际使用不超过3GB, 并且一周监控无明显波动

问题排查一阶段

1. 猜测是Netty版本4.1.34,  DirectBuffer回收机制与JDK17不兼容, 有堆外内存泄露
2. 分析Netty堆外内存配置及回收部分源码, 发现如果 -Dio.netty.maxDirectMemory >0 是不触发主动回收
```
io.netty.util.internal.PlatformDependent

        // Here is how the system property is used:
        //
        // * <  0  - Don't use cleaner, and inherit max direct memory from java. In this case the
        //           "practical max direct memory" would be 2 * max memory as defined by the JDK.
        // * == 0  - Use cleaner, Netty will not enforce max memory, and instead will defer to JDK.
        // * >  0  - Don't use cleaner. This will limit Netty's total direct memory
        //           (note: that JDK's direct memory limit is independent of this).
        long maxDirectMemory = SystemPropertyUtil.getLong("io.netty.maxDirectMemory", -1);

```
3. 将JVM参数修改为 -Xms8G -Xmx8G -Xss256K -XX:MaxDirectMemorySize=512M -Dio.netty.maxDirectMemory=0. 重新测试i观察问题未解决
4. 关闭堆外内存使用, 亦未解决问题
```
        // We should always prefer direct buffers by default if we can use a Cleaner to release direct buffers.
        DIRECT_BUFFER_PREFERRED = CLEANER != NOOP
                                  && !SystemPropertyUtil.getBoolean("io.netty.noPreferDirect", false);
        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.noPreferDirect: {}", !DIRECT_BUFFER_PREFERRED);
        }
```


问题排查二阶段

1. JDK8后引入了NativeMemroyTracker工具, 那么通过JVM参数开启:  -Xms8G -Xmx8G -Xss256K -XX:MaxDirectMemorySize=512M -XX:NativeMemoryTracking=detail
2. 在POD重新拉起后, 使用jcmd工具定义内存统计基线, 并小时级别持续跟踪虚拟机各个模块malloc内存增长情况
```
// 设定内存跟踪基线
jcmd <pid> VM.native_memory baseline

// 查看虚拟机malloc内存增长diff
jcmd <pid> VM.native_memory detail.diff
```
3. 有持续增长的模块及代码块发现如下
```
-    Native Memory Tracking (reserved=12777KB +375KB, committed=12777KB +375KB)
                            (malloc=868KB +264KB #12911 +4154)
                            (tracking overhead=11908KB +110KB)

-        Shared class space (reserved=12288KB, committed=11964KB)
                            (mmap: reserved=12288KB, committed=11964KB)

-               Arena Chunk (reserved=236KB -38052KB, committed=236KB -38052KB)
                            (malloc=236KB -38052KB)

-                   Tracing (reserved=32KB, committed=32KB)
                            (arena=32KB #1)

-                   Logging (reserved=5KB, committed=5KB)
                            (malloc=5KB #219)

-                 Arguments (reserved=3KB, committed=3KB)
                            (malloc=3KB #105)

-                    Module (reserved=659KB, committed=659KB)
                            (malloc=659KB #3558)

-                 Safepoint (reserved=8KB, committed=8KB)
                            (mmap: reserved=8KB, committed=8KB)

-           Synchronization (reserved=294KB +2KB, committed=294KB +2KB)
                            (malloc=294KB +2KB #1951 +3)
             // 持续增长
-            Serviceability (reserved=16096KB +1948KB, committed=16096KB +1948KB)
                            (malloc=16096KB +1948KB #25681 +3156)

-                 Metaspace (reserved=98802KB, committed=97330KB)
                            (malloc=498KB #360)
                            (mmap: reserved=98304KB, committed=96832KB)
```

```
[0x00007fb566be7a3e] OopStorage::try_add_block()+0x2e
[0x00007fb566be80ad] OopStorage::allocate()+0x3d
[0x00007fb566dabc58] StackFrameInfo::StackFrameInfo(javaVFrame*, bool)+0x68
[0x00007fb566dac7d4] ThreadStackTrace::dump_stack_at_safepoint(int)+0xe4
                             (malloc=13533KB type=Serviceability +1684KB #21927 +2728)
```
4. 通过openjdk源码分析, 这部分是在dump线程堆栈, 所有线程进入safepoint的相关逻辑. 最终在openjdk的issue中发现了这个Bug

5. 触发的原因是CAT Client会每分钟 dump 所有线程的对账并上报, 从而触发jvm memory leak bug

问题修复

1. openjdk 17.0.2 / oraclejdk 17.0.2 均声明修复了该bug


参考链接

1. https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html
2. https://openjdk.java.net/groups/serviceability/
3. https://bugs.openjdk.java.net/browse/JDK-8273902
