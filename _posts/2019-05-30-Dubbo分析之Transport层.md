---
layout:     post
title:      Dubbo分析之Transport层
subtitle:   dubbo
date:       2019-05-30
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - dubbo
---

前言
--

本文分析Serialize层的上一层transport网络传输层，此层使用了现有的一些通讯开源框架(ex:netty,mina,grizzly)来做底层通讯，上文也做了简单介绍，本文将做更深入的了解；

Transporter类分析
--------------

dubbo为通讯框架提供了统一的bind和connet接口，方便进行管理和扩展，封装在接口类：Transporter中：

    @SPI("netty")
    public interface Transporter {
     
        @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
        Server bind(URL url, ChannelHandler handler) throws RemotingException;
     
        @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
        Client connect(URL url, ChannelHandler handler) throws RemotingException;
    }

提供了bind和connect接口，分别对应这服务器端和客户端，具体有哪些实现类，如下图所示：

以默认使用的netty框架为例，代码如下：

    public class NettyTransporter implements Transporter {
     
        public static final String NAME = "netty";
     
        @Override
        public Server bind(URL url, ChannelHandler listener) throws RemotingException {
            return new NettyServer(url, listener);
        }
     
        @Override
        public Client connect(URL url, ChannelHandler listener) throws RemotingException {
            return new NettyClient(url, listener);
        }
     
    }

具体的服务器端封装在NettyServer中，客户端封装在NettyClient；url参数是包含了xml配置的信息(包括：对外的接口，使用的协议，使用的序列化方式，使用的通讯框架等)，listener是一个Handler，在解码之后将数据交给它做后续的业务处理；对应以上的几种通讯开源框架，分别提供了对应的Transporter包括：NettyTransporter，NettyTransporter(netty4)，MinaTransporter以及GrizzlyTransporter，具体使用哪种类型的Transporter，在Transporters类中提供了getTransporter方法：

    public static Transporter getTransporter() {
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }

这里并没有像在获取具体serialization类一样，通过在url指定transporter参数，然后加载具体的transporter类，而是生成了一个动态的transporter，由此动态transporter去加载具体的类；  
因为Server端和Client可以分别设置成不同的通讯框架，一次获取唯一的Transporter不能满足此需求；具体的生成动态代码的方法在ExtensionLoader的createAdaptiveExtensionClassCode方法中，此处不在列出源码，在此展示一下默认生成的动态代码扩展类：

    package com.alibaba.dubbo.remoting;
     
    import com.alibaba.dubbo.common.extension.ExtensionLoader;
     
    public class Transporter$Adaptive implements com.alibaba.dubbo.remoting.Transporter {
        public com.alibaba.dubbo.remoting.Server bind(
            com.alibaba.dubbo.common.URL arg0,
            com.alibaba.dubbo.remoting.ChannelHandler arg1)
            throws com.alibaba.dubbo.remoting.RemotingException {
            if (arg0 == null) {
                throw new IllegalArgumentException("url == null");
            }
     
            com.alibaba.dubbo.common.URL url = arg0;
            String extName = url.getParameter("server",
                    url.getParameter("transporter", "netty"));
     
            if (extName == null) {
                throw new IllegalStateException(
                    "Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" +
                    url.toString() + ") use keys([server, transporter])");
            }
     
            com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class)
                                                                                                                       .getExtension(extName);
     
            return extension.bind(arg0, arg1);
        }
     
        public com.alibaba.dubbo.remoting.Client connect(
            com.alibaba.dubbo.common.URL arg0,
            com.alibaba.dubbo.remoting.ChannelHandler arg1)
            throws com.alibaba.dubbo.remoting.RemotingException {
            if (arg0 == null) {
                throw new IllegalArgumentException("url == null");
            }
     
            com.alibaba.dubbo.common.URL url = arg0;
            String extName = url.getParameter("client",
                    url.getParameter("transporter", "netty"));
     
            if (extName == null) {
                throw new IllegalStateException(
                    "Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" +
                    url.toString() + ") use keys([client, transporter])");
            }
     
            com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class)
                                                                                                                       .getExtension(extName);
     
            return extension.connect(arg0, arg1);
        }
    }

可以发现Server端可以通过transporter和server两个参数来设置扩展类，而且server参数设置的值是可以覆盖transporter参数的值，同理Client也类似；最后不管是bind()还是connet()都是通过ExtensionLoader的getExtension方法来获取具体的transporter类；同serialize层，相关的transporter也同样定义在META-INF/dubbo/internal/com.alibaba.dubbo.remoting.Transporter文件中：

    netty=com.alibaba.dubbo.remoting.transport.netty.NettyTransporter
    netty4=com.alibaba.dubbo.remoting.transport.netty4.NettyTransporter
    mina=com.alibaba.dubbo.remoting.transport.mina.MinaTransporter
    grizzly=com.alibaba.dubbo.remoting.transport.grizzly.GrizzlyTransporter

Server端和Client分析
----------------

### 1.Server端

在实例化具体的Server类时，会首先调用父类的构造器，进行参数初始化，同时调用bind()方法，启动服务器；父类AbstractServer构造器如下：

    public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
            super(url, handler);
            localAddress = getUrl().toInetSocketAddress();
     
            String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
            int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
            if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
                bindIp = NetUtils.ANYHOST;
            }
            bindAddress = new InetSocketAddress(bindIp, bindPort);
            this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
            this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
            try {
                doOpen();
                if (logger.isInfoEnabled()) {
                    logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
                }
            } catch (Throwable t) {
                throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
                        + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
            }
            //fixme replace this with better method
            DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
            executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
        }

主要从url获取启动参数包括：ip，port，accepts(可接受的连接数，0表示不受限制数量，默认为0)，idleTimeout等；然后调用doOpen方法通过具体的通讯框架绑定端口启动服务；已默认使用的Netty为例，查看doOpen()方法如下：

    protected void doOpen() throws Throwable {
           NettyHelper.setNettyLoggerFactory();
           ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
           ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
           ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
           bootstrap = new ServerBootstrap(channelFactory);
     
           final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
           channels = nettyHandler.getChannels();
           // https://issues.jboss.org/browse/NETTY-365
           // https://issues.jboss.org/browse/NETTY-379
           // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
           bootstrap.setOption("child.tcpNoDelay", true);
           bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
               @Override
               public ChannelPipeline getPipeline() {
                   NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                   ChannelPipeline pipeline = Channels.pipeline();
                   /*int idleTimeout = getIdleTimeout();
                   if (idleTimeout > 10000) {
                       pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
                   }*/
                   pipeline.addLast("decoder", adapter.getDecoder());
                   pipeline.addLast("encoder", adapter.getEncoder());
                   pipeline.addLast("handler", nettyHandler);
                   return pipeline;
               }
           });
           // bind
           channel = bootstrap.bind(getBindAddress());
       }

以上是常规的启动netty程序，需要指定编解码器，nettyHandler；编解码已经在上文中介绍过了，此处不在详细介绍，重点介绍nettyHandler；server端在数据经过解码之后就交给NettyHandler来处理，NettyHandler继承于Netty的SimpleChannelHandler类，重写了channelConnected，channelDisconnected，messageReceived，writeRequested以及exceptionCaught方法，基本上就是常规的几种操作：建立连接，断开连接，接收消息，发送消息，异常处理；看一下部分源码：

    @Override
       public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
           NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
           try {
               if (channel != null) {
                   channels.put(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()), channel);
               }
               handler.connected(channel);
           } finally {
               NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
           }
       }
     
       @Override
       public void channelDisconnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
           NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
           try {
               channels.remove(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()));
               handler.disconnected(channel);
           } finally {
               NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
           }
       }
        @Override
       public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
           NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
           try {
               handler.received(channel, e.getMessage());
           } finally {
               NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
           }
       }
     
       @Override
       public void writeRequested(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
           super.writeRequested(ctx, e);
           NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
           try {
               handler.sent(channel, e.getMessage());
           } finally {
               NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
           }
       }
     
       @Override
       public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
           NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
           try {
               handler.caught(channel, e.getCause());
           } finally {
               NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
           }
       }

将netty原生的channel包装成dubbo的NettyChannel，同时将NettyChannel保存在NettyChannel的内部静态变量channelMap中；这里的方法都统一调用了getOrAddChannel方法，先添加进去，最后在finally中判定channel是否已经关闭，如果关闭从channelMap中移除；中间部分调用了handler对应的方法，此处的handler就是在实例化时传入的NettyServer，NettyServer本身也是一个ChannelHandler，可以看一下channelHandler接口类：

    public interface ChannelHandler {
     
        void connected(Channel channel) throws RemotingException;
     
        void disconnected(Channel channel) throws RemotingException;
     
        void sent(Channel channel, Object message) throws RemotingException;
     
        void received(Channel channel, Object message) throws RemotingException;
     
        void caught(Channel channel, Throwable exception) throws RemotingException;
    }

具体的server类中也可以做一些处理，比如connected时判段是否超过accepts，如果超过拒绝连接；处理完之后交给实例化Server时传入的ChannelHandler处理，此类具体是在HeaderExchanger中被初始化的：

    public class HeaderExchanger implements Exchanger {
     
        public static final String NAME = "header";
     
        @Override
        public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
            return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
        }
     
        @Override
        public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
            return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
        }
     
    }

可以发现这里具体的ChannelHandler是DecodeHandler，注这里的Decode和Netty本身的decode不一样，Netty本身的decode在执行NettyHandler之前就执行解码了；后续的操作在Exchange层进行处理，本文暂时先不做介绍；

### 2.Client端

同样查看父类AbstractClient，构造方法如下：

    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
            super(url, handler);
     
            send_reconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, false);
     
            shutdown_timeout = url.getParameter(Constants.SHUTDOWN_TIMEOUT_KEY, Constants.DEFAULT_SHUTDOWN_TIMEOUT);
     
            // The default reconnection interval is 2s, 1800 means warning interval is 1 hour.
            reconnect_warning_period = url.getParameter("reconnect.waring.period", 1800);
     
            try {
                doOpen();
            } catch (Throwable t) {
                close();
                throw new RemotingException(url.toInetSocketAddress(), null,
                        "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                                + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
            }
            try {
                // connect.
                connect();
                if (logger.isInfoEnabled()) {
                    logger.info("Start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress() + " connect to the server " + getRemoteAddress());
                }
            } catch (RemotingException t) {
                if (url.getParameter(Constants.CHECK_KEY, true)) {
                    close();
                    throw t;
                } else {
                    logger.warn("Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                            + " connect to the server " + getRemoteAddress() + " (check == false, ignore and retry later!), cause: " + t.getMessage(), t);
                }
            } catch (Throwable t) {
                close();
                throw new RemotingException(url.toInetSocketAddress(), null,
                        "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                                + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
            }
     
            executor = (ExecutorService) ExtensionLoader.getExtensionLoader(DataStore.class)
                    .getDefaultExtension().get(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
            ExtensionLoader.getExtensionLoader(DataStore.class)
                    .getDefaultExtension().remove(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
        }

客户端需要提供重连机制，所以初始化的几个参数都和重连有关，send\_reconnect表示在发送消息时发现连接已经断开是否发起重连，reconnect\_warning\_period表示多久报一次重连警告，shutdown\_timeout表示连接服务器一直连接不上的超时时间；接下来就是调用doOpen()方法，同样已Netty为例：

    protected void doOpen() throws Throwable {
           NettyHelper.setNettyLoggerFactory();
           bootstrap = new ClientBootstrap(channelFactory);
           // config
           // @see org.jboss.netty.channel.socket.SocketChannelConfig
           bootstrap.setOption("keepAlive", true);
           bootstrap.setOption("tcpNoDelay", true);
           bootstrap.setOption("connectTimeoutMillis", getTimeout());
           final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
           bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
               @Override
               public ChannelPipeline getPipeline() {
                   NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                   ChannelPipeline pipeline = Channels.pipeline();
                   pipeline.addLast("decoder", adapter.getDecoder());
                   pipeline.addLast("encoder", adapter.getEncoder());
                   pipeline.addLast("handler", nettyHandler);
                   return pipeline;
               }
           });
       }

Netty客户端的常规代码，设置了和Server端相同的NettyHandler，decoder和encoder；下面重点看看connect方法：

    protected void connect() throws RemotingException {
            connectLock.lock();
            try {
                if (isConnected()) {
                    return;
                }
                initConnectStatusCheckCommand();
                doConnect();
                if (!isConnected()) {
                    throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                            + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                            + ", cause: Connect wait timeout: " + getTimeout() + "ms.");
                } else {
                    if (logger.isInfoEnabled()) {
                        logger.info("Successed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                                + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                                + ", channel is " + this.getChannel());
                    }
                }
                reconnect_count.set(0);
                reconnect_error_log_flag.set(false);
            } catch (RemotingException e) {
                throw e;
            } catch (Throwable e) {
                throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                        + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                        + ", cause: " + e.getMessage(), e);
            } finally {
                connectLock.unlock();
            }
        }

首先判定是否已经连接，如果连接直接return；接下来初始化连接状态检查器，定期检查channel是否连接，连接断开会进行重连操作，具体代码如下：

    private synchronized void initConnectStatusCheckCommand() {
            //reconnect=false to close reconnect
            int reconnect = getReconnectParam(getUrl());
            if (reconnect > 0 && (reconnectExecutorFuture == null || reconnectExecutorFuture.isCancelled())) {
                Runnable connectStatusCheckCommand = new Runnable() {
                    @Override
                    public void run() {
                        try {
                            if (!isConnected()) {
                                connect();
                            } else {
                                lastConnectedTime = System.currentTimeMillis();
                            }
                        } catch (Throwable t) {
                            String errorMsg = "client reconnect to " + getUrl().getAddress() + " find error . url: " + getUrl();
                            // wait registry sync provider list
                            if (System.currentTimeMillis() - lastConnectedTime > shutdown_timeout) {
                                if (!reconnect_error_log_flag.get()) {
                                    reconnect_error_log_flag.set(true);
                                    logger.error(errorMsg, t);
                                    return;
                                }
                            }
                            if (reconnect_count.getAndIncrement() % reconnect_warning_period == 0) {
                                logger.warn(errorMsg, t);
                            }
                        }
                    }
                };
                reconnectExecutorFuture = reconnectExecutorService.scheduleWithFixedDelay(connectStatusCheckCommand, reconnect, reconnect, TimeUnit.MILLISECONDS);
            }
        }

创建了一个Runnable，用来检测是否连接，如果连接断开，调用connect方法；定时调度交给ScheduledThreadPoolExecutor来执行；初始化之后就调用具体Client的doConnect操作，也是通讯框架的一些常规代码，此处不列出了；后续关于NettyChannel的介绍和Server端类似，不过多进行介绍；

总结
--

本文重点分析了dubbo架构中的transport层，具体围绕Transporter, Client, Server，ChannelHandler几个类展开，关于后续的处理将在exchange信息交换层；

示例代码地址
------

[https://github.com/ksfzhaohui...](https://github.com/ksfzhaohui/blog)  
[https://gitee.com/OutOfMemory...](https://gitee.com/OutOfMemory/blog)
