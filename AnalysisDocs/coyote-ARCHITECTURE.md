# Coyote 架构文档

## 概述

Coyote 是 Tomcat 的核心连接器模块，负责处理底层网络 I/O 和协议处理。它作为网络层（socket 连接）和 Servlet 容器（Catalina）之间的桥梁。Coyote 的主要职责包括：

- **协议处理**：HTTP/1.0、HTTP/1.1、HTTP/2 和 AJP (Apache JServ Protocol)
- **连接管理**：接受、处理和管理客户端连接
- **请求/响应处理**：解析请求和生成响应
- **I/O 多路复用**：基于 NIO 的非阻塞 I/O 实现高性能连接处理

## 模块结构

```
java/org/apache/coyote/
├── 核心接口和类
│   ├── Processor.java           # 协议处理器接口
│   ├── ProtocolHandler.java     # 协议处理器接口
│   ├── Adapter.java             # 连接到 Servlet 容器的桥接器
│   ├── Request.java             # 底层请求表示
│   ├── Response.java            # 底层响应表示
│   ├── AbstractProcessor.java   # 处理器基类实现
│   ├── AbstractProtocol.java    # 协议处理器基类
│   ├── ActionCode.java          # 处理器动作代码
│   ├── ActionHook.java          # 处理器动作回调
│   ├── UpgradeProtocol.java     # 协议升级支持
│   ├── UpgradeToken.java        # 升级状态令牌
│   └── Constants.java           # 模块常量
│
├── HTTP/1.1 协议
│   ├── Http11Processor.java     # HTTP/1.1 请求处理器
│   ├── Http11NioProtocol.java   # HTTP/1.1 NIO 协议处理器
│   ├── AbstractHttp11Protocol.java
│   ├── Http11InputBuffer.java   # HTTP 请求解析
│   ├── Http11OutputBuffer.java  # HTTP 响应生成
│   ├── filters/                 # 输入/输出过滤器
│   │   ├── ChunkedInputFilter.java    # 分块输入过滤器
│   │   ├── ChunkedOutputFilter.java   # 分块输出过滤器
│   │   ├── GzipOutputFilter.java      # GZIP 压缩过滤器
│   │   ├── IdentityInputFilter.java   # 恒等输入过滤器
│   │   ├── IdentityOutputFilter.java  # 恒等输出过滤器
│   │   ├── BufferedInputFilter.java   # 缓冲输入过滤器
│   │   ├── SavedRequestInputFilter.java
│   │   ├── VoidInputFilter.java       # 空输入过滤器
│   │   └── VoidOutputFilter.java      # 空输出过滤器
│   └── upgrade/                 # HTTP 升级处理器
│
├── AJP 协议
│   ├── AjpProcessor.java        # AJP 请求处理器
│   ├── AjpNioProtocol.java      # AJP NIO 协议处理器
│   ├── AbstractAjpProtocol.java
│   ├── AjpMessage.java          # AJP 消息格式
│   └── Constants.java
│
├── HTTP/2 协议
│   └── (HTTP/2 实现类)
│
└── 工具类
    ├── AsyncStateMachine.java   # 异步处理状态机
    ├── CompressionConfig.java   # 压缩配置
    ├── ErrorState.java          # 错误状态跟踪
    └── ContinueResponseTiming.java
```

## 核心组件

### 1. ProtocolHandler（协议处理器）

`ProtocolHandler` 接口 (java/org/apache/coyote/ProtocolHandler.java:31) 定义了协议实现的契约：

```java
public interface ProtocolHandler {
    Adapter getAdapter();
    void setAdapter(Adapter adapter);
    Executor getExecutor();
    void init() throws Exception;
    void start() throws Exception;
    void pause() throws Exception;
    void resume() throws Exception;
    void stop() throws Exception;
    void destroy() throws Exception;
    // ... SSL 和升级协议管理
}
```

**主要职责：**
- 生命周期管理（初始化、启动、暂停、恢复、停止、销毁）
- 通过 Executor 管理线程池
- SSL 配置
- 协议升级支持
- 通过 Adapter 连接到 Servlet 容器

### 2. AbstractProtocol（抽象协议）

`AbstractProtocol` 类 (java/org/apache/coyote/AbstractProtocol.java:55) 为所有协议处理器提供基础实现：

```java
public abstract class AbstractProtocol<S> implements ProtocolHandler, MBeanRegistration {
    private final AbstractEndpoint<S,?> endpoint;  // 底层网络 I/O
    protected Adapter adapter;                       // 连接到 Servlet 容器
    private final Set<Processor> waitingProcessors;  // 异步处理器

    // 子类实现的抽象方法
    protected abstract Processor createProcessor();
    protected abstract String getNamePrefix();
    protected abstract String getProtocolName();
}
```

**主要特性：**
- 管理 `AbstractEndpoint` 用于底层 socket 操作
- 通过 `RecycledProcessors` 处理处理器池化和回收
- 协调异步超时处理
- JMX 注册用于监控

### 3. Processor（处理器）

`Processor` 接口 (java/org/apache/coyote/Processor.java:30) 处理单个连接：

```java
public interface Processor {
    SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status);
    UpgradeToken getUpgradeToken();
    boolean isUpgrade();
    boolean isAsync();
    void timeoutAsync(long now);
    Request getRequest();
    void recycle();
    void setSslSupport(SSLSupport sslSupport);
    ByteBuffer getLeftoverInput();
    void pause();
}
```

**状态转换：**
```
新建 → 解析中 → 服务中 → 输入结束 → 输出结束 → 保持连接 → 已关闭
               ↓
           异步/长连接
               ↓
           协议升级
```

### 4. Adapter（适配器）

`Adapter` 接口 (java/org/apache/coyote/Adapter.java:26) 是 Coyote 连接到 Servlet 容器的桥梁：

```java
public interface Adapter {
    void service(Request req, Response res) throws Exception;
    boolean prepare(Request req, Response res) throws Exception;
    boolean asyncDispatch(Request req, Response res, SocketEvent status);
    void log(Request req, Response res, long time);
    void checkRecycled(Request req, Response res);
    String getDomain();
}
```

Adapter 由 Catalina 的 `CoyoteAdapter` 实现，用于调用 Servlet 容器。

### 5. Request 和 Response

**Request** (java/org/apache/coyote/Request.java:54)：
- 高效的、GC 友好的底层 HTTP 请求表示
- 使用 `MessageBytes` 进行延迟字符串转换
- 包含：方法、URI、查询字符串、请求头、Cookie、参数
- 支持 `ReadListener` 实现非阻塞 I/O

**Response** (java/org/apache/coyote/Response.java:45)：
- 底层响应表示
- 状态码、响应头、输出缓冲区管理
- 错误状态跟踪（无 → 未报告 → 已报告）
- 支持 `WriteListener` 实现非阻塞 I/O

## 关键工作流程

### 请求处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     AbstractEndpoint                             │
│  (接受连接，分发到处理器池)                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              ConnectionHandler.process()                         │
│  • 获取/回收处理器从池中                                          │
│  • 处理协议协商 (ALPN)                                           │
│  • 处理 HTTP 升级                                                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Processor.process()                             │
│  • 解析请求行                                                    │
│  • 解析请求头                                                    │
│  • 准备请求（过滤器、验证）                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Adapter.service()                             │
│  • 将 coyote Request/Response 转换为 Servlet 对象                │
│  • 调用 Servlet 容器 (Catalina)                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                 响应生成                                          │
│  • 准备响应（请求头、压缩、分块）                                 │
│  • 写入响应体                                                     │
│  • 处理保持连接或关闭                                             │
└───────────────────────────────────────────────────────────────────┘
```

### HTTP/1.1 处理流程 (Http11Processor)

位于 `java/org/apache/coyote/http11/Http11Processor.java:72`：

1. **请求行解析**：解析方法、URI、协议版本
2. **请求头解析**：使用 `Http11InputBuffer` 解析所有请求头
3. **过滤器设置**：根据以下内容配置输入/输出过滤器：
   - Transfer-Encoding（分块）
   - Content-Length
   - 压缩设置
4. **Adapter.service()**：委托给 Servlet 容器
5. **响应准备**：
   - 确定内容分界（分块 vs content-length）
   - 如果配置了压缩则应用压缩
   - 设置 Connection 请求头（keep-alive vs close）
6. **清理**：回收处理器以供重用

### 过滤器链模式

Coyote 对输入和输出都使用过滤器链：

**输入过滤器** (java/org/apache/coyote/http11/filters/)：
```
Socket → 分块输入过滤器 → GZIP输入过滤器 → 缓冲输入过滤器 → 恒等输入过滤器 → 应用程序
```

**输出过滤器** (java/org/apache/coyote/http11/filters/)：
```
应用程序 → 恒等输出过滤器 → GZIP输出过滤器 → 分块输出过滤器 → Socket
```

### 异步处理流程

```
请求 → service() → Servlet.startAsync()
         ↓
    SocketState.LONG
         ↓
    Processor 添加到 waitingProcessors
         ↓
    异步超时监控定期检查
         ↓
    I/O 就绪或超时时 asyncDispatch()
         ↓
    Servlet complete/call
         ↓
    返回到 SocketState.OPEN/CLOSED
```

## 依赖关系

### 内部依赖

- **org.apache.tomcat.util.net**：底层 NIO 端点实现
- **org.apache.tomcat.util.http**：HTTP 解析工具（`HttpParser`、`MimeHeaders`）
- **org.apache.tomcat.util.buf**：字节缓冲区工具（`MessageBytes`、`ByteChunk`）
- **org.apache.tomcat.util.collections**：`SynchronizedStack` 用于处理器池化
- **org.apache.catalina**：Servlet 容器（通过 Adapter 接口）

### 外部依赖

- **jakarta.servlet**：Servlet API 规范
- **java.nio**：非阻塞 I/O 操作

## 集成点

### 与 Catalina (Servlet 容器) 集成

```
┌─────────────────┐         Adapter          ┌─────────────────┐
│     Coyote      │ ───────────────────────→ │    Catalina     │
│  (连接器)       │ ←───────────────────────  │ (Servlet 引擎)  │
└─────────────────┘   请求/响应              └─────────────────┘
```

`Adapter` 接口是集成点。Catalina 的 `CoyoteAdapter` 实现了这个接口。

### 与 Endpoint (网络层) 集成

```
┌─────────────────┐      AbstractEndpoint     ┌─────────────────┐
│ AbstractProtocol│ ───────────────────────→ │     NIOEndpoint │
│                 │ ←───────────────────────  │                 │
└─────────────────┘      SocketState          └─────────────────┘
```

`AbstractEndpoint` 处理：
- Socket 接受
- I/O 事件轮询
- Socket 状态管理
- Sendfile 支持

### 协议升级流程

```
HTTP/1.1 带有 Upgrade 请求头的请求
         ↓
Http11Processor 检测升级
         ↓
UpgradeProtocol.accept() 被调用
         ↓
创建带有 HttpUpgradeHandler 的 UpgradeToken
         ↓
返回 SocketState.UPGRADING
         ↓
ConnectionHandler 创建 UpgradeProcessor
         ↓
新协议接管连接
```

## 设计模式

### 1. 抽象工厂模式
- `ProtocolHandler.create(String protocol)` 工厂方法创建协议实例
- 每个协议（HTTP、AJP、HTTP/2）都有自己的具体工厂

### 2. 策略模式
- `InputFilter` 和 `OutputFilter` 接口允许可插拔的内容处理
- 可以交换不同的压缩算法

### 3. 责任链模式
- 用于请求/响应处理的过滤器链
- ActionCode 枚举和 ActionHook 用于处理器动作

### 4. 对象池模式
- `RecycledProcessors` 扩展 `SynchronizedStack` 用于处理器重用
- 减少高流量场景的 GC 压力

### 5. 模板方法模式
- `AbstractProcessor` 定义骨架
- 子类实现特定的协议行为（HTTP vs AJP）

## 代码示例

### 创建自定义协议处理器

```java
public class CustomProtocol extends AbstractProtocol<Socket> {

    public CustomProtocol() {
        super(new NioEndpoint());
    }

    @Override
    protected Log getLog() {
        return LogFactory.getLog(CustomProtocol.class);
    }

    @Override
    protected String getNamePrefix() {
        return "custom";
    }

    @Override
    protected String getProtocolName() {
        return "Custom";
    }

    @Override
    protected Processor createProcessor() {
        return new CustomProcessor(this, getAdapter());
    }

    @Override
    protected UpgradeProtocol getNegotiatedProtocol(String name) {
        return null; // 不支持 ALPN
    }

    @Override
    protected UpgradeProtocol getUpgradeProtocol(String name) {
        return null; // 不支持升级
    }

    @Override
    protected Processor createUpgradeProcessor(SocketWrapperBase<?> socket,
                                               UpgradeToken upgradeToken) {
        return null;
    }
}
```

### 自定义输入过滤器

```java
public class CustomInputFilter implements InputFilter {

    @Override
    public int doRead(ApplicationBufferHandler handler) throws IOException {
        // 自定义处理逻辑
        return -1;
    }

    @Override
    public void setRequest(Request request) {
        // 存储请求引用
    }

    @Override
    public long end() throws IOException {
        return 0;
    }

    @Override
    public int available() {
        return 0;
    }

    @Override
    public void recycle() {
        // 重置状态
    }

    @Override
    public MessageBytes getEncodingName() {
        return MessageBytes.newInstance("custom");
    }
}
```

## 性能考虑

1. **处理器池化**：重用 Processor 对象减少分配开销
2. **MessageBytes**：从字节延迟字符串转换（避免不必要的复制）
3. **非阻塞 I/O**：NIO 端点用少量线程处理大量连接
4. **Sendfile**：静态内容的零拷贝文件传输
5. **压缩**：GZIP 输出过滤器减少网络带宽
6. **保持连接**：连接复用减少 TLS 握手开销

## 线程安全

- **Processor**：每个连接通常是单线程的
- **AbstractProtocol**：使用 `ConcurrentHashMap` 管理等待中的处理器
- **Request/Response**：通过 Processor 关联实现线程本地
- **RecycledProcessors**：使用 `SynchronizedStack` 保证安全的并发访问

## 错误处理

Coyote 使用 `ErrorState` 枚举跟踪错误条件：

```
无 → 正常关闭 → 立即关闭
    ↓            ↓
 (正常)      (立即)
```

- 请求头解析期间的错误 → 400 Bad Request
- 不支持的协议版本 → 505 HTTP Version Not Supported
- I/O 错误 → 连接关闭
- Servlet 错误 → 记录日志并传递给错误处理

## 监控 (JMX)

Coyote 暴露多个 MBean：

- `ProtocolHandler`：协议级别统计
- `GlobalRequestProcessor`：聚合请求统计
- `RequestProcessor`：每个请求的指标

MBean 命名示例：
```
domain:type=ProtocolHandler,port=8080
domain:type=RequestProcessor,worker="http-nio-8080",name=HttpRequest1
```

## 关键文件引用

| 文件 | 行号 | 描述 |
|------|------|------|
| Processor.java | 30 | 处理器接口定义 |
| ProtocolHandler.java | 31 | 协议处理器接口定义 |
| Adapter.java | 26 | 适配器接口定义 |
| AbstractProtocol.java | 55 | 协议处理器基类 |
| AbstractProcessor.java | 42 | 处理器基类 |
| Request.java | 54 | 请求对象 |
| Response.java | 45 | 响应对象 |
| Http11Processor.java | 72 | HTTP/1.1 处理器 |
| Http11InputBuffer.java | 56 | HTTP 输入缓冲区 |
| Http11OutputBuffer.java | 54 | HTTP 输出缓冲区 |
| AjpProcessor.java | 69 | AJP 处理器 |

---

*文档位置：`./AnalysisDocs/coyote-ARCHITECTURE.md`*
*为 Tomcat 源码分析生成*
