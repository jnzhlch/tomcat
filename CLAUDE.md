# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Apache Tomcat 是 Jakarta Servlet、Jakarta Pages (JSP)、Jakarta Expression Language (EL) 和 Jakarta WebSocket 技术的开源实现。这是一个成熟的企业级 Java Web 服务器和 Servlet 容器。

## 构建和测试命令

### 前置要求
- **JDK**: 版本 21 或更高 (设置 `JAVA_HOME` 环境变量)
- **Ant**: 版本 1.10.2 或更高

### 构建命令

```bash
# 默认构建 - 构建可用的 Tomcat 实例
ant

# 输出位置: output/build/
```

### 测试命令

```bash
# 运行所有测试
ant test

# 测试报告输出位置: output/build/logs/

# 运行单个测试类 (在 build.properties 中配置)
# test.entry=org.apache.catalina.util.TestServerInfo

# 运行单个测试的特定方法
# test.entry=org.apache.el.lang.TestELArithmetic
# test.entry.methods=testMultiply01,testMultiply02

# 运行特定测试集 (在 build.properties 中配置)
# test.name=**/TestSsl.java

# 仅运行 NIO 测试
ant test-nio
```

### 其他有用的构建目标

```bash
# 清理输出目录
ant clean

# 构建 Javadoc
ant javadoc

# 构建文档
ant build-docs

# 构建 extras 组件
ant extras

# 构建嵌入式包
ant embed

# 验证代码风格 (Checkstyle)
ant validate

# 创建完整发布版本
ant release
```

### 构建属性配置

在 `build.properties` 文件中配置以下属性以自定义构建：

```properties
# 依赖下载目录 (推荐设置在源码树外)
base.path=/path/to/tomcat-build-libs

# 代理配置 (如果需要)
proxy.use=true
proxy.host=proxy.domain
proxy.port=8080
```

### IDE 支持

```bash
# Eclipse
ant ide-eclipse

# IntelliJ IDEA
ant ide-intellij

# NetBeans
ant ide-netbeans
```

## 高层架构

Tomcat 采用分层架构设计，主要包含以下核心模块：

### 启动和类加载层次

```
Bootstrap.main()
    ↓ (初始化类加载器)
Common ClassLoader (共享类)
    ├── Catalina ClassLoader (Tomcat 内部类)
    └── Shared ClassLoader (Web 应用共享)
    ↓
Catalina.load() → Catalina.start()
    ↓
Server → Service → Engine → Host → Context
```

**关键入口点:**
- `java/org/apache/catalina/startup/Bootstrap.java` - Java 层入口，类加载器管理
- `java/org/apache/catalina/startup/Catalina.java` - 配置解析和启动协调

### 核心组件层次

1. **Server** - 顶层容器，代表整个 Tomcat 实例
2. **Service** - 组合一个 Engine 和多个 Connector
3. **Connector (Coyote)** - 处理网络连接和协议
4. **Engine** - 请求处理容器
5. **Host** - 虚拟主机
6. **Context** - Web 应用上下文

**核心实现类:**
- `java/org/apache/catalina/core/StandardServer.java`
- `java/org/apache/catalina/core/StandardService.java`
- `java/org/apache/catalina/core/StandardEngine.java`
- `java/org/apache/catalina/core/StandardHost.java`
- `java/org/apache/catalina/core/StandardContext.java`

### Coyote 连接器架构

Coyote 是 Tomcat 的连接器模块，负责处理底层网络 I/O 和协议：

```
AbstractEndpoint (网络 I/O)
    ↓
ProtocolHandler (HTTP/1.1, HTTP/2, AJP)
    ↓
Processor (请求处理)
    ↓
Adapter (连接到 Servlet 容器)
```

**关键包:**
- `java/org/apache/coyote/` - 核心接口和抽象类
- `java/org/apache/coyote/http11/` - HTTP/1.1 协议实现
- `java/org/apache/coyote/http2/` - HTTP/2 协议实现
- `java/org/apache/coyote/ajp/` - AJP 协议实现

**核心类:**
- `java/org/apache/coyote/AbstractProtocol.java` - 协议处理器基类
- `java/org/apache/coyote/Processor.java` - 请求处理器接口
- `java/org/apache/coyote/Adapter.java` - 连接到 Servlet 容器的桥接器

### 生命周期管理

所有组件实现统一的生命周期接口 (`Lifecycle`):

```
NEW → INITIALIZING → INITIALIZED → STARTING_PREP → STARTING → STARTED
                                                      ↓
                                                STOPPING_PREP → STOPPING → STOPPED
                                                      ↓
                                                DESTROYING → DESTROYED
```

**关键类:**
- `java/org/apache/catalina/Lifecycle.java` - 生命周期接口
- `java/org/apache/catalina/util/LifecycleBase.java` - 生命周期模板方法基类

### 配置文件

- **conf/server.xml** - 主配置文件，定义容器层次结构
- **conf/catalina.properties** - 类加载器和系统属性配置
- **conf/web.xml** - 默认 Web 应用配置

Tomcat 使用 Digester 框架解析 `server.xml`，将 XML 元素映射到 Java 对象。

### 请求处理流程

```
Client Request
    ↓
Connector (Coyote) - 解析协议
    ↓
Adapter - 转换为 Servlet 对象
    ↓
Engine → Host → Context → Wrapper
    ↓
Valve Chain (责任链模式)
    ↓
Filter Chain
    ↓
Servlet
```

## 代码风格指南

- **缩进**: 4 空格 (Java), 2 空格 (XML)
- **行宽**: 120 字符 (Java 源码), 80 字符 (文档)
- **大括号**: `{` 放在行尾
- **空格**: 使用空格而非 Tab

## 关键设计模式

1. **模板方法模式**: `LifecycleBase` 定义生命周期管理模板
2. **观察者模式**: `LifecycleListener` 用于状态变化通知
3. **责任链模式**: Valve 链处理请求
4. **工厂模式**: Digester 使用对象创建规则
5. **对象池模式**: `RecycledProcessors` 用于处理器重用

## 目录结构

```
java/org/apache/
├── catalina/          # Servlet 容器核心
│   ├── core/          # 核心容器实现 (Standard*)
│   ├── startup/       # 启动相关类
│   ├── connector/     # 连接器实现
│   ├── authenticator/ # 认证器
│   └── ...
├── coyote/            # 连接器框架
│   ├── http11/        # HTTP/1.1 协议
│   ├── http2/         # HTTP/2 协议
│   └── ajp/           # AJP 协议
├── tomcat/util/       # Tomcat 工具类
│   ├── net/           # 网络工具 (NIO Endpoint)
│   ├── http/          # HTTP 解析工具
│   └── buf/           # 字节缓冲区工具
├── jasper/            # JSP 编译器
└── ...
```

## 文档

详细的架构分析文档位于 `AnalysisDocs/` 目录：
- `TOMCAT-STARTUP-ARCHITECTURE.md` - 启动过程架构
- `coyote-ARCHITECTURE.md` - Coyote 连接器架构
- `tomcat-startup-process.md` - 启动过程详解
