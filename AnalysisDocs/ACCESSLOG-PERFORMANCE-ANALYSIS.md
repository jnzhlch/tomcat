# Tomcat 访问日志性能分析

## 概述

访问日志是 Web 服务器中最常用的功能之一，但在高并发场景下，日志记录可能成为性能瓶颈。本文档深入分析 Tomcat 访问日志模块的性能特征、潜在问题和优化方案。

## 性能基准测试

### Tomcat 官方基准测试

Tomcat 包含专门的基准测试代码，用于比较不同的线程安全实现策略：

**测试文件：** `test/org/apache/catalina/valves/Benchmarks.java`

#### 测试场景：日期格式化性能

| 实现方式 | 描述 | 性能特点 |
|---------|------|---------|
| **Syncs** | 使用 synchronized 同步块 | 简单但存在锁竞争 |
| **ThreadLocals** | 多个 ThreadLocal 变量 | 无锁但内存开销大 |
| **ThreadLocals with mutable Long** | ThreadLocal + 可变 Long | 减少 Long 拆箱开销 |
| **single ThreadLocal** | 单个 ThreadLocal 结构体 | **最优方案** |

**测试结论：**
```
单 ThreadLocal 结构体 > ThreadLocals with mutable Long > ThreadLocals > Syncs
```

## 性能瓶颈分析

### 1. 日期格式化 - 主要瓶颈

#### 问题根源

```java
// SimpleDateFormat 是线程不安全的
// 每次创建新对象开销巨大
SimpleDateFormat formatter = new SimpleDateFormat("dd/MMM/yyyy:HH:mm:ss Z");
String formatted = formatter.format(new Date());
```

**SimpleDateFormat 性能问题：**
- 对象创建成本高（约 2000-5000 CPU 周期）
- 每次格式化需要解析模式字符串
- 内部使用 Calendar 进行日期计算

#### Tomcat 解决方案：二级缓存

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         二级日期缓存架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Thread 1        Thread 2        Thread 3        Thread 4              │
│     │               │               │               │                   │
│     ▼               ▼               ▼               ▼                   │
│ ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐                 │
│ │  Local  │   │  Local  │   │  Local  │   │  Local  │  ← 一级缓存     │
│ │  Cache  │   │  Cache  │   │  Cache  │   │  Cache  │    (60 entries) │
│ │ (60条)  │   │ (60条)  │   │ (60条)  │   │ (60条)  │                 │
│ └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘                 │
│      │             │             │             │                       │
│      └─────────────┴─────────────┴─────────────┘                       │
│                    │ miss                                               │
│                    ▼                                                    │
│            ┌─────────────┐                                             │
│            │   Global    │  ← 二级缓存                                  │
│            │   Cache     │    (300 entries)                             │
│            │  (300条)    │                                             │
│            └─────┬───────┘                                             │
│                  │ miss                                                │
│                  ▼                                                     │
│            ┌─────────────┐                                             │
│            │SimpleDateFmt│  ← 最终格式化                                │
│            │   .format   │                                             │
│            └─────────────┘                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 缓存实现代码

```java
// AbstractAccessLogValve.java:227-348
protected static class DateFormatCache {
    // 循环缓冲区实现
    private final String[] cache;      // 缓存数组
    private long first;                 // 起始秒数
    private long last;                  // 结束秒数
    private int offset;                 // 循环偏移
    private long previousSeconds;       // 上次访问的秒数
    private String previousFormat;      // 上次的格式化结果

    private String getFormatInternal(long time) {
        long seconds = time / 1000;

        // 第一层：上次访问秒数命中
        if (seconds == previousSeconds) {
            return previousFormat;
        }

        // 第二层：在缓存范围内查找
        int index = (offset + (int)(seconds - first)) % cacheSize;
        if (seconds >= first && seconds <= last) {
            if (cache[index] != null) {
                previousFormat = cache[index];
                return previousFormat;
            }
        }

        // 第三层：缓存未命中，使用父缓存或格式化
        if (parent != null) {
            synchronized (parent) {
                previousFormat = parent.getFormatInternal(time);
            }
        } else {
            currentDate.setTime(time);
            previousFormat = formatter.format(currentDate);
        }
        cache[index] = previousFormat;
        return previousFormat;
    }
}
```

#### 缓存配置

```java
// AbstractAccessLogValve.java:191-196
private static final int globalCacheSize = 300;  // 全局缓存 300 秒
private static final int localCacheSize = 60;     // 线程本地缓存 60 秒
```

### 2. 同步锁竞争

#### 锁竞争点分析

| 位置 | 锁对象 | 竞争频率 | 影响 |
|------|--------|---------|------|
| `rotate()` | `this` | 每秒最多 1 次 | 低 |
| `log()` | `this` (buffered=false) | 每个请求 | **高** |
| 全局缓存 | `parent` | 缓存未命中时 | 中等 |

#### 日志写入同步

```java
// AccessLogValve.java:559-596
public void log(CharArrayWriter message) {
    rotate();

    // 外部程序移动文件的处理
    if (checkExists) {
        synchronized (this) {  // 锁竞争点
            if (currentLogFile != null && !currentLogFile.exists()) {
                close(false);
                open();
            }
        }
    }

    // 写入日志
    try {
        message.write(System.lineSeparator());
        synchronized (this) {  // 锁竞争点
            if (writer != null) {
                message.writeTo(writer);
                if (!buffered) {
                    writer.flush();  // 非缓冲模式每次 flush
                }
            }
        }
    } catch (IOException ioe) {
        log.warn(sm.getString("accessLogValve.writeFail", message.toString()), ioe);
    }
}
```

### 3. 字符串拼接开销

#### 传统方式问题

```java
// 低效的字符串拼接
String logEntry = host + " " + user + " " + timestamp + " \"" +
                  method + " " + uri + " " + protocol + "\" " +
                  status + " " + bytes;
```

**问题：**
- 每次创建多个临时 String 对象
- 大量内存分配和拷贝
- GC 压力大

#### Tomcat 优化方案：CharArrayWriter + 对象池

```java
// AbstractAccessLogValve.java:654-676
public void log(Request request, Response response, long time) {
    // 从对象池获取
    CharArrayWriter result = charArrayWriters.pop();
    if (result == null) {
        result = new CharArrayWriter(128);
    }

    // 逐元素写入，避免中间字符串
    for (AccessLogElement logElement : logElements) {
        logElement.addElement(result, request, response, time);
    }

    log(result);

    // 回收到对象池
    if (result.size() <= maxLogMessageBufferSize) {
        result.reset();
        charArrayWriters.push(result);
    }
}
```

### 4. I/O 操作开销

#### 文件写入瓶颈

```java
// AccessLogValve.java:620-622
writer = new PrintWriter(
    new BufferedWriter(
        new OutputStreamWriter(
            new FileOutputStream(pathname, true),  // append=true
            charset
        ),
        128000  // 128KB 缓冲区
    ),
    false
);
```

**I/O 性能特征：**
- `append=true`: 每次 write 都需要 seek 到文件末尾
- 128KB 缓冲区：平衡延迟和吞吐
- `buffered=false`: 每条日志立即刷盘（低延迟，低吞吐）
- `buffered=true`: 批量刷盘（高吞吐，有丢失风险）

### 5. 模式解析开销

#### 首次解析

```java
// AbstractAccessLogValve.java:414-455
protected AccessLogElement[] createLogElements() {
    List<AccessLogElement> logElements = new ArrayList<>();
    // 解析 pattern 字符串，创建对应的 AccessLogElement
    // 只在首次调用或 pattern 变化时执行
}
```

**影响：** 首次请求延迟稍高，但后续无影响

## 性能优化建议

### 1. 配置优化

#### 推荐 server.xml 配置

```xml
<!-- 高性能配置 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="access."
       suffix=".log"
       pattern="combined"
       rotatable="true"
       buffered="true"           <!-- 启用缓冲，减少 I/O -->
       encoding="UTF-8"
       fileDateFormat=".yyyy-MM-dd"
       maxDays="30"/>

<!-- 超高吞吐场景 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="/dev/shm/tomcat-logs"  <!-- 使用内存文件系统 -->
       prefix="access."
       suffix=".log"
       pattern="%h %l %u %t \"%r\" %s %b %D"  <!-- 简化模式 -->
       buffered="true"
       maxDays="0"               <!-- 不自动清理，由外部处理 -->
       checkExists="false"/>     <!-- 禁用文件检查 -->

<!-- 低延迟场景 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="access."
       suffix=".log"
       pattern="common"
       buffered="false"          <!-- 每条日志立即刷盘 -->
       checkExists="true"/>      <!-- 支持外部日志轮转 -->
```

### 2. 模式优化

#### 不同模式的性能对比

| 模式 | 复杂度 | 相对性能 | 用途 |
|------|--------|---------|------|
| `common` | 低 | 100% (基准) | 一般访问日志 |
| `combined` | 低 | 98% | 包含 Referer 和 User-Agent |
| `%t` (CLF) | 中 | 85% | 标准时间格式，需要日期缓存 |
| `%{xxx}i` | 中 | 80% | 读取请求头 |
| `%{xxx}c` | 高 | 70% | 解析 Cookie |
| `%{xxx}r` | 高 | 65% | 读取请求属性 |
| `%{xxx}s` | 高 | 60% | 读取 Session 属性 |

#### 优化建议

```xml
<!-- 高性能模式：只记录必要信息 -->
pattern="%a %t %m %U %s %b %D"
<!-- 输出: 192.168.1.100 [12/Mar/2025:10:30:45 +0800] GET /api/data 200 1234 125000 -->
```

### 3. 条件日志

```xml
<!-- 跳过健康检查请求 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       pattern="common"
       conditionUnless="skipLog"/>

<!-- 在应用中设置属性 -->
request.setAttribute("skipLog", "true");  // 不记录
```

### 4. 异步日志（第三方方案）

虽然 Tomcat 核心不支持异步日志，但可以集成第三方方案：

#### 方案 A：使用 Log4j2 Async Logger

```xml
<!-- 使用 Log4j2 作为日志后端 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       pattern="%{X-Request-ID}r %h %l %u %t &quot;%r&quot; %s %b"/>
```

配合 Log4j2 配置：
```xml
<Configuration status="WARN">
    <Appenders>
        <RollingFile name="AccessLog" fileName="logs/access.log"
                     filePattern="logs/access-%d{yyyy-MM-dd}.log">
            <PatternLayout pattern="%d %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <AsyncLogger name="org.apache.catalina.valves.AccessLogValve"
                     level="info" additivity="false">
            <AppenderRef ref="AccessLog"/>
        </AsyncLogger>
    </Loggers>
</Configuration>
```

#### 方案 B：使用独立的日志收集器

```
Tomcat (禁用内置日志) → Unix Socket → Fluentd/Logstash → ES/文件
```

### 5. 文件系统优化

#### 使用内存文件系统（Linux）

```bash
# 创建内存文件系统挂载点
sudo mkdir -p /dev/shm/tomcat-logs
sudo mount -t tmpfs -o size=1G tmpfs /dev/shm/tomcat-logs

# 配置 Tomcat 使用该目录
# 然后使用 cron/rsync 定期同步到磁盘
```

#### 使用 SSD

```
HDD → 100-200 IOPS
SSD → 10,000-100,000 IOPS
NVMe → 500,000+ IOPS
```

### 6. JVM 优化

#### JVM 参数建议

```bash
# 减少 GC 压力
-XX:+UseG1GC                    # 使用 G1 垃圾回收器
-XX:MaxGCPauseMillis=200        # 目标最大 GC 暂停时间
-XX:G1HeapRegionSize=16m        # Region 大小

# 增加年轻代，减少对象晋升
-Xmn2g                          # 年轻代 2GB
-XX:SurvivorRatio=8             # Eden:Survivor = 8:1

# 紧急内存优化
-XX:+UseStringDeduplication     # 字符串去重
-XX:+UseCompressedOops          # 压缩普通对象指针
```

## 性能测试场景

### 测试工具：JMeter

```xml
<!-- JMeter 测试计划示例 -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="访问日志性能测试">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments">
          <elementProp name="THREADS" elementType="Argument">
            <stringProp name="Argument.name">THREADS</stringProp>
            <stringProp name="Argument.value">100</stringProp>
          </elementProp>
          <elementProp name="LOOPS" elementType="Argument">
            <stringProp name="Argument.name">LOOPS</stringProp>
            <stringProp name="Argument.value">10000</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="线程组">
        <stringProp name="ThreadGroup.num_threads">${THREADS}</stringProp>
        <stringProp name="ThreadGroup.loops">${LOOPS}</stringProp>
      </ThreadGroup>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### 预期性能指标

| 场景 | QPS | 平均响应时间 | P99 响应时间 | CPU 使用率 |
|------|-----|-------------|-------------|-----------|
| 无日志 | 50,000+ | < 1ms | < 5ms | 60% |
| 简单日志 (common) | 45,000+ | < 1ms | < 6ms | 65% |
| 完整日志 (combined) | 40,000+ | < 2ms | < 8ms | 70% |
| 复杂日志 (%{xxx}s) | 25,000+ | < 3ms | < 15ms | 80% |
| 非缓冲日志 | 10,000+ | < 5ms | < 30ms | 85% |

## 常见性能问题

### 问题 1：日志文件满导致磁盘 I/O 100%

**现象：**
```
# iostat 显示
Device         rrqm/s   wrqm/s     r/s     w/s  %util
sda               0.0     0.0  1000.0  5000.0  100.00
```

**解决方案：**
```xml
<!-- 减小日志保留天数 -->
<Valve maxDays="7"/>

<!-- 或使用独立磁盘 -->
<Valve directory="/mnt/ssd/tomcat-logs"/>
```

### 问题 2：内存泄漏（对象池未回收）

**现象：**
```
# jmap -histo 显示
java.util.ArrayList  # 不断增加
char[]               # 不断增加
```

**检查：**
```java
// AbstractAccessLogValve.java:672-675
if (result.size() <= maxLogMessageBufferSize) {
    result.reset();
    charArrayWriters.push(result);
}
// 确保大日志消息不会被回收到对象池
```

### 问题 3：时区切换导致缓存失效

**现象：** 每次请求都格式化日期，性能下降

**解决方案：**
```xml
<!-- 固定时区 -->
<Valve pattern="%{yyyy-MM-dd HH:mm:ss Z}t"/>
```

## 监控指标

### 关键指标

| 指标 | 获取方式 | 告警阈值 |
|------|---------|---------|
| 日志写入延迟 | JMX `AccessLogValve@AvgLogTime` | > 10ms |
| 日志队列深度 | JMX `AccessLogValve@QueueSize` | > 1000 |
| 文件描述符数 | `lsof -p <pid> \| wc -l` | > 10000 |
| 磁盘 I/O 等待 | `iostat -x 1` %iowait | > 20% |
| GC 频率 | `jstat -gcutil` | Full GC > 1次/分钟 |

### 监控脚本示例

```bash
#!/bin/bash
# 访问日志性能监控脚本

while true; do
    echo "=== $(date) ==="

    # 检查日志文件大小
    for file in /opt/tomcat/logs/access*.log; do
        size=$(stat -f%z "$file" 2>/dev/null || stat -c%s "$file")
        echo "Log file $file: $(($size / 1024)) KB"
    done

    # 检查磁盘 I/O
    iostat -x 1 1 | grep -E "Device|sd"

    # 检查打开的文件数
    lsof -p $(pgrep -f tomcat) | grep -c "\.log"

    sleep 60
done
```

## 参考资料

### 相关源码文件

| 文件 | 描述 |
|------|------|
| `AbstractAccessLogValve.java` | 访问日志核心实现，包含日期缓存 |
| `AccessLogValve.java` | 文件输出实现 |
| `Benchmarks.java` | 官方性能基准测试 |
| `DateFormatCache.java` | 日期格式缓存工具类 |
| `TestAccessLogValveDateFormatCache.java` | 缓存单元测试 |

### 相关文档

- `AnalysisDocs/ACCESSLOG-ARCHITECTURE.md` - 访问日志架构文档
- `webapps/docs/config/valve.html` - 官方 Valve 配置文档
