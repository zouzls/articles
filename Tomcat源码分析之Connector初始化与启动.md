---
title: Tomcat源码分析之Connector初始化与启动
date: 2016-07-20 16:06:35
tags: [Tomcat,源码分析,环境搭建]
categories: Tomcat
---
Tomcat主要由两大核心组件，一个是connector,一个是container。connector负责的是底层的网络通信的实现，而container负责的是上层servlet业务的实现。一个应用服务器的性能很大程度上取决于网络通信模块的实现，因此connector对于tomcat而言是重中之重。源码看到Connector的时候，确实看的比较费劲，里面错综复杂的类结构和继承关系，确实头昏脑涨，下面我尝试用清晰一点的脉络去理解Connector。
<!--more-->
### Connector配置
在{源码根目录}/conf路径下的server.xml文件中，service内部可以看到Connector的配置：
```
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```
尤其是HTTP/1.1协议的配置，后面在Connector初始化的过程中，我们可以看到它的作用（下面的讨论都是基于HTTP/1.1之上，不再说明）。
### 主要类和接口
下面类会比较常见，而且关系比较错综复杂，提前心中有数比较好，如下：
- Http11Protocol
1、继承关系：Http11Protocol - AbstractProtocol - ProtocolHandler。
2、Http11Protocol是ProtocolHandler的的最终实现，并且Connector对于请求的真正的处理是由Http11Protocol这个boss搞定的，这个boss负责有三个重要的实例，分别是JIoEndpoint、Http11ConnectionHandler和CoyoteAdapter，他们各司其职。如图：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/1358483637_2091.png)
- JIoEndpoint
1、继承关系：JIoEndpoint - AbstractEndpoint<Socket>。
2、在Http11Protocol的构造函数中被初始化。
3、完成端口的绑定，对请求进行监听，并将获取的socket交给Http11ConnectionHandler来process。
- Http11ConnectionHandler
1、继承关系：Http11ConnectionHandler - AbstractConnectionHandler -  Handler。
2、在Http11Protocol的构造函数中被初始化，并被设置为JIoEndpoint的cHandler。
3、作为Http11Protocol的内部类，处理socket任务。
- CoyoteAdapter
1、继承关系：CoyoteAdapter - Adapter
2、在Connector的init方法中被初始化，并被设置为protocolHandler的adapter。
3、作为Connector和Container之间的桥梁，可以理解为有两个作用，一个是将Connector的请求交给Container，同时隔离了两边请求；二个是可以作为拓展。
![](http://oaewlsdmg.bkt.clouddn.com/blog%2Fimage%2F20150426172846043.png)

【注】：JIoEndpoint和Http11ConnectionHandler是在Http11Protocol的构造函数中被初始化，代码如下：
```java
    public Http11Protocol() {
        endpoint = new JIoEndpoint();
        cHandler = new Http11ConnectionHandler(this);
        ((JIoEndpoint) endpoint).setHandler(cHandler);
        setSoLinger(Constants.DEFAULT_CONNECTION_LINGER);
        setSoTimeout(Constants.DEFAULT_CONNECTION_TIMEOUT);
        setTcpNoDelay(Constants.DEFAULT_TCP_NO_DELAY);
    }
```
JIoEndpoint设置了Handler为Http11ConnectionHandler，至于后面怎么用，会继续分析。下面分析Connector是如何初始化和启动的。
### Connector初始化
#### Connector的构造函数
构造函数里面主要是搞定Http11Protocol这个超级Handler的实例化，Connector的通信的实际任务就是由该对象一手策划完成，代码如下：
```java
    protected String protocolHandlerClassName =
        "org.apache.coyote.http11.Http11Protocol";
    ...
    public Connector(String protocol) {
        //step1 设置协议变量
        setProtocol(protocol);
        // Instantiate protocol handler
        try {
            //step2 根据协议动态加载协议类和实例化该对象
            Class<?> clazz = Class.forName(protocolHandlerClassName);
            this.protocolHandler = (ProtocolHandler) clazz.newInstance();
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        }
    }
    //setProtocol方法来确定class名称
    public void setProtocol(String protocol) {
        if ("HTTP/1.1".equals(protocol)) {//配置文件中已经配置
            setProtocolHandlerClassName
                ("org.apache.coyote.http11.Http11Protocol");
        } 
    }
```
#### Connector的initInternal方法
Connector的initInternal()初始化方法被调用主要是设置当前Connector的适配器并且完成Http11Protocol的初始化，如下：
```java
    protected void initInternal() throws LifecycleException {
        ...
        //step1 初始化一个适配器（Connector和Container的桥梁）
        adapter = new CoyoteAdapter(this);
        //并将adapter设置Handler的适配器
        protocolHandler.setAdapter(adapter);

        ...

        try {
            //step2 协议初始化
            protocolHandler.init();//
        } catch (Exception e) {
            throw new LifecycleException
                (sm.getString
                 ("coyoteConnector.protocolHandlerInitializationFailed"), e);
        }

        //step3 Initialize mapper listener
        mapperListener.init();
    }
```
【注】：这里先对CoyoteAdapter做一个简单的解释，Connector监听到底层的请求之后，
会中间经过一系列准备工作之后，创建org.apache.coyote.Request和org.apache.coyote.Response（注意Request和Response所属的范围），交给CoyoteAdapter的service(org.apache.coyote.Request req,org.apache.coyote.Response res)方法，在这个方法内部会进行适配，也可以理解为转换，变成org.apache.catalina.connector.Request和org.apache.catalina.connector.Response，然后交给Container处理，service方法的代码可以简单了解如下(部分省略)：
```java
    public void service(org.apache.coyote.Request req,
                        org.apache.coyote.Response res)
        throws Exception {
        ...
        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);
        if (request == null) {
            ...
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);
            ...
        }
        ...
        // Calling the container
        connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);//请求最终交给Container
        ...
    }
```

#### JIoEndpoint的init方法
protocolHandler.init()方法会调用Http11Protocol祖父类（AbstractProtocol）的init方法，然后接着调用JIoEndpoint抽象父类（AbstractEndpoint）的init，最终是由IoEndpoint的band方法完成端口的绑定。我们可以看到AbstractEndpoint的init()方法，如下：
```java
    public abstract void bind() throws Exception;//定义了抽象的绑定操作
    ...
    public final void init() throws Exception {
        testServerCipherSuitesOrderSupport();
        if (bindOnInit) {
            bind();
            bindState = BindState.BOUND_ON_INIT;
        }
    }
```
如果在eclipse中打开bind()方法会看到三个不同类的实现，毫无疑问应该是第二个实现JIoEndpoint，如下图：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E5%9B%BE%E7%89%8720160721100955.png)
在这个过程中我们可以知道设计者的心思，设计者在AbstractProtocol和AbstractEndpoint中分别将各种共用的关键的方法和变量进行了定义，这样体现了抽象类的作用。
在JIoEndpoint中bind方法代码实现如下：
```java
    @Override
    public void bind() throws Exception {

        // Initialize thread count defaults for acceptor
        if (acceptorThreadCount == 0) {
            acceptorThreadCount = 1;
        }
        // Initialize maxConnections
        if (getMaxConnections() == 0) {
            // User hasn't set a value - use the default
            setMaxConnections(getMaxThreadsExecutor(true));
        }
        ...
        serverSocketFactory =
                    handler.getSslImplementation().getServerSocketFactory(this);
        serverSocket = serverSocketFactory.createSocket(getPort(),
                            getBacklog());  
        ...
        }
    }
```
### Connector启动
#### Connector的startInternal方法
这个过程跟Connector的初始化过程前半部分相似，代码实现如下：
```java
    @Override
    protected void startInternal() throws LifecycleException {
        ...
        try {
            protocolHandler.start();
        } catch (Exception e) {
            ...
        }

        mapperListener.start();
    }
```
毫无疑问是调用protocolHandler的start方法，由Http11Protocol的祖父类AbstractProtocol的start方法，代码如下：
```java
    @Override
    public void start() throws Exception {
        ...
        try {
            endpoint.start();
        } catch (Exception ex) {
            ....
        }
    }
```
父类抽象的实现了start方法，接下来调用endpoint.start()，目前为止与上文中的endpoint.init()方法如出一辙，如下AbstractEndpoint中的start方法：
```
    public final void start() throws Exception {
        if (bindState == BindState.UNBOUND) {
            bind();
            bindState = BindState.BOUND_ON_START;
        }
        startInternal();
    }
```
#### JIoEndpoint中的startInternal方法
同样startInternal()也有三个实现，进入JIoEndpoint中，则有：
```java
    @Override
    public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;

            // Create worker collection
            if (getExecutor() == null) {
                createExecutor();
            }

            initializeConnectionLatch();

            startAcceptorThreads();//这个就是重头戏啦

            // Start async timeout thread
            Thread timeoutThread = new Thread(new AsyncTimeout(),
                    getName() + "-AsyncTimeout");
            timeoutThread.setPriority(threadPriority);
            timeoutThread.setDaemon(true);
            timeoutThread.start();
        }
    }
```
#### JIoEndpoint的startAcceptorThreads方法
中间的startAcceptorThreads()就是开启接受请求线程咯，我们进去一睹庐山真面看看，进去之后会进入AbstractEndpoint中，又来到这个类了，代码如下：
```java
    protected final void startAcceptorThreads() {
        int count = getAcceptorThreadCount();
        acceptors = new Acceptor[count];//此处的Acceptor是个接口

        for (int i = 0; i < count; i++) {
            acceptors[i] = createAcceptor();//此处根据协议不同创建不同的
            String threadName = getName() + "-Acceptor-" + i;
            acceptors[i].setThreadName(threadName);
            Thread t = new Thread(acceptors[i], threadName);
            t.setPriority(getAcceptorThreadPriority());
            t.setDaemon(getDaemon());
            t.start();
        }
    }
    ...
    protected abstract Acceptor createAcceptor();//AbstractEndpoint已经定义好了这个抽象方法
```
new Acceptor[count]中的Acceptor其实就是AbstractEndpoint的一个抽象的内部类，看看它大概是什么样的吧：
```java
    public abstract static class Acceptor implements Runnable {
        ....
    }
```
一看就是个线程对不对，但是只是定义了一些变量，没有定义run() 方法，所以我们要看createAcceptor()中是如何实现的，进入由JIoEndpoint中，如下：
```java
    @Override
    protected AbstractEndpoint.Acceptor createAcceptor() {
        return new Acceptor();
    }
```
尼玛搞了半天，以为快水落石出了，结果你只是又扔出来一个Acceptor，好吧看看那么这个有何不同，进去看看：
```java
    protected class Acceptor extends AbstractEndpoint.Acceptor {

        @Override
        public void run() {
            // Loop until we receive a shutdown command
            while (running) {
                ...
                try {
                    ...
                    Socket socket = null;
                    socket = serverSocketFactory.acceptSocket(serverSocket);//获取连接

                    // Configure the socket
                    if (running && !paused && setSocketOptions(socket)) {
                        // Hand this socket off to an appropriate processor
                        if (!processSocket(socket)) {//关键处理
                            ...
                        }
                    } 
                } 
            }
        }
    }
```
从processSocket(socket)中进去看到socket被包装了起来，交给了SocketProcessor，该类是JIoEndpoint的内部类，作为专门处理socket的线程。
#### Http11ConnectionHandler的process方法
接下来就可以看到真正处理socket的东西了，最前面前面我们提到过，JIoEndpoint和Http11ConnectionHandler是在Http11Protocol的构造函数中被初始化，JIoEndpoint同时设置了Http11ConnectionHandler作为Handler。所以，回头来看，我们知道JIoEndpoint在接收到socket请求之后，实际是经过一定的准备工作，然后将socket交给了Http11ConnectionHandler这个去处理了。代码如下：
```java
    /**
     * Handling of accepted sockets.
     */
    protected Handler handler = null;
    public void setHandler(Handler handler ) { this.handler = handler; }
    ...
    protected class SocketProcessor implements Runnable {
        ...
        @Override
        public void run() {
            if ((state != SocketState.CLOSED)) {
                if (status == null) {
                    state = handler.process(socket, SocketStatus.OPEN_READ);
                } else {
                    state = handler.process(socket,status);
                }
            }
        }
    }
```
由源码可知，通过Http11ConnectionHandler的父类AbstractConnectionHandler定义的process方法（设计者可能是为了方便拓展），然后又用Http11Processor的父类的AbstractHttp11Processor的process方法，才调用了埋伏已久的adapter.service(req,resp)方法，等这一刻等的有点久了。代码如下：
```java
 public SocketState process(SocketWrapper<S> socketWrapper)
        throws IOException {
        
        adapter.service(request, response);
    }
```
至此，Connector的初始化和启动分析的差不多了，并将这个过程中创建的Request和Response交给了adapter这个桥梁，至于service方法内部做了哪些工作，在前面有分析，如果看到这儿，再回头看看文章开始的那几个类，大概整个流程差不多了然于胸了。

### 参考资料
[tomcat源码分析-Connector初始化与启动](http://wely.iteye.com/blog/2295171)
[tomcat6中的请求流程 - CSDN.NET](http://blog.csdn.net/liweisnake/article/details/8513295)
[tomcat connector学习笔记 ](http://www.it610.com/article/2301453.htm)
[连接器与容器的桥梁——CoyoteAdapter](http://blog.csdn.net/wangyangzhizhou/article/details/45290061)


-EOF-

