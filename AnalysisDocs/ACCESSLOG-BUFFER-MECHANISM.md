# Tomcat 访问日志缓冲机制详解

## 概述

Tomcat 访问日志使用**双层缓冲架构**来优化性能：内存中的 `BufferedWriter` 和周期性的后台刷新线程。这种设计在**吞吐量**和**数据可靠性**之间取得了平衡。

## 缓冲架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         访问日志缓冲架构                                      │
└─────────────────────────────────────────────────────────────────────────────┘

应用程序请求              访问日志线程
    │                        │
    ▼                        ▼
┌────────────────┐   ┌─────────────────────────────────────────┐
│  请求处理完成   │   │      ContainerBackgroundProcessor        │
│  logAccess()   │──▶│         (后台线程，默认 10 秒间隔)          │
└────────────────┘   │                                         │
                     │  每 backgroundProcessorDelay 秒调用一次  │
                     └──────────────┬──────────────────────────┘
                                    │
                                    ▼
                     ┌─────────────────────────────────────────┐
                     │       AccessLogValve.backgroundProcess()│
                     │                                         │
                     │  1. writer.flush() - 刷新缓冲区到磁盘    │
                     │  2. 清理过期日志文件                      │
                     └──────────────┬──────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         数据写入流程                                       │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Request N                                                        Request │
│     │                                                                N+1 │
│     ▼                                                                  ▼  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│  │ CharArrayWriter │───▶│ synchronized {  │───▶│  BufferedWriter  │       │
│  │  (对象池复用)    │    │   writeTo(writer)│    │   (128KB 缓冲)   │       │
│  └─────────────────┘    └─────────────────┘    │   ┌───────────┐  │       │
│                                              │   │  内部缓冲  │  │       │
│                                              │   │  char[]    │  │       │
│                                              │   └───────────┘  │       │
│                                              └─────────┬─────────┘       │
│                                                        │                 │
│                           buffered=true               │ buffered=false   │
│                           (默认)                      │                 │
│                                                        │                 │
│                           ┌───────────────────┐       │                 │
│                           │ 不立即 flush      │       │                 │
│                           │ 等待后台线程刷新  │       │                 │
│                           └───────────────────┘       ▼                 │
│                                              ┌─────────────┐           │
│                                              │ 立即 flush   │           │
│                                              └─────────────┘           │
│                                                        │                 │
│                                                        ▼                 │
│                                              ┌─────────────────┐         │
│                                              │ OutputStreamWriter│      │
│                                              │   FileOutputStream│      │
│                                              └─────────┬───────────┘     │
│                                                        │                 │
│                                                        ▼                 │
│                                              ┌─────────────────┐         │
│                                              │   磁盘文件       │         │
│                                              │ access_log.xxx  │         │
│                                              └─────────────────┘         │
└───────────────────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. 缓冲配置

```java
// AccessLogValve.java:103
private boolean buffered = true;  // 默认启用缓冲
```

| 配置值 | 行为 | 性能影响 | 可靠性 |
|--------|------|---------|--------|
| `true` (默认) | 数据写入内存缓冲区，等待后台线程刷新 | 高吞吐量 | 有丢失风险（进程崩溃） |
| `false` | 每条日志立即刷新到磁盘 | 低吞吐量 | 高可靠性 |

### 2. BufferedWriter 层

```java
// AccessLogValve.java:620-622
writer = new PrintWriter(
    new BufferedWriter(
        new OutputStreamWriter(
            new FileOutputStream(pathname, true),  // append=true
            charset
        ),
        128000  // 128KB 缓冲区大小
    ),
    false
);
```

**缓冲区大小分析：**

| 缓冲区大小 | 适用场景 | 优缺点 |
|-----------|---------|--------|
| 8KB (默认) | 低流量 | 内存占用小，但系统调用频繁 |
| **128KB** | 中高流量 | **平衡选择**，Tomcat 默认值 |
| 1MB+ | 极高流量 | 系统调用最少，但内存占用高 |

**128KB 容量计算：**
```
假设平均日志行长度 = 200 字节
128000 / 200 = 640 条日志

10000 QPS 场景下：
640 / 10000 = 0.064 秒 ≈ 64 毫秒即可填满缓冲区
```

### 3. 后台刷新线程

#### 调度机制

```java
// ContainerBase.java:136-142
protected int backgroundProcessorDelay = -1;  // 默认禁用
protected ScheduledFuture<?> backgroundProcessorFuture;

// 启动后台线程
if (backgroundProcessorDelay > 0) {
    monitorFuture = Container.getService(ContainerBase.this)
        .getServer()
        .getUtilityExecutor()
        .scheduleWithFixedDelay(
            new ContainerBackgroundProcessorMonitor(),
            0, 60, TimeUnit.SECONDS
        );
}
```

#### 执行流程

```java
// ContainerBase.java:907-939
@Override
public synchronized void backgroundProcess() {
    if (!getState().isAvailable()) {
        return;
    }

    // 1. 处理 Cluster
    Cluster cluster = getClusterInternal();
    if (cluster != null) {
        cluster.backgroundProcess();
    }

    // 2. 处理 Realm
    Realm realm = getRealmInternal();
    if (realm != null) {
        realm.backgroundProcess();
    }

    // 3. 处理所有 Valve (包括 AccessLogValve)
    Valve current = pipeline.getFirst();
    while (current != null) {
        current.backgroundProcess();  // 调用 AccessLogValve.backgroundProcess()
        current = current.getNext();
    }

    // 4. 触发生命周期事件
    fireLifecycleEvent(PERIODIC_EVENT, null);
}
```

#### AccessLogValve 后台处理

```java
// AccessLogValve.java:358-404
@Override
public synchronized void backgroundProcess() {
    // ====== 第一部分：刷新缓冲区 ======
    if (getState().isAvailable() && getEnabled() && writer != null && buffered) {
        writer.flush();  // 将 BufferedWriter 的内容刷新到磁盘
    }

    // ====== 第二部分：清理过期日志 ======
    int maxDays = this.maxDays;
    String prefix = this.prefix;
    String suffix = this.suffix;

    if (rotatable && checkForOldLogs && maxDays > 0) {
        long deleteIfLastModifiedBefore =
            System.currentTimeMillis() - (maxDays * 24L * 60 * 60 * 1000);
        File dir = getDirectoryFile();
        if (dir.isDirectory()) {
            String[] oldAccessLogs = dir.list();
            if (oldAccessLogs != null) {
                for (String oldAccessLog : oldAccessLogs) {
                    // ... 删除逻辑 ...
                }
            }
        }
        checkForOldLogs = false;
    }
}
```

### 4. 日志写入同步

```java
// AccessLogValve.java:559-596
public void log(CharArrayWriter message) {
    rotate();

    // 外部程序移动文件的处理
    if (checkExists) {
        synchronized (this) {
            if (currentLogFile != null && !currentLogFile.exists()) {
                close(false);
                open();
            }
        }
    }

    // 写入日志
    try {
        message.write(System.lineSeparator());
        synchronized (this) {
            if (writer != null) {
                message.writeTo(writer);  // 写入到 BufferedWriter

                // ====== 关键分支：缓冲 vs 非缓冲 ======
                if (!buffered) {
                    writer.flush();  // 非缓冲模式：立即刷新
                }
                // 缓冲模式：不刷新，等待后台线程
            }
        }
    } catch (IOException ioe) {
        log.warn(sm.getString("accessLogValve.writeFail", message.toString()), ioe);
    }
}
```

## 缓冲 vs 非缓冲模式对比

### 模式对比表

| 特性 | 缓冲模式 (`buffered=true`) | 非缓冲模式 (`buffered=false`) |
|------|---------------------------|------------------------------|
| **数据路径** | 应用 → 内存缓冲 → 后台线程 → 磁盘 | 应用 → 直接写入磁盘 |
| **刷新时机** | 周期性 (默认 10 秒) 或 缓冲区满 | 每条日志后立即 |
| **系统调用频率** | 低 (每 10 秒或每 640 条) | 高 (每条日志一次) |
| **CPU 开销** | 低 (批量写入) | 高 (频繁上下文切换) |
| **响应延迟** | 低 (不阻塞请求) | 高 (每次 flush 阻塞) |
| **数据丢失风险** | 有 (进程崩溃丢失缓冲区数据) | 无 |
| **磁盘 I/O 模式** | 批量顺序写入 | 频繁小量写入 |
| **典型 QPS** | 10,000+ | < 5,000 |

### 时序对比图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        缓冲模式时序图                                        │
└─────────────────────────────────────────────────────────────────────────────┘

时间轴 ────────────────────────────────────────────────────────────────────────▶

请求:  R1   R2   R3   ...   R100   R101   ...   R640   R641   ...
      │    │    │         │      │             │      │
      ▼    ▼    ▼         ▼      ▼             ▼      ▼
写入: W1   W2   W3   ...   W100   W101   ...   W640   [缓冲区满触发flush]
      │    │    │         │      │             │      │
      └────┴────┴─────────┴──────┴─────────────┴──────┴──▶ BufferedWriter(128KB)
                                                                     │
                                                                     ▼
                                                             Flush 到磁盘
                                                                  │
后台:                                    ┌─────────────────────────┴─────────┐
线程:                                    │  backgroundProcess() 每 10 秒   │
                                        │  writer.flush()                 │
                                        └──────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                       非缓冲模式时序图                                       │
└─────────────────────────────────────────────────────────────────────────────┘

时间轴 ────────────────────────────────────────────────────────────────────────▶

请求:  R1   R2   R3   ...   R100   ...
      │    │    │         │      │
      ▼    ▼    ▼         ▼      ▼
写入: W1   W2   W3   ...   W100   ...
      │    │    │         │      │
      ▼    ▼    ▼         ▼      ▼
Flush: F1   F2   F3   ...   F100   ...
      │    │    │         │      │
      ▼    ▼    ▼         ▼      ▼
磁盘: D1   D2   D3   ...   D100   ...  (每次都阻塞)
```

## 刷新策略详解

### 1. 自动刷新触发条件

| 条件 | 描述 | 代码位置 |
|------|------|---------|
| **后台周期** | 每 `backgroundProcessorDelay` 秒 | `backgroundProcess()` |
| **缓冲区满** | BufferedWriter 内部缓冲区满 (8KB) | Java 内部自动 |
| **日志轮转** | 日期变更，关闭旧文件时 | `close(true)` |
| **手动触发** | JMX 调用 `rotate()` | `rotate()` 方法 |

### 2. BufferedWriter 内部刷新

Java 的 `BufferedWriter` 在以下情况下自动刷新底层流：

```java
// Java 内部逻辑 (简化)
public void write(int c) throws IOException {
    synchronized (lock) {
        if (nextChar >= nChars)  // 缓冲区满
            flushBuffer();
        cb[nextChar++] = (char) c;
    }
}

private void flushBuffer() throws IOException {
    out.write(cb, 0, nextChar);  // 写入到底层流
    nextChar = 0;
}
```

**关键点：** `BufferedWriter` 的内部缓冲区 (8KB) 是 Java 默认的，与 Tomcat 配置的 128KB **不是同一个**！

### 3. 双重缓冲结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         双重缓冲结构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                         PrintWriter                                   │ │
│  │                         (无缓冲，直接委托)                             │ │
│  └───────────────────────────────────┬───────────────────────────────────┘ │
│                                      ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    BufferedWriter                                     │ │
│  │                    内部缓冲区：8KB (Java 默认)                        │ │
│  │                                                                      │ │
│  │  ┌────────────────────────────────────────────────────────────────┐  │ │
│  │  │                     char[] cb (8192 字节)                      │  │ │
│  │  │  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────────┐       │  │ │
│  │  │  │ 日 │ 志 │ 行 │ 1  │ 日 │ 志 │ 行 │ 2  │ ... │   空    │       │  │ │
│  │  │  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────────┘       │  │ │
│  │  │  ↑                                                      ↑       │  │ │
│  │  │  └──────────── 已写入 (2048 字节) ──────────────────┘       │  │ │
│  │  │                                                           │  │ │
│  │  │  满 8KB 时自动 flush 到 OutputStreamWriter                   │  │ │
│  │  └────────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────┬───────────────────────────────────┘ │
│                                      │                                     │
│                                      ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    OutputStreamWriter                                 │ │
│  │                    字符 → 字节转换                                      │ │
│  └───────────────────────────────────┬───────────────────────────────────┘ │
│                                      │                                     │
│                                      ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    FileOutputStream                                   │ │
│  │                    写入到磁盘文件                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  注：128KB 配置在 BufferedWriter 构造函数中，但实际上 BufferedWriter 的      │
│      内部缓冲区仍然是 8KB，128KB 只是给 FileOutputStream 的提示。            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 性能特征分析

### 1. 写入延迟分析

**缓冲模式：**
```
单条日志写入延迟 = CharArrayWriter 写入 + synchronized 锁获取
                ≈ 0.1 - 1 微秒

刷新延迟（后台线程） = 最多 128KB 数据的写入时间
                     ≈ 128KB / 磁盘写入速度
                     ≈ 128KB / 100MB/s ≈ 1.3 毫秒
```

**非缓冲模式：**
```
单条日志写入延迟 = CharArrayWriter 写入 + synchronized 锁获取 + writer.flush()
                = 0.1μs + 1μs + (磁盘寻道 + 写入)
                ≈ 1 - 10 毫秒
```

### 2. 吞吐量分析

```
理论吞吐量 = 1 / 单条日志延迟

缓冲模式:  1 / 1μs = 1,000,000 条/秒
非缓冲模式: 1 / 5ms = 200 条/秒

实际吞吐量受限于：
- 磁盘 I/O 带宽
- CPU 处理能力
- 网络带宽
- synchronized 锁竞争
```

### 3. 内存占用

```
缓冲模式内存占用 = BufferedWriter (8KB) + CharArrayWriter 对象池
非缓冲模式内存占用 = CharArrayWriter 对象池

CharArrayWriter 对象池大小 ≈ 线程数 × 128 字节 × 池大小
```

## 配置建议

### 生产环境推荐配置

```xml
<!-- 标准高吞吐场景 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="access."
       suffix=".log"
       pattern="combined"
       buffered="true"              <!-- 启用缓冲 -->
       maxDays="30"/>

<!-- 极端高吞吐场景 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="/ssd/tomcat-logs"
       prefix="access."
       suffix=".log"
       pattern="%h %t %r %s %b %D"    <!-- 简化模式 -->
       buffered="true"
       maxDays="7"
       checkExists="false"/>         <!-- 禁用文件检查 -->

<!-- 金融/审计场景（必须保证日志完整性） -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="access."
       suffix=".log"
       pattern="combined"
       buffered="false"             <!-- 禁用缓冲 -->
       checkExists="true"/>         <!-- 支持外部日志轮转 -->
```

### 后台线程间隔配置

```xml
<!-- 在 Engine/Host/Context 级别配置 -->
<Engine backgroundProcessorDelay="10">  <!-- 10 秒间隔，默认值 -->
    ...
</Engine>

<!-- 调整建议 -->
backgroundProcessorDelay="5"   <!-- 更频繁刷新，数据丢失窗口更小 -->
backgroundProcessorDelay="30"  <!-- 更少刷新，减少 I/O 开销 -->
```

## 故障场景分析

### 场景 1：进程崩溃

```
缓冲模式：
├── 崩溃时缓冲区数据丢失
├── 最多丢失 10 秒 × QPS 条日志
└── 10000 QPS 场景下最多丢失 100,000 条

非缓冲模式：
├── 每条日志立即写入磁盘
└── 崩溃时不丢失已写入的日志
```

### 场景 2：磁盘满

```
两种模式的行为：

缓冲模式：
├── BufferedWriter 继续接受数据
├── 后台 flush 时抛出 IOException
└── 日志文件大小 = 最后成功 flush 的位置

非缓冲模式：
├── 每次写入立即抛出 IOException
├── 影响请求处理性能
└── 日志记录失败但请求继续处理
```

### 场景 3：高并发锁竞争

```
非缓冲模式下的问题：

Thread-1: synchronized → write → flush → 释放锁 (5ms)
Thread-2:                        等待 → synchronized → write → flush → 释放锁 (5ms)
Thread-3:                                          等待 → ...

结果：串行化，QPS 下降到 200 左右

缓冲模式的改进：

Thread-1: synchronized → write → 释放锁 (1μs)
Thread-2:              等待 → synchronized → write → 释放锁 (1μs)
Thread-3:                       等待 → synchronized → write → 释放锁 (1μs)

后台线程: 定期 flush (不影响请求线程)
```

## 监控指标

### 关键监控点

| 指标 | 获取方式 | 告警阈值 |
|------|---------|---------|
| 缓冲区刷新频率 | JMX `AccessLogValve@FlushCount` | 异常下降 |
| 后台线程延迟 | 日志时间戳差异 | > 15 秒 |
| 日志文件大小增长 | `ls -l` | 过快/过慢 |
| I/O 等待时间 | `iostat` | > 20% |

### 监控脚本

```bash
#!/bin/bash
# 访问日志缓冲监控脚本

LOG_DIR="/opt/tomcat/logs"
ALERT_FILE="/tmp/access_log_alert.log"

while true; do
    # 检查日志更新时间
    ACCESS_LOG=$(ls -t $LOG_DIR/access*.log 2>/dev/null | head -1)
    if [ -n "$ACCESS_LOG" ]; then
        LAST_UPDATE=$(stat -c %Y "$ACCESS_LOG" 2>/dev/null || stat -f %m "$ACCESS_LOG")
        CURRENT_TIME=$(date +%s)
        DIFF=$((CURRENT_TIME - LAST_UPDATE))

        if [ $DIFF -gt 15 ]; then
            echo "[$(date)] 警告: 访问日志超过 ${DIFF} 秒未更新" >> $ALERT_FILE
        fi
    fi

    # 检查日志文件大小（确保在增长）
    PREV_SIZE=${CURR_SIZE:-0}
    CURR_SIZE=$(stat -c %s "$ACCESS_LOG" 2>/dev/null || stat -f %z "$ACCESS_LOG")
    if [ "$CURR_SIZE" = "$PREV_SIZE" ]; then
        echo "[$(date)] 警告: 访问日志文件大小未变化" >> $ALERT_FILE
    fi

    sleep 5
done
```

## 参考资料

### 相关源码

| 文件 | 行号 | 描述 |
|------|------|------|
| `AccessLogValve.java` | 103 | buffered 配置 |
| `AccessLogValve.java` | 620-622 | BufferedWriter 创建 |
| `AccessLogValve.java` | 358-404 | backgroundProcess() |
| `AccessLogValve.java` | 559-596 | log() 方法 |
| `ContainerBase.java` | 907-939 | 容器 backgroundProcess() |
| `ContainerBase.java` | 1115-1137 | 后台线程实现 |
