---
layout: post
title: 'xdag netty模型'
date: 2020-9-9
author: Warmingzzz
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: netty

---

>netty模型第一篇

## xdagServer

​	封装的许多的类先不看

​	start线程：

```java
				// 主线程组, 用于接受客户端的连接，但是不做任何处理，跟老板一样，不做事        
				EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // 从线程组, 老板线程组会把任务丢给他，让手下线程组去做任务
        EventLoopGroup workerGroup = new NioEventLoopGroup();
					// netty服务器的创建, 辅助工具类，用于服务器通道的一系列配置
            ServerBootstrap b = new ServerBootstrap();
            //绑定两个线程组
            b.group(bossGroup, workerGroup);
```

close线程：

```java
								channelFuture.channel().close().sync();
                workerGroup.shutdownGracefully();
                bossGroup.shutdownGracefully();
                workerGroup.terminationFuture().sync();
```

把start的开启的workerGroup和bossGroup关闭了



## XdagClient



先看client类

```java
    public XdagClient(Config config) {
        this.config = config;
        this.ip = config.getNodeIp();
        this.port = config.getNodePort();
        this.workerGroup = new NioEventLoopGroup(0, factory);
        log.debug("XdagClient nodeId:" + getNode().getHexId());
    }
```

初始化了client的ip port 还定义了workerGroup工作线程组

connect类

```java
    public void connect(String host, int port, XdagChannelInitializer xdagChannelInitializer) {
        try {
            //以非同步的方式连接host port
            f = connectAsync(host, port, xdagChannelInitializer);
            f.sync();
        } catch (Exception e) {
            if (e instanceof IOException) {
                log.debug(
                        "XdagClient: Can't connect to " + host + ":" + port + " (" + e.getMessage() + ")");
                log.debug("XdagClient.connect(" + host + ":" + port + ") exception:");
            } else {
                log.error("Exception:", e);
            }
        }
    }
```

主要是connectAsync方法，下面会进行介绍

```java
    public ChannelFuture connectAsync(
            String host, int port, XdagChannelInitializer xdagChannelInitializer) {
        // xdagListener.trace("Connecting to: " + host + ":" + port);
        Bootstrap b = new Bootstrap();
        b.group(workerGroup);
        b.channel(NioSocketChannel.class);
        // b.option(ChannelOption.SO_KEEPALIVE, true);
        b.option(ChannelOption.MESSAGE_SIZE_ESTIMATOR, DefaultMessageSizeEstimator.DEFAULT);
        b.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, config.getConnectionTimeout());
        b.remoteAddress(host, port);
        b.handler(xdagChannelInitializer);
        // Start the client.
        return b.connect();
    }
```

这个类也是一些netty的常规写法，返回一个connect

```java
    public void close() {
        log.debug("Shutdown XdagClient");
        workerGroup.shutdownGracefully();
        workerGroup.terminationFuture().syncUninterruptibly();
    }
```

关闭workerGroup线程组

## XdagChannel

（这个文件应该需要改）

初始化init

```java	
    public void init(
            ChannelPipeline pipeline,
            Kernel kernel,
            boolean isServer,
            InetSocketAddress inetSocketAddress) {
      this.config = kernel.getConfig();
        this.inetSocketAddress = inetSocketAddress;
        this.handshakeHandler = new XdagHandshakeHandler(kernel, config, this);
        handshakeHandler.setServer(isServer);
        pipeline.addLast("handshakeHandler", handshakeHandler);
        this.msgQueue = new MessageQueue(this);
        this.messageCodec = new MessageCodes(this);
        this.blockHandler = new XdagBlockHandler(this);
        this.xdagHandlerFactory = new XdagHandlerFactoryImpl(kernel, this);
    }
```

各种初始化，this 是指本类channel

逻辑发送公钥，私钥，区块



### ConnectionLimitHandler

这个类还没有用到

计算channel的连接个数



### Node

用host 和port 定义了一个Node，如果id不为空就直接传，为空就输入一个随机数

### NodeManager

```java
    /** start the node manager */
    public synchronized void start() {
        if (!isRunning) {
            // addNodes(getSeedNodes(config.getWhiteListDir()));
            addNodes(getSeedNodes(netDBManager.getWhiteDB()));

            // every 0.5 seconds, delayed by 1 seconds (kernel boot up)
            connectFuture = exec.scheduleAtFixedRate(this::doConnect, 1000, 500, TimeUnit.MILLISECONDS);
            // every 100 seconds, delayed by 5 seconds (public IP lookup)
            fetchFuture = exec.scheduleAtFixedRate(this::doFetch, 5, 100, TimeUnit.SECONDS);

            isRunning = true;
            log.debug("Node manager started");
        }
    }
```

Start 建立两个定时线程connectFuture和fetchFuture

```java
    public synchronized void stop() {
        if (isRunning) {
            connectFuture.cancel(true);
            fetchFuture.cancel(false);

            isRunning = false;
            exec.shutdown();
            log.debug("Node manager stop...");
        }
    }
```

Stop 关闭这两个定时线程

doFetch更新种子节点（种子节点是啥）

```java
    /** from net update seed nodes */
    protected void doFetch() {
        log.debug("Do fetch");
        if (config.isEnableRefresh()) {
            netDBManager.refresh();
        }
        // 从白名单获得新节点
        addNodes(getSeedNodes(netDBManager.getWhiteDB()));
        // 从netdb获取新节点
        addNodes(getSeedNodes(netDBManager.getNetDB()));
        log.debug("node size:" + deque.size());
    }
```

doConnect 定义了两个，一个无参数，一个有节点参数，无参数的节点有deque.poll出去，有参数节点的直接连接

```java
public void doConnect() {
    Set<InetSocketAddress> activeAddress = channelMgr.getActiveAddresses();
    Node node;
    while ((node = deque.pollFirst()) != null && channelMgr.size() < config.getMAX_CHANNELS()) {
        Long lastCon = lastConnect.getIfPresent(node);
        long now = System.currentTimeMillis();

        if (!client.getNode().equals(node)
                && !(Objects.equals(node.getHost(), client.getNode().getHost())
                        && node.getPort() == client.getNode().getPort())
                && !activeAddress.contains(node.getAddress())
                && (lastCon == null || lastCon + RECONNECT_WAIT < now)) {
            XdagChannelInitializer initializer = new XdagChannelInitializer(kernel, false, node);
            client.connect(node.getHost(), node.getPort(), initializer);
            lastConnect.put(node, now);
            break;
        }
    }
}

public void doConnect(String ip, int port) {
    Node remotenode = new Node(ip, port);
    if (!client.getNode().equals(remotenode) && !channelMgr.containsNode(remotenode)) {
        XdagChannelInitializer initializer = new XdagChannelInitializer(kernel, false, remotenode);
        client.connect(ip, port, initializer);
    }
}
```

getActiveNode 返回一个map<node ,lastConnect time>

### NodeStat

用来统计节点的个数

### meseeage

消息部分应该是不用改的 先简单看看

### NetDBManager

更新白名单，whiteDB

### XdagChannelManager

```java
    public XdagChannelManager(Kernel kernel) {
        this.kernel = kernel;

        // Resending new blocks to network in loop
        this.blockDistributeThread = new Thread(this::newBlocksDistributeLoop, "NewSyncThreadBlocks");
        blockDistributeThread.start();
    }
```

初始化的时候定一个循环线程发送区块

remove（XdagChannel）移除某个特定的连接

### Xdag03

对不同的区块逻辑处理，应该不用改

### xdagBolckHandler

待看