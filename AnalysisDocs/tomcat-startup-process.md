# Tomcat 启动过程详解

## 概述

Tomcat 的启动过程是一个多阶段、分层级的初始化和启动流程，涉及多个核心组件的协同工作。整个流程从 `Bootstrap.main()` 方法开始，最终完成 Servlet 容器的初始化并开始接受 HTTP 请求。

**启动命令层次：**
```
startup.sh/catalina.sh
    ↓
Bootstrap.main()
    ↓
Catalina.load() → Catalina.start()
    ↓
Server.init() → Server.start()
    ↓
Service.init() → Service.start()
    ↓
Engine.init() → Engine.start()
    ↓
Host.init() → Host.start()
    ↓
Context.init() → Context.start()
```

## 启动入口

### 1. Bootstrap 类

**文件位置：** `java/org/apache/catalina/startup/Bootstrap.java`

**主要职责：**
- 初始化类加载器（Common、Catalina、Shared）
- 加载并实例化 Catalina 类
- 通过反射调用 Catalina 的方法

**关键代码 (Bootstrap.java:429)：**
```java
public static void main(String[] args) {
    // 1. 创建并初始化 Bootstrap 实例
    synchronized (daemonLock) {
        if (daemon == null) {
            Bootstrap bootstrap = new Bootstrap();
            try {
                bootstrap.init();  // 初始化类加载器
            } catch (Throwable t) {
                handleThrowable(t);
                log.error("Init exception", t);
                return;
            }
            daemon = bootstrap;
        }
    }

    // 2. 解析命令并执行
    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }

        switch (command) {
            case "start":
                daemon.setAwait(true);
                daemon.load(args);   // 加载配置
                daemon.start();     // 启动服务器
                break;
            // ... 其他命令
        }
    } catch (Throwable t) {
        handleThrowable(t);
    }
}
```

### 2. 类加载器初始化

**Bootstrap.init() 方法 (Bootstrap.java:245)：**
```java
public void init() throws Exception {
    // 1. 初始化三层类加载器
    initClassLoaders();

    // 2. 设置当前线程的上下文类加载器
    Thread.currentThread().setContextClassLoader(catalinaLoader);

    // 3. 加载 Catalina 类
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();

    // 4. 设置共享类加载器
    String methodName = "setParentClassLoader";
    // ... 通过反射调用 setParentClassLoader(sharedLoader)

    catalinaDaemon = startupInstance;
}
```

**类加载器层次结构：**
```
Bootstrap ClassLoader (JDK)
    ↓
System ClassLoader
    ↓
Common ClassLoader (加载 common/ 目录下的 JAR)
    ↓
    ├── Catalina ClassLoader (加载 server/ 目录 - Tomcat 内部类)
    └── Shared ClassLoader (加载 shared/ 目录 - Web 应用共享)
```

**initClassLoaders() 方法 (Bootstrap.java:136)：**
```java
private void initClassLoaders() {
    try {
        // Common 类加载器 - 所有 Tomcat 组件和 Web 应用共享
        commonLoader = createClassLoader("common", null);
        if (commonLoader == null) {
            commonLoader = this.getClass().getClassLoader();
        }

        // Catalina 类加载器 - Tomcat 内部使用，对 Web 应用不可见
        catalinaLoader = createClassLoader("server", commonLoader);

        // Shared 类加载器 - Web 应用共享的类
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        handleThrowable(t);
        log.error("Class loader creation threw exception", t);
        System.exit(1);
    }
}
```

## 配置加载阶段

### 3. Catalina.load() 方法

**文件位置：** `java/org/apache/catalina/startup/Catalina.java`

**Catalina.load() 方法 (Catalina.java:679)：**
```java
public void load() {
    if (loaded) {
        return;
    }
    loaded = true;

    // 1. 初始化 JNDI
    initNaming();

    // 2. 解析 server.xml 配置文件
    parseServerXml(true);
    Server s = getServer();
    if (s == null) {
        return;
    }

    // 3. 设置 Server 的 catalina 相关属性
    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    // 4. 初始化流重定向
    initStreams();

    // 5. 初始化 Server
    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (throwOnInitFailure) {
            throw new Error(e);
        } else {
            log.error(sm.getString("catalina.initError"), e);
        }
    }
}
```

### 4. server.xml 解析

**createStartDigester() 方法 (Catalina.java:383)：**

Tomcat 使用 Digester 框架解析 XML 配置文件，采用规则驱动的方式：

```java
protected Digester createStartDigester() {
    Digester digester = new Digester();
    digester.setValidating(false);

    // Server 对象创建规则
    digester.addObjectCreate("Server", "org.apache.catalina.core.StandardServer", "className");
    digester.addSetProperties("Server");
    digester.addSetNext("Server", "setServer", "org.apache.catalina.Server");

    // GlobalNamingResources 规则
    digester.addObjectCreate("Server/GlobalNamingResources",
        "org.apache.catalina.deploy.NamingResourcesImpl");
    digester.addSetProperties("Server/GlobalNamingResources");
    digester.addSetNext("Server/GlobalNamingResources", "setGlobalNamingResources");

    // Listener 规则
    digester.addRule("Server/Listener", new ListenerCreateRule(null, "className"));
    digester.addSetProperties("Server/Listener");
    digester.addSetNext("Server/Listener", "addLifecycleListener");

    // Service 规则
    digester.addObjectCreate("Server/Service", "org.apache.catalina.core.StandardService", "className");
    digester.addSetProperties("Server/Service");
    digester.addSetNext("Server/Service", "addService", "org.apache.catalina.Service");

    // Executor 规则
    digester.addObjectCreate("Server/Service/Executor",
        "org.apache.catalina.core.StandardThreadExecutor", "className");
    digester.addSetProperties("Server/Service/Executor");
    digester.addSetNext("Server/Service/Executor", "addExecutor");

    // Connector 规则
    digester.addRule("Server/Service/Connector", new ConnectorCreateRule());
    digester.addSetProperties("Server/Service/Connector");
    digester.addSetNext("Server/Service/Connector", "addConnector");

    // Engine 规则 (使用 RuleSet)
    digester.addRuleSet(new EngineRuleSet("Server/Service/"));

    // Host 规则 (使用 RuleSet)
    digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));

    // Context 规则 (使用 RuleSet)
    digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));

    return digester;
}
```

**配置解析流程 (Catalina.java:532)：**
```java
protected void parseServerXml(boolean start) {
    // 1. 设置配置文件源
    ConfigFileLoader.setSource(new CatalinaBaseConfigurationSource(
        Bootstrap.getCatalinaBaseFile(), getConfigFile()));

    // 2. 创建 Digester
    Digester digester = start ? createStartDigester() : createStopDigester();

    // 3. 解析配置文件
    try (ConfigurationSource.Resource resource = ConfigFileLoader.getSource().getServerXml()) {
        InputStream inputStream = resource.getInputStream();
        InputSource inputSource = new InputSource(resource.getURI().toURL().toString());
        inputSource.setByteStream(inputStream);

        digester.push(this);  // 将 Catalina 对象压入栈顶
        digester.parse(inputSource);  // 解析 XML
    } catch (Exception e) {
        log.warn(sm.getString("catalina.configFail", file.getAbsolutePath()), e);
    }
}
```

## 生命周期管理

### 5. LifecycleBase 类

**文件位置：** `java/org/apache/catalina/util/LifecycleBase.java`

**生命周期状态转换：**
```
NEW → INITIALIZING → INITIALIZED
  ↓                    ↓
STARTING_PREP → STARTING → STARTED
  ↓              ↓
STOPPING_PREP → STOPPING → STOPPED
  ↓              ↓
DESTROYING → DESTROYED
```

**init() 方法 (LifecycleBase.java:115)：**
```java
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(BEFORE_INIT_EVENT);
    }

    try {
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        initInternal();  // 调用子类实现
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.initFail", toString());
    }
}
```

**start() 方法 (LifecycleBase.java:139)：**
```java
public final synchronized void start() throws LifecycleException {
    // 检查状态
    if (LifecycleState.STARTING_PREP.equals(state) ||
        LifecycleState.STARTING.equals(state) ||
        LifecycleState.STARTED.equals(state)) {
        return;  // 已启动
    }

    // 如果是 NEW 状态，先初始化
    if (state.equals(LifecycleState.NEW)) {
        init();
    }
    // 如果是 FAILED 状态，先停止
    else if (state.equals(LifecycleState.FAILED)) {
        stop();
    }
    // 必须在 INITIALIZED 或 STOPPED 状态才能启动
    else if (!state.equals(LifecycleState.INITIALIZED) &&
             !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(BEFORE_START_EVENT);
    }

    try {
        setStateInternal(LifecycleState.STARTING_PREP, null, false);
        startInternal();  // 调用子类实现
        if (state.equals(LifecycleState.FAILED)) {
            stop();
        } else if (!state.equals(LifecycleState.STARTING)) {
            invalidTransition(AFTER_START_EVENT);
        } else {
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.startFail", toString());
    }
}
```

## Server 初始化和启动

### 6. StandardServer.initInternal()

**文件位置：** `java/org/apache/catalina/core/StandardServer.java`

**initInternal() 方法 (StandardServer.java:931)：**
```java
protected void initInternal() throws LifecycleException {
    super.initInternal();

    // 1. 注册全局 String 缓存 (JMX)
    onameStringCache = register(new StringCache(), "type=StringCache");

    // 2. 注册 MBeanFactory (JMX 管理工厂)
    MBeanFactory factory = new MBeanFactory();
    factory.setContainer(this);
    onameMBeanFactory = register(factory, "type=MBeanFactory");

    // 3. 初始化全局命名资源
    globalNamingResources.init();

    // 4. 初始化所有 Service
    for (Service service : findServices()) {
        service.init();
    }
}
```

**startInternal() 方法 (StandardServer.java:849)：**
```java
protected void startInternal() throws LifecycleException {
    fireLifecycleEvent(CONFIGURE_START_EVENT, null);
    setState(LifecycleState.STARTING);

    // 1. 初始化工具线程池
    synchronized (utilityExecutorLock) {
        reconfigureUtilityExecutor(getUtilityThreadsInternal(utilityThreads));
        register(utilityExecutor, "type=UtilityExecutor");
    }

    // 2. 启动全局命名资源
    globalNamingResources.start();

    // 3. 启动所有 Service
    for (Service service : findServices()) {
        service.start();
    }

    // 4. 启动周期性生命周期事件
    if (periodicEventDelay > 0) {
        monitorFuture = getUtilityExecutor().scheduleWithFixedDelay(
            this::startPeriodicLifecycleEvent, 0, 60, TimeUnit.SECONDS);
    }
}
```

### 7. await() 方法 - 关闭端口监听

**await() 方法 (StandardServer.java:505)：**
```java
@Override
public void await() {
    // 负值表示不等待端口（嵌入式模式）
    if (getPortWithOffset() == -2) {
        return;
    }

    // -1 表示使用线程等待模式
    if (getPortWithOffset() == -1) {
        Thread currentThread = Thread.currentThread();
        try {
            awaitThread = currentThread;
            while (!stopAwait) {
                Thread.sleep(10000);
            }
        } finally {
            awaitThread = null;
        }
        return;
    }

    // 创建 ServerSocket 监听关闭命令
    try {
        awaitSocket = new ServerSocket(getPortWithOffset(), 1,
            InetAddress.getByName(address));
    } catch (IOException ioe) {
        log.error(sm.getString("standardServer.awaitSocket.fail",
            address, String.valueOf(getPortWithOffset())));
        return;
    }

    // 循环等待关闭命令
    while (!stopAwait) {
        try {
            Socket socket = awaitSocket.accept();
            socket.setSoTimeout(10 * 1000);
            InputStream stream = socket.getInputStream();

            // 读取命令字符串
            StringBuilder command = new StringBuilder();
            int expected = 1024;
            while (expected > 0) {
                int ch = stream.read();
                if (ch < 32 || ch == 127) break;  // 控制字符终止
                command.append((char) ch);
                expected--;
            }

            // 检查是否匹配关闭命令
            boolean match = command.toString().equals(shutdown);
            if (match) {
                log.info(sm.getString("standardServer.shutdownViaPort"));
                break;
            }
        } finally {
            if (socket != null) {
                socket.close();
            }
        }
    }
}
```

## Service 初始化和启动

### 8. StandardService

**文件位置：** `java/org/apache/catalina/core/StandardService.java`

**initInternal() 方法：**
```java
protected void initInternal() throws LifecycleException {
    super.initInternal();

    // 初始化所有容器
    if (engine != null) {
        engine.init();
    }

    // 初始化 Executor 线程池
    for (Executor executor : findExecutors()) {
        if (executor instanceof JmxEnabled) {
            ((JmxEnabled) executor).setDomain(getDomain());
        }
        executor.init();
    }

    // 初始化 Connector（将 Connector 与 Executor 关联）
    for (Connector connector : findConnectors()) {
        if (connector.getState().isAvailable()) {
            connector.init();
        }
    }
}
```

**startInternal() 方法：**
```java
protected void startInternal() throws LifecycleException {
    // 启动 Engine
    if (engine != null) {
        engine.start();
    }

    // 启动 Executor 线程池
    for (Executor executor : findExecutors()) {
        executor.start();
    }

    // 启动 Connector（开始接受连接）
    for (Connector connector : findConnectors()) {
        // 只有状态为 available 的 Connector 才启动
        if (connector.getState().isAvailable()) {
            connector.start();
        }
    }
}
```

## Engine、Host、Context 启动

### 9. 组件层级启动流程

```
StandardService
    ↓
StandardEngine (容器顶层)
    ↓
StandardHost (虚拟主机)
    ↓
StandardContext (Web 应用上下文)
```

**StandardEngine.initInternal()：**
```java
protected void initInternal() throws LifecycleException {
    super.initInternal();
    // 初始化所有 Host
    for (Container child : findChildren()) {
        child.init();
    }
}
```

**StandardHost.initInternal()：**
```java
protected void initInternal() throws LifecycleException {
    super.initInternal();
    // 初始化所有 Context
    for (Container child : findChildren()) {
        child.init();
    }
}
```

**StandardContext.initInternal() - 最复杂的初始化：**
```java
protected void initInternal() throws LifecycleException {
    super.initInternal();

    // 1. 创建资源集合
    if (resources == null) {
        resources = new StandardRoot(this);
    }

    // 2. 创建 Loader (Web 应用类加载器)
    if (loader == null) {
        loader = new WebappLoader(parentClassLoader);
    }

    // 3. 创建 Manager (Session 管理)
    if (manager == null) {
        manager = new StandardManager();
    }

    // 4. 初始化字符集映射器
    charsetMapperStart();

    // 5. 初始化工作目录
    workDir = getWorkDirectory();

    // 6. 命名资源初始化
    if (getNamingContextListener() != null) {
        namingContextListener.lifecycleEvent(
            new LifecycleEvent(this, Lifecycle.INIT_EVENT, null));
    }

    // 7. 启动时加载 Web 应用配置
    if (processTlds) {
        // 处理 TLD 文件（标签库描述符）
    }
}
```

**StandardContext.startInternal() - Web 应用启动：**
```java
protected void startInternal() throws LifecycleException {
    // 1. 设置状态
    setAvailable(false);

    // 2. 创建 WebappClassLoader
    if (getLoader() == null) {
        WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
        webappLoader.setDelegate(getDelegate());
        setLoader(webappLoader);
    }

    // 3. 配置资源
    if (getResources() == null) {
        setResources(new StandardRoot(this));
    }

    // 4. 初始化 JNDI
    if (getNamingContextListener() != null) {
        namingContextListener.lifecycleEvent(
            new LifecycleEvent(this, Lifecycle.START_EVENT, null));
    }

    // 5. 加载 web.xml 并配置 Servlet
    for (Container child : findChildren()) {
        if (!child.getState().isAvailable()) {
            child.start();
        }
    }

    // 6. 启动 Web 应用类加载器
    if (getLoader() != null) {
        getLoader().start();
    }

    // 7. 触发 ServletContainerInitializer
    for (ServletContainerInitializer initializer : initializers) {
        initializer.onStartup(initializerClasses, getServletContext());
    }

    // 8. 标记 Web 应用可用
    setAvailable(true);

    // 9. 启动 Session 管理
    if (getManager() != null) {
        getManager().start();
    }
}
```

## Connector 启动流程

### 10. CoyoteAdapter 初始化

**Connector 初始化序列：**
```
Connector.init()
    ↓
ProtocolHandler.init()  (例如 Http11NioProtocol)
    ↓
Endpoint.init()  (NioEndpoint)
    ├── 创建 Acceptor 线程
    ├── 创建 Poller 线程
    └── 初始化 SSL
```

**Http11NioProtocol.init() 方法：**
```java
public void init() throws Exception {
    // 初始化 Endpoint
    endpoint.init();

    // 初始化连接超时
    if (connectionTimeout > 0) {
        endpoint.setConnectionTimeout(connectionTimeout);
    }

    // 初始化 keep-alive 超时
    if (keepAliveTimeout > 0) {
        endpoint.setKeepAliveTimeout(keepAliveTimeout);
    }
}
```

**NioEndpoint.init() 方法：**
```java
public void init() throws Exception {
    if (getMaxConnections() == 0) {
        // 默认最大连接数
        setMaxConnections(getMaxThreads() * 5);
    }

    if (getMaxThreads() == 0) {
        // 默认最大线程数
        setMaxThreads(Math.max(getMinSpareThreads(), 200));
    }

    // 初始化 ServerSocketChannel
    if (serverSocket == null) {
        serverSocket = ServerSocketChannel.open();
        socketProperties.setProperties(serverSocket.socket());
    }
}
```

### 11. Connector 启动和端口绑定

**Connector.start() 方法：**
```java
public void start() throws LifecycleException {
    setState(LifecycleState.STARTING);

    // 启动 ProtocolHandler
    try {
        protocolHandler.start();
    } catch (Exception e) {
        setState(LifecycleState.FAILED);
        throw e;
    }

    setState(LifecycleState.STARTED);
}
```

**NioEndpoint.start() 方法：**
```java
public void start() throws Exception {
    // 1. 绑定端口
    if (bindState == BindState.UNBOUND) {
        bind();
        bindState = BindState.BOUND_ONSTART;
    }

    // 2. 初始化线程池
    if (internalExecutor == null) {
        internalExecutor = new ThreadPoolExecutor(
            getMinSpareThreads(),
            getMaxThreads(),
            60,
            TimeUnit.SECONDS,
            taskQueue,
            new TaskThreadFactory("Tomcat-http--"));
    }

    // 3. 启动 Poller 线程（I/O 事件轮询）
    for (int i = 0; i < pollers; i++) {
        Poller poller = new Poller();
        poller.init();
        poller.setPollerIndex(i);
        poller.start();
    }

    // 4. 启动 Acceptor 线程（接受新连接）
    for (int i = 0; i < acceptors; i++) {
        Acceptor acceptor = new Acceptor<>(this);
        acceptor.setPollerIndex(i % pollers);
        acceptors[i].start();
    }
}
```

## 完整启动流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Bootstrap.main()                             │
│  1. 创建 Bootstrap 实例                                                │
│  2. init() - 初始化类加载器                                            │
│     ├── CommonClassLoader (common/)                                   │
│     ├── CatalinaClassLoader (server/)                                │
│     └── SharedClassLoader (shared/)                                  │
│  3. load(args) - 加载配置并初始化                                      │
│  4. start() - 启动服务器                                              │
│  5. await() - 阻塞主线程等待关闭命令                                   │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Catalina.load()                                 │
│  1. initNaming() - 初始化 JNDI                                        │
│  2. parseServerXml(true) - 解析 server.xml                           │
│     ├── 创建 Digester 和规则                                           │
│     ├── 解析 XML 元素                                                 │
│     ├── 创建对象：Server → Service → Connector/Engine               │
│     └── 建立对象层次结构                                               │
│  3. server.init() - 初始化 Server                                     │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    StandardServer.initInternal()                     │
│  1. 注册 StringCache (JMX)                                            │
│  2. 注册 MBeanFactory (JMX)                                           │
│  3. globalNamingResources.init()                                      │
│  4. 遍历所有 Service → service.init()                                │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   StandardService.initInternal()                     │
│  1. engine.init()                                                    │
│  2. 遍历所有 Executor → executor.init()                              │
│  3. 遍历所有 Connector → connector.init()                            │
│     └── protocolHandler.init()                                       │
│         └── endpoint.init()                                           │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Catalina.start()                                 │
│  1. server.start()                                                   │
│  2. 注册关闭钩子 (shutdown hook)                                     │
│  3. await() - 阻塞等待关闭命令                                        │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   StandardServer.startInternal()                    │
│  1. 初始化 utilityExecutor                                            │
│  2. globalNamingResources.start()                                    │
│  3. 遍历所有 Service → service.start()                                │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   StandardService.startInternal()                    │
│  1. engine.start()                                                   │
│  2. 遍历所有 Executor → executor.start()                            │
│  3. 遍历所有 Connector → connector.start()                            │
│     └── protocolHandler.start()                                      │
│         └── endpoint.start()                                         │
│             ├── 启动 Poller 线程                                        │
│             ├── 启动 Acceptor 线程                                     │
│             └── 绑定端口，开始接受连接                                 │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    容器层级启动                                       │
│  StandardEngine.startInternal()                                      │
│      → StandardHost.startInternal()                                  │
│          → StandardContext.startInternal()                          │
│              ├── 加载 web.xml                                         │
│              ├── 创建 WebappClassLoader                               │
│              ├── 初始化 Servlet                                       │
│              ├── 调用 ServletContainerInitializer                   │
│              └── 启动 Session 管理                                    │
└──────────────────────────────────────────────────────────────────────┘
```

## 关键时间点

| 阶段 | 操作 | 说明 |
|------|------|------|
| Bootstrap.init() | 类加载器创建 | 创建 Common、Catalina、Shared 三层类加载器 |
| Catalina.load() | 配置解析 | 解析 server.xml，创建对象层次结构 |
| Server.init() | 对象初始化 | 初始化所有 Service、Engine、Host、Context |
| Server.start() | 服务启动 | 启动所有 Service，Connector 开始接受连接 |
| Context.start() | Web 应用加载 | 加载 Servlet，Filter，Listener 等组件 |

## 启动参数

**命令行参数：**
- `-config {path}` - 指定配置文件路径（默认 conf/server.xml）
- `-nonaming` - 禁用 JNDI 支持
- `-help` - 显示帮助信息
- `start` - 启动服务器
- `stop` - 停止服务器
- `configtest` - 测试配置文件

**系统属性：**
- `catalina.home` - Tomcat 安装目录
- `catalina.base` - Tomcat 实例目录（默认与 home 相同）
- `java.util.logging.manager` - 日志管理器
- `java.util.logging.config.file` - 日志配置文件

## 关键文件引用

| 文件 | 行号 | 描述 |
|------|------|------|
| Bootstrap.java | 429 | main() 方法入口 |
| Bootstrap.java | 245 | init() 方法，类加载器初始化 |
| Bootstrap.java | 136 | initClassLoaders() 方法 |
| Catalina.java | 679 | load() 方法 |
| Catalina.java | 741 | start() 方法 |
| Catalina.java | 383 | createStartDigester() 方法 |
| Catalina.java | 532 | parseServerXml() 方法 |
| LifecycleBase.java | 115 | init() 模板方法 |
| LifecycleBase.java | 139 | start() 模板方法 |
| StandardServer.java | 931 | initInternal() 方法 |
| StandardServer.java | 849 | startInternal() 方法 |
| StandardServer.java | 505 | await() 方法 |

## JMX 注册

启动过程中会注册大量 MBean 用于监控和管理：

```java
// Server 级别
Catalina:type=Server
Catalina:type=StringCache
Catalina:type=MBeanFactory

// Service 级别
Catalina:type=Service,name=***

// Executor 线程池
Catalina:type=Executor,name=***

// Connector 连接器
Catalina:type=Connector,port=***

// RequestProcessor
Catalina:type=RequestProcessor,worker="http-nio-8080",name=HttpRequest*
```

## 错误处理

**启动失败处理：**
1. 配置文件解析失败 → 记录错误日志，退出
2. 端口绑定失败 → 抛出 LifecycleException
3. 类加载失败 → 记录错误，可能使用默认类加载器
4. 组件初始化失败 → 设置 FAILED 状态，触发 stop() 清理

**关闭流程：**
1. 通过 shutdown 端口接收关闭命令（默认 8005）
2. 或通过 shutdown hook（JVM 关闭时）
3. 按相反顺序停止组件：Context → Host → Engine → Service → Server

---

*文档位置：`./AnalysisDocs/tomcat-startup-process.md`*
*为 Tomcat 源码分析生成*
