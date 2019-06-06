---
layout:     post
title:      Dubbo源码分析（三）Dubbo的服务引用Refer
subtitle:   dubbo
date:       2019-06-06
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - Dubbo
---

Dubbo源码分析（三）Dubbo的服务引用Refer
===========================

Dubbo的服务引用
==========

服务引用
----

先从Dubbo的配置文件看起

    <dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService"/>
    复制代码

源码入口： 根据上一篇说的，我们通过DubboNamespaceHandler类找到ReferenceBean类，在afterPropertiesSet()方法中我们找到关键代码getObject()  
进入ReferenceConfig类中的get()方法，这个get() 方法是一个同步方法，调用了init()方法  
我们看到init()方法中的最后一行代码ref = createProxy(map);我们从这这个方法开始分析：

    private T createProxy(Map<String, String> map) {
            ···
                if (urls.size() == 1) {
                    invoker = refprotocol.refer(interfaceClass, urls.get(0));
                } else {
                    List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                    URL registryURL = null;
                    for (URL url : urls) {
                        invokers.add(refprotocol.refer(interfaceClass, url));
                        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            registryURL = url; // 用了最后一个registry url
                        }
                    }
                    if (registryURL != null) { // 有 注册中心协议的URL
                        // 对有注册中心的Cluster 只用 AvailableCluster
                        URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                        invoker = cluster.join(new StaticDirectory(u, invokers));
                    } else { // 不是 注册中心的URL
                        invoker = cluster.join(new StaticDirectory(invokers));
                    }
                }
            }
            ···
            // 创建服务代理
            return (T) proxyFactory.getProxy(invoker);
        }
    复制代码

先看invoker = refprotocol.refer(interfaceClass, urls.get(0))这行代码  
此时的refprotocol= Protocol$Adatptive，进入refer方法：

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
            if (arg1 == null) throw new IllegalArgumentException("url == null");
            com.alibaba.dubbo.common.URL url = arg1;
            String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
            if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
            com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
            return extension.refer(arg0, arg1);
        }
    复制代码

此时的extName=registry，所以extension=ProtocolFilterWrapper(ProtocolListenerWrapper(RegistryProtocol)),我们直接进入RegistryProtocol.refer()方法中

    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
            Registry registry = registryFactory.getRegistry(url);
            if (RegistryService.class.equals(type)) {
                return proxyFactory.getInvoker((T) registry, type, url);
            }
    
            // group="a,b" or group="*"
            Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
            String group = qs.get(Constants.GROUP_KEY);
            if (group != null && group.length() > 0) {
                if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                        || "*".equals(group)) {
                    return doRefer(getMergeableCluster(), registry, type, url);
                }
            }
            return doRefer(cluster, registry, type, url);
    }
    复制代码

看到第二行代码Registry registry = registryFactory.getRegistry(url);这里从字面上理解应该是建立和注册中心的连接，这里的代码和服务端发布是一样的，这里跳过，继续往下走group，Dubbo里面是可以对服务进行分组，这里不影响主流程走向，我们跳过，看到最后一行代码，我们进入

    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
            RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
            directory.setRegistry(registry);
            directory.setProtocol(protocol);
            URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
            if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
                    && url.getParameter(Constants.REGISTER_KEY, true)) {
                registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                        Constants.CHECK_KEY, String.valueOf(false)));
            }
            directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                    Constants.PROVIDERS_CATEGORY
                            + "," + Constants.CONFIGURATORS_CATEGORY
                            + "," + Constants.ROUTERS_CATEGORY));
            return cluster.join(directory);
        }
    复制代码

先看subscribeUrl是啥，这里的url是consumer开头的url，看到registry.register()方法，这里是向注册中心去注册消费端信息，具体注册的节点是：/dubbo/com.alibaba.dubbo.demo.DemoService/consumers  
directory.subscribe(),这句代码一看就明白，应该是向注册中心订阅我们刚刚注册的地址，我们进入到这个方法里面去看看如果目录地址有变化，怎么通知，该做什么样的处理，最终的实现类是ZookeeperRegistry.doSubscribe()方法中，这里用到了模板方法，我们看到doSubscribe()方法中的这段代码notify(url, listener, urls)

     protected void notify(URL url, NotifyListener listener, List<URL> urls) {
           ···
            try {
                doNotify(url, listener, urls);
            } catch (Exception t) {
                // 将失败的通知请求记录到失败列表，定时重试
                Map<NotifyListener, List<URL>> listeners = failedNotified.get(url);
                ···
                listeners.put(listener, urls);
               ···
            }
        }
    复制代码

这里面执行了doNotify方法，如果执行失败，对应的通过定时策略去重试，继续进入doNotify方法

    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
            ···
            for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
                String category = entry.getKey();
                List<URL> categoryList = entry.getValue();
                categoryNotified.put(category, categoryList);
                saveProperties(url);
                listener.notify(categoryList);
            }
        }
    复制代码

这个是AbstractRegistry类中的方法，我们看到saveProperties方法，作用是把消费端注册的url信息缓存到本地

    registryCacheExecutor.execute(new SaveProperties(version));
    复制代码

然后通过线程池来定时缓存数据，我们继续看一下listener.notify(categoryList)这句代码,这里的listener是RegistryDirectory

    public synchronized void notify(List<URL> urls) {
            List<URL> invokerUrls = new ArrayList<URL>();
            List<URL> routerUrls = new ArrayList<URL>();
            List<URL> configuratorUrls = new ArrayList<URL>();
            for (URL url : urls) {
                String protocol = url.getProtocol();
                String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
                if (Constants.ROUTERS_CATEGORY.equals(category)
                        || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                    routerUrls.add(url);
                } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                        || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                    configuratorUrls.add(url);
                } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                    invokerUrls.add(url);
                } else {
                    logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
                }
            }
           ···
            // providers
            refreshInvoker(invokerUrls);
        }
    复制代码

看到最后一段代码refreshInvoker(invokerUrls)

     /**
         * 根据invokerURL列表转换为invoker列表。转换规则如下：
         * 1.如果url已经被转换为invoker，则不在重新引用，直接从缓存中获取，注意如果url中任何一个参数变更也会重新引用
         * 2.如果传入的invoker列表不为空，则表示最新的invoker列表
         * 3.如果传入的invokerUrl列表是空，则表示只是下发的override规则或route规则，需要重新交叉对比，决定是否需要重新引用。
         *
         * @param invokerUrls 传入的参数不能为null
         */
        // TODO: FIXME 使用线程池去刷新地址，否则可能会导致任务堆积
        private void refreshInvoker(List<URL> invokerUrls) {
            if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
                    && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
                this.forbidden = true; // 禁止访问
                this.methodInvokerMap = null; // 置空列表
                destroyAllInvokers(); // 关闭所有Invoker
            } else {
                this.forbidden = false; // 允许访问
                Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
                if (invokerUrls.size() == 0 && this.cachedInvokerUrls != null) {
                    invokerUrls.addAll(this.cachedInvokerUrls);
                } else {
                    this.cachedInvokerUrls = new HashSet<URL>();
                    this.cachedInvokerUrls.addAll(invokerUrls);//缓存invokerUrls列表，便于交叉对比
                }
                if (invokerUrls.size() == 0) {
                    return;
                }
                Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// 将URL列表转成Invoker列表
                Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // 换方法名映射Invoker列表
                // state change
                //如果计算错误，则不进行处理.
                if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
                    logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls.toString()));
                    return;
                }
                this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
                this.urlInvokerMap = newUrlInvokerMap;
                try {
                    destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // 关闭未使用的Invoker
                } catch (Exception e) {
                    logger.warn("destroyUnusedInvokers error. ", e);
                }
            }
        }
    复制代码

这段代码的最终目的是刷新urlInvokerMap缓存，并且关闭关闭未使用的Invoker 接下来我们继续cluster.join(directory)这个方法 ，此时的cluster=Cluster$Adaptive

    public com.alibaba.dubbo.rpc.Invoker join(com.alibaba.dubbo.rpc.cluster.Directory arg0) throws com.alibaba.dubbo.rpc.RpcException {
            if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument == null");
            if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
            String extName = url.getParameter("cluster", "failover");
            if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.cluster.Cluster) name from url(" + url.toString() + ") use keys([cluster])");
            com.alibaba.dubbo.rpc.cluster.Cluster extension = (com.alibaba.dubbo.rpc.cluster.Cluster)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
            return extension.join(arg0);
        }
    复制代码

此时extension=MockClusterWrapper(FaileOverCluster), 这里有一个Mock包装类，猜想一下，这个Mock应该是Dubbo的容错机制中用到的Mock，进入MockClusterWrapper.join方法

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
            return new MockClusterInvoker<T>(directory,
                    this.cluster.join(directory));
        }
    复制代码

这里new了一个MockClusterInvoker，进入FaileOverCluster.join方法

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
            return new FailoverClusterInvoker<T>(directory);
        }
    复制代码

这里new 了一个FailoverClusterInvoker，然后回到最初的ReferenceConfig.createProxy方法,看到最后一段代码return (T) proxyFactory.getProxy(invoker);这段代码的作用是创建服务代理，这里的invoker就是我们刚刚new的MockClusterInvoker，这里的proxyFactory=ProxyFactory$Adaptive，直接贴结果，进入StubProxyFactoryWrapper.getProxy

    public <T> T getProxy(Invoker<T> invoker) throws RpcException {
            T proxy = proxyFactory.getProxy(invoker);
            if (GenericService.class != invoker.getInterface()) {
                String stub = invoker.getUrl().getParameter(Constants.STUB_KEY, invoker.getUrl().getParameter(Constants.LOCAL_KEY));
                if (ConfigUtils.isNotEmpty(stub)) {
                    Class<?> serviceType = invoker.getInterface();
                    if (ConfigUtils.isDefault(stub)) {
                        if (invoker.getUrl().hasParameter(Constants.STUB_KEY)) {
                            stub = serviceType.getName() + "Stub";
                        } else {
                            stub = serviceType.getName() + "Local";
                        }
                    }
                    try {
                        Class<?> stubClass = ReflectUtils.forName(stub);
                        if (!serviceType.isAssignableFrom(stubClass)) {
                            throw new IllegalStateException("The stub implemention class " + stubClass.getName() + " not implement interface " + serviceType.getName());
                        }
                        try {
                            Constructor<?> constructor = ReflectUtils.findConstructor(stubClass, serviceType);
                            proxy = (T) constructor.newInstance(new Object[]{proxy});
                            //export stub service
                            URL url = invoker.getUrl();
                            if (url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT)) {
                                url = url.addParameter(Constants.STUB_EVENT_METHODS_KEY, StringUtils.join(Wrapper.getWrapper(proxy.getClass()).getDeclaredMethodNames(), ","));
                                url = url.addParameter(Constants.IS_SERVER_KEY, Boolean.FALSE.toString());
                                try {
                                    export(proxy, (Class) invoker.getInterface(), url);
                                } catch (Exception e) {
                                    LOGGER.error("export a stub service error.", e);
                                }
                            }
                        } catch (NoSuchMethodException e) {
                            throw new IllegalStateException("No such constructor \"public " + stubClass.getSimpleName() + "(" + serviceType.getName() + ")\" in stub implemention class " + stubClass.getName(), e);
                        }
                    } catch (Throwable t) {
                        LOGGER.error("Failed to create stub implemention class " + stub + " in consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", cause: " + t.getMessage(), t);
                        // ignore
                    }
                }
            }
            return proxy;
        }
    复制代码

我们先看第一行代码 T proxy = proxyFactory.getProxy(invoker);  
这里的proxyFactory=JavassitProxyFactory，我们首先进入的是AbstractProxyFactory.getProxy方法，这里又是一个模版方法，

    public <T> T getProxy(Invoker<T> invoker) throws RpcException {
            Class<?>[] interfaces = null;
            String config = invoker.getUrl().getParameter("interfaces");
            ···
            if (interfaces == null) {
                interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
            }
            return getProxy(invoker, interfaces);
        }
    复制代码

进入JavassitProxyFactory.getProxy方法，

    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
            return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
    复制代码

这里传入的interfaces=\[interface com.alibaba.dubbo.demo.DemoService, interface com.alibaba.dubbo.rpc.service.EchoService\]  
再进入new InvokerInvocationHandler(invoker)，这里初始化一个InvokerInvocationHandler对象，我们看下这个对象

    public class InvokerInvocationHandler implements InvocationHandler {
    
        private final Invoker<?> invoker;
    
        public InvokerInvocationHandler(Invoker<?> handler) {
            this.invoker = handler;
        }
    
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            String methodName = method.getName();
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(invoker, args);
            }
            if ("toString".equals(methodName) && parameterTypes.length == 0) {
                return invoker.toString();
            }
            if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
                return invoker.hashCode();
            }
            if ("equals".equals(methodName) && parameterTypes.length == 1) {
                return invoker.equals(args[0]);
            }
            return invoker.invoke(new RpcInvocation(method, args)).recreate();
        }
    
    }
    复制代码

这里用了JDK自带的动态代理Proxy类和InvocationHandler接口，到这里proxy代理类创建完成。