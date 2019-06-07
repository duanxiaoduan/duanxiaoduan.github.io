---
layout:     post
title:      Dubbo源码分析（四）Dubbo调用链-消费端（集群容错机制）
subtitle:   dubbo
date:       2019-06-07
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - Dubbo
---

Dubbo源码分析（四）Dubbo调用链-消费端（集群容错机制）
================================

这篇来分析Dubbo消费端调用服务端的过程，先看一张调用链的整体流程图

下面蓝色部分是消费端的调用过程，大致过程分为Proxy-->Filter-->Invoker-->Directory-->LoadBalance-->Filter-->Invoker-->Client  
接着我们再来看一张集群容错的架构图，在集群调用失败时，Dubbo 提供了多种容错方案，缺省为 failover 重试。

我们对比一下两张图可以发现消费端的消费过程其实主要就是Dubbo的集群容错过程，下面开始分析源码

源码入口
----

    public class Consumer {
    
        public static void main(String[] args) {
            ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-consumer.xml"});
            context.start(); 
            //这是服务引用的源码入口，获取代理类
            DemoService demoService = (DemoService) context.getBean("demoService"); // 获取远程服务代理
            //这是服务调用链的源码入口
            String hello = demoService.sayHello("world"); // 执行远程方法
            System.out.println(hello); // 显示调用结果
        }
    }
    复制代码

我们知道，demoService是一个proxy代理类，执行demoService.sayHello方法，其实是调用InvokerInvocationHandler.invoke方法，应该还记得proxy代理类中我们new了一个InvokerInvocationHandler实例

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            String methodName = method.getName();
            Class<?>[] parameterTypes = method.getParameterTypes();
            ···
            return invoker.invoke(new RpcInvocation(method, args)).recreate();
        }
    复制代码

这里的invoker=MockClusterWrapper(FaileOverCluster)，new RpcInvocation是将所有请求参数都会转换为RpcInvocation，接下来我们进入集群部分

进入集群
----

首先进入MockClusterWrapper.invoke方法

    public Result invoke(Invocation invocation) throws RpcException {
            Result result = null;
            String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
            if (value.length() == 0 || value.equalsIgnoreCase("false")) {
                //no mock
                result = this.invoker.invoke(invocation);
            } else if (value.startsWith("force")) {
                if (logger.isWarnEnabled()) {
                    logger.info("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
                }
                //force:direct mock
                result = doMockInvoke(invocation, null);
            } else {
                //fail-mock
                try {
                    result = this.invoker.invoke(invocation);
                } catch (RpcException e) {
                    if (e.isBiz()) {
                        throw e;
                    } else {
                        if (logger.isWarnEnabled()) {
                            logger.info("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                        }
                        result = doMockInvoke(invocation, e);
                    }
                }
            }
            return result;
        }
    复制代码

因为我们的配置文件中没有配置mock，所以直接进入FaileOverCluster.invoke方法，其实是进入父类AbstractClusterInvoker.invoke方法，看一下这个方法

    public Result invoke(final Invocation invocation) throws RpcException {
    
            checkWhetherDestroyed();
    
            LoadBalance loadbalance;
    
            List<Invoker<T>> invokers = list(invocation);
            if (invokers != null && invokers.size() > 0) {
                loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                        .getMethodParameter(invocation.getMethodName(), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
            } else {
                loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
            }
            RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
            return doInvoke(invocation, invokers, loadbalance);
        }
    复制代码

先看下list(invocation)方法

    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
            List<Invoker<T>> invokers = directory.list(invocation);
            return invokers;
        }
    复制代码

进入目录查找
------

我们看下directory.list(invocation)方法，这里directory=RegistryDirectory,进入RegistryDirectory.list方法

    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
           ···
            List<Invoker<T>> invokers = doList(invocation);
            List<Router> localRouters = this.routers; // local reference
            if (localRouters != null && localRouters.size() > 0) {
                for (Router router : localRouters) {
                    try {
                        if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
                            invokers = router.route(invokers, getConsumerUrl(), invocation);
                        }
                    ···
            return invokers;
        }
    复制代码

再进入doList方法：

    public List<Invoker<T>> doList(Invocation invocation) {
            if (forbidden) {
                // 1. 没有服务提供者 2. 服务提供者被禁用
                throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
                    "No provider available from registry " + getUrl().getAddress() + " for service " + ··
            }
            List<Invoker<T>> invokers = null;
            Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
            ···
            return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
        }
    复制代码

从this.methodInvokerMap里面查找一个 List<Invoker>返回

进入路由
----

接着进入路由，返回到AbstractDirectory.list方法，进入router.route()方法，此时的router=MockInvokersSelector

    public <T> List<Invoker<T>> route(final List<Invoker<T>> invokers,
                                          URL url, final Invocation invocation) throws RpcException {
            if (invocation.getAttachments() == null) {
                return getNormalInvokers(invokers);
            } else {
                String value = invocation.getAttachments().get(Constants.INVOCATION_NEED_MOCK);
                if (value == null)
                    return getNormalInvokers(invokers);
                else if (Boolean.TRUE.toString().equalsIgnoreCase(value)) {
                    return getMockedInvokers(invokers);
                }
            }
            return invokers;
        }
    复制代码

进入getMockedInvokers()方法，这个方法就是将传入的invokers和设置的路由规则匹配，获得符合条件的invokers返回

    private <T> List<Invoker<T>> getNormalInvokers(final List<Invoker<T>> invokers) {
            if (!hasMockProviders(invokers)) {
                return invokers;
            } else {
                List<Invoker<T>> sInvokers = new ArrayList<Invoker<T>>(invokers.size());
                for (Invoker<T> invoker : invokers) {
                    if (!invoker.getUrl().getProtocol().equals(Constants.MOCK_PROTOCOL)) {
                        sInvokers.add(invoker);
                    }
                }
                return sInvokers;
            }
        }
    复制代码

进入负载均衡
------

继续回到AbstractClusterInvoker.invoke方法，

    public Result invoke(final Invocation invocation) throws RpcException {
            ···
            if (invokers != null && invokers.size() > 0) {
                loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                        .getMethodParameter(invocation.getMethodName(), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
            } 
            ···
            return doInvoke(invocation, invokers, loadbalance);
        }
    复制代码

这里先获取loadbalance扩展点适配器LoadBalance$Adaptive，默认是RandomLoadBalance随机负载，所以loadbalance=RandomLoadBalance，进入FailoverClusterInvoker.doInvoke方法

    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
            ···
                Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
                invoked.add(invoker);
                RpcContext.getContext().setInvokers((List) invoked);
                try {
                    Result result = invoker.invoke(invocation);
                    if (le != null && logger.isWarnEnabled()) {
                        logger.warn("Although retry the method " + invocation.getMethodName()
                               ····);
                    }
                    return result;
                } catch (RpcException e) {
                   ···
                } finally {
                    providers.add(invoker.getUrl().getAddress());
                }
            }
           ···
        }
    复制代码

进入select(loadbalance, invocation, copyinvokers, invoked)方法，最终进入RandomLoadBalance.doSelect()方法，这个随机算法中可以配置权重，Dubbo根据权重最终选择一个invoker返回

远程调用
----

回到 FaileOverCluster.doInvoke方法中，执行Result result = invoker.invoke(invocation);此时的invoker就是负载均衡选出来的invoker=RegistryDirectory$InvokerDelegete, 走完8个Filter，我们进入DubboInvoker.doInvoke()方法

    protected Result doInvoke(final Invocation invocation) throws Throwable {
            RpcInvocation inv = (RpcInvocation) invocation;
            final String methodName = RpcUtils.getMethodName(invocation);
            inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
            inv.setAttachment(Constants.VERSION_KEY, version);
    
            ExchangeClient currentClient;
            if (clients.length == 1) {
                currentClient = clients[0];
            } else {
                currentClient = clients[index.getAndIncrement() % clients.length];
            }
            try {
                boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
                boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
                int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
                if (isOneway) {
                    boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                    currentClient.send(inv, isSent);
                    RpcContext.getContext().setFuture(null);
                    return new RpcResult();
                } else if (isAsync) {
                    ResponseFuture future = currentClient.request(inv, timeout);
                    RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                    return new RpcResult();
                } else {
                    RpcContext.getContext().setFuture(null);
                    return (Result) currentClient.request(inv, timeout).get();
                }
            } catch (TimeoutException e) {
                throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
            } catch (RemotingException e) {
                throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
            }
        }
    复制代码

这里为什么DubboInvoker是个protocol? 因为RegistryDirectory.refreshInvoker.toInvokers： protocol.refer，我们进入currentClient.request(inv, timeout).get()方法，进入HeaderExchangeChannel.request方法，进入NettyChannel.send方法，

    public void send(Object message, boolean sent) throws RemotingException {
            super.send(message, sent);
    
            boolean success = true;
            int timeout = 0;
            try {
                ChannelFuture future = channel.write(message);
                if (sent) {
                    timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
                    success = future.await(timeout);
                }
               ···
        }
    复制代码

这里最终执行ChannelFuture future = channel.write(message)，通过Netty发送网络请求