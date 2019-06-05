---
layout:     post
title:      Dubbo源码分析（二）Dubbo服务发布Export
subtitle:   dubbo
date:       2019-06-05
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - Dubbo
---

Dubbo源码分析（二）Dubbo服务发布Export
===========================

服务发布
====

打印的日志
-----

    [INFO ]  com.alibaba.dubbo.config.AbstractConfig {ServiceBean.java:107} -  [DUBBO] The service ready on spring started. service: com.alibaba.dubbo.demo.DemoService, dubbo version: 2.0.0, current host: 127.0.0.1
    [INFO ]  com.alibaba.dubbo.config.AbstractConfig {ServiceConfig.java:575} -  [DUBBO] Export dubbo service com.alibaba.dubbo.demo.DemoService to local registry, dubbo version: 2.0.0, current host: 127.0.0.1
    [INFO ]  com.alibaba.dubbo.config.AbstractConfig {ServiceConfig.java:535} -  [DUBBO] Export dubbo service com.alibaba.dubbo.demo.DemoService to url dubbo://192.168.31.132:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=948&side=provider&timestamp=1538476342717, dubbo version: 2.0.0, current host: 127.0.0.1
    [INFO ]  com.alibaba.dubbo.config.AbstractConfig {ServiceConfig.java:546} -  [DUBBO] Register dubbo service com.alibaba.dubbo.demo.DemoService url dubbo://192.168.31.132:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=1173&side=provider&timestamp=1538490950321 to registry registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=1173&registry=zookeeper&timestamp=1538490950209, dubbo version: 2.0.0, current host: 127.0.0.1
    [INFO ]  com.alibaba.dubbo.remoting.transport.AbstractServer {AbstractServer.java:64} -  [DUBBO] Start NettyServer bind /0.0.0.0:20880, export /192.168.31.132:20880, dubbo version: 2.0.0, current host: 127.0.0.
    [INFO ]  com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistry {AbstractRegistry.java:223} -  [DUBBO] Load registry store file /Users/isz_pm/.dubbo/dubbo-registry-localhost.cache, data: {
    [INFO ]  org.apache.zookeeper.ZooKeeper {ZooKeeper.java:438} - Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.I0Itec.zkclient.ZkClient@26fc13bc
    
    复制代码

我们通过启动provider，看下控制台的日志输出，基本上可以看出Dubbo的服务发布的几个步骤

Dubbo怎么和Spring融合
----------------

Spring 提供了可扩展的Schema，Dubbo是怎么扩展的呢，先看下Dubbo的配置文件：

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
           xmlns="http://www.springframework.org/schema/beans"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
    	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    
        <!-- 提供方应用信息，用于计算依赖关系 -->
        <dubbo:application name="demo-provider"/>
        <!-- 使用multicast广播注册中心暴露服务地址 -->
        <dubbo:registry address="zookeeper://localhost:2181"/>
        <!-- 用dubbo协议在20880端口暴露服务 -->
        <dubbo:protocol name="dubbo" port="20880"/>
        <!-- 声明需要暴露的服务接口 -->
        <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>
        <!-- 和本地bean一样实现服务 -->
        <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
    </beans>
    复制代码

我看可以看到http://code.alibabatech.com/schema/dubbo/dubbo.xsd，从源码中我们找到：

要想扩展Schema，需要这五个文件，具体的作用就不说了，直接看DubboNamespaceHandler类

    public class DubboNamespaceHandler extends NamespaceHandlerSupport {
        static {
            Version.checkDuplicate(DubboNamespaceHandler.class);
        }
        public void init() {
            registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
            registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
            registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
            registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
            registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
            registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
            registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
            registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
            registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
            registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
        }
    }
    复制代码

这里面初始化了Dubbo配置文件，剩下的就不多说了，直接看ServiceBean中Dubbo是怎么发布服务的

暴露本地服务
------

我们先看ServiceBean，这个是Dubbo服务发布的入口代码

    public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware {
    复制代码

这个类实现了很多接口，其中的ApplicationListener接口,实现了onApplicationEvent()事件方法，

    public void onApplicationEvent(ApplicationEvent event) {
            if (ContextRefreshedEvent.class.getName().equals(event.getClass().getName())) {
                //判断是否需要延迟发布
                if (isDelay() && !isExported() && !isUnexported()) {
                    if (logger.isInfoEnabled()) {
                        logger.info("The service ready on spring started. service: " + getInterface());
                    }
                    //调用发布方法
                    export();
                }
            }
        }
    复制代码

进入export()方法

    public synchronized void export() {
            ···
            if (delay != null && delay > 0) {
                Thread thread = new Thread(new Runnable() {
                    public void run() {
                        try {
                            //1.如果需要延迟发布，直接sleep
                            Thread.sleep(delay);
                        } catch (Throwable e) {
                        }
                        doExport();
                    }
                });
                thread.setDaemon(true);
                thread.setName("DelayExportServiceThread");
                thread.start();
            } else {
                //调用发布方法
                doExport();
            }
        }
    复制代码

进入doExport()方法，继续各种check，进入doExportUrls()方法

     private void doExportUrls() {
            List<URL> registryURLs = loadRegistries(true);
            for (ProtocolConfig protocolConfig : protocols) {
                doExportUrlsFor1Protocol(protocolConfig, registryURLs);
            }
        }
    复制代码

这里registryURL是啥，看debug结果：

loadRegistries()方法中返回了一个URL的List，我们看下这个URL是以registry开头的，localhost:2181是我们配置在xml中<dubbo:registry address="zookeeper://localhost:2181"/>的这段代码中获取的，URL后面都是需要的数据，有没有理解上一篇中说的一句话，**Dubbo是基于URL驱动的**  
那什么会有多个URL呢，因为我们在xml中可以配置多个dubbo:registry，可以把服务发布到多个注册中心  
调用doExportUrlsFor1Protocol()方法，此时protocolConfig是<dubbo:protocol name="dubbo" port="20880"/>

    private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
            ···
            URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
    
            if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .hasExtension(url.getProtocol())) {
                url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                        .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
            }
            
            String scope = url.getParameter(Constants.SCOPE_KEY);
            //配置为none不暴露
            if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
    
                //配置不是remote的情况下做本地暴露 (配置为remote，则表示只暴露远程服务)
                if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
                    exportLocal(url);
                }
                //如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露本地服务)
                if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                    }
                    if (registryURLs != null && registryURLs.size() > 0
                            && url.getParameter("register", true)) {
                        for (URL registryURL : registryURLs) {
                            url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                            URL monitorUrl = loadMonitor(registryURL);
                            if (monitorUrl != null) {
                                url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                            }
                            if (logger.isInfoEnabled()) {
                                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                            }
                            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
    
                            Exporter<?> exporter = protocol.export(invoker);
                            exporters.add(exporter);
                        }
                    } else {
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
    
                        Exporter<?> exporter = protocol.export(invoker);
                        exporters.add(exporter);
                    }
                }
            }
            this.urls.add(url);
        }
    复制代码

先看服务本地发布exportLocal()方法  
此时的url=dubbo://192.168.31.132:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=843&side=provider&timestamp=1538464205631

    private void exportLocal(URL url) {
            if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                URL local = URL.valueOf(url.toFullString())
                        .setProtocol(Constants.LOCAL_PROTOCOL)
                        .setHost(NetUtils.LOCALHOST)
                        .setPort(0);
                Exporter<?> exporter = protocol.export(
                        proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
                exporters.add(exporter);
                logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
            }
        }
    复制代码

继续组装url，local=injvm://127.0.0.1/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=843&side=provider&timestamp=1538464205631  
执行proxyFactory.getInvoker(ref, (Class) interfaceClass, local)，这个时候的proxyFactory是什么？

    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
    复制代码

这个时候proxyFactory是一个动态适配器代理类ProxyFactory$Adpative，我们把这个代理类拿出来

    package com.alibaba.dubbo.rpc;
    import com.alibaba.dubbo.common.extension.ExtensionLoader;
    public class ProxyFactory$Adpative implements com.alibaba.dubbo.rpc.ProxyFactory {
        public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws com.alibaba.dubbo.rpc.RpcException {
            if (arg2 == null) throw new IllegalArgumentException("url == null");
            com.alibaba.dubbo.common.URL url = arg2;
            String extName = url.getParameter("proxy", "javassist");
            if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
            com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
            return extension.getInvoker(arg0, arg1, arg2);
        }
        public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
            if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
            if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
            String extName = url.getParameter("proxy", "javassist");
            if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
            com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
            return extension.getProxy(arg0);
        }
    }
    复制代码

这个时候执行proxyFactory.getInvoker(ref, (Class) interfaceClass, local)，这个时候的local是injvm开头，所以在getInvoke方法中，extName=javassist,同理执行getExtension("javassist")方法，这个时候应该返回什么呢？是JavassistProxyFactory吗？不是，因为

我们看到，这个时候有一个包装类，所以这个时候返回的是StubProxyFactoryWrapper，所以这个时候执行的是StubProxyFactoryWrapper.getInvoker方法

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException {
            return proxyFactory.getInvoker(proxy, type, url);
    }
    复制代码

先理一下这个方法的三个参数，proxy是接口实现类的引用，type是接口，url是injvm开头的  
这个时候的proxyFactory是什么？我们知道Dubbo的IOC帮我们注入了包装类需要的实例参数，所以这个时候proxyFactory是JavassistProxyFactory，

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
            // TODO Wrapper类不能正确处理带$的类名
            final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
            return new AbstractProxyInvoker<T>(proxy, type, url) {
                @Override
                protected Object doInvoke(T proxy, String methodName,
                                          Class<?>[] parameterTypes,
                                          Object[] arguments) throws Throwable {
                    return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
                }
            };
        }
    复制代码

先看下Wrapper wrapper = Wrapper.getWrapper()方法，传入的参数是proxy.getClass()就是我们的实现类DemoServiveImpl的Class，

    public static Wrapper getWrapper(Class<?> c) {
            ···
                ret = makeWrapper(c);
                WRAPPER_MAP.put(c, ret);
            }
            return ret;
        }
    复制代码

这个里面的makeWrapper方法就是创建一个动态代理类，我们看下这个方法的部分内容：

    private static Wrapper makeWrapper(Class<?> c) {
        ···
            StringBuilder c1 = new StringBuilder("public void setPropertyValue(Object o, String n, Object v){ ");
            StringBuilder c2 = new StringBuilder("public Object getPropertyValue(Object o, String n){ ");
            StringBuilder c3 = new StringBuilder("public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws " + InvocationTargetException.class.getName() + "{ ");
        ···
        return (Wrapper) wc.newInstance();
        ···
    复制代码

这里生成了invokeMethod方法，最终把这个对象实例化，我们猜想一下，Dubbo的消费端来调用服务端的接口的时候，是不是通过这个代理对象中的invokerMethod方法最终去执行接口实现类的代码呢？ 我们看到 new AbstractProxyInvoker(proxy, type, url)中重写了一个doInvoke方法，这个doInvoke方法中调用了wrapper.invokeMethod()，这个wrapper就是刚刚生成的wrapper  
接着回到ServiceConfig.exportLocal()方法中  
执行Exporter<?> exporter = protocol.export(invoker);这个时候protocol是什么？贴代码

    private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    复制代码

这个时候protocol=Protocol$Adpative，执行Protocol$Adpative.export方法  
Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("injvm")， 这个时候得到的extension什么？是ProtocolListenrWrapper(ProtocolFiterWrapper(InjvmProtocol)),这样的一个wrapper对象,执行export方法，这里的Fiter会生成8个Fiter链，看下InjvmProtocol中的export方法，目的是exporterMap.put(key, this) 此时，key=com.alibaba.dubbo.demo.DemoService, this=InjvmExporter

    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
           return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
       }
       InjvmExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
           super(invoker);
           this.key = key;
           this.exporterMap = exporterMap;
           exporterMap.put(key, this);
       }
    复制代码

最终得到一个ListenerExportWrapper,放到exporters中,到此，本地服务发布完成

    [INFO ]  com.alibaba.dubbo.config.AbstractConfig {ServiceConfig.java:575} -  [DUBBO] Export dubbo service com.alibaba.dubbo.demo.DemoService to local registry, dubbo version: 2.0.0, current host: 127.0.0.1
    复制代码

暴露远程服务
------

从这段代码开始读

    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
    复制代码

registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString())，这个返回一个URL=registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&export=dubbo%3A%2F%2F192.168.31.132%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D948%26side%3Dprovider%26timestamp%3D1538476342717&pid=948&registry=zookeeper&timestamp=1538476342610  
这个时候proxyFactory同样是Protocol$Adpative，进入getInvoker方法： ProxyFactory extension = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension("javassist");  
这里extension=StubProxyFactoryWrapper包装类，继续执行getInvoker()方法，最终在JavassistProxyFactory的getInvoker中 new AbstractProxyInvoker(proxy, type, url)并返回，这个就是我们需要的invoker  
继续执行Exporter<?> exporter = protocol.export(invoker);  
此时protocol=Protocol$Adaptive ,执行Protocol$Adaptive.export方法

    Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("registry");
    return extension.export(arg0);
    复制代码

同理extension=ProtocolListenrWrapper(ProtocolFiterWrapper(RegistryProtocol)),直接进入RegistryProtocol.export方法中：

    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
           //export invoker  先启动本地监听服务
           final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
           //registry provider
           final Registry registry = getRegistry(originInvoker);
           final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
           registry.register(registedProviderUrl);
           // 订阅override数据
           // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
           final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
           final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
           overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
           registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
           //保证每次export都返回一个新的exporter实例
           return new Exporter<T>() {
               public Invoker<T> getInvoker() {
                   return exporter.getInvoker();
               }
    
               public void unexport() {
                   try {
                       exporter.unexport();
                   } catch (Throwable t) {
                       logger.warn(t.getMessage(), t);
                   }
                   try {
                       registry.unregister(registedProviderUrl);
                   } catch (Throwable t) {
                       logger.warn(t.getMessage(), t);
                   }
                   try {
                       overrideListeners.remove(overrideSubscribeUrl);
                       registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
                   } catch (Throwable t) {
                       logger.warn(t.getMessage(), t);
                   }
               }
           };
       }
    
    复制代码

进入doLocalExport：

    private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
           String key = getCacheKey(originInvoker);
           ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
           if (exporter == null) {
               synchronized (bounds) {
                   exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
                   if (exporter == null) {
                       final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                       exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
                       bounds.put(key, exporter);
                   }
               }
           }
           return (ExporterChangeableWrapper<T>) exporter;
       }
    复制代码

此时，我们看到(Exporter) protocol.export(invokerDelegete)这段代码，这个时候protocol是什么呢，是Protocol$Adaptive,什么时候时候赋值的呢，是加载扩展点的时候，有个injectExtension方法，依赖注入了protocol  
继续执行Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("dubbo"),这个extension是ProtocolListenrWrapper(ProtocolFiterWrapper(DubboProtocol))  
执行DubboProtocol.export方法：

    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            URL url = invoker.getUrl();
            // export service.
            String key = serviceKey(url);
            DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
            exporterMap.put(key, exporter);
            //export an stub service for dispaching event
            Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
            Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
            if (isStubSupportEvent && !isCallbackservice) {
                String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
                if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                    if (logger.isWarnEnabled()) {
                        logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                                "], has set stubproxy support event ,but no stub methods founded."));
                    }
                } else {
                    stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
                }
            }
    
            openServer(url);
    
            return exporter;
        }
    复制代码

serviceKey(url), 这里我们得到一个key=com.alibaba.dubbo.demo.DemoService:20880，目的是要exporterMap.put(key, this)// key=com.alibaba.dubbo.demo.DemoService:20880, this=DubboExporter ，这里其中最重要的一点就是将invoker转化为exporter，专题最后我们会分析一下所有的invoker

终于我们看到了一段代码，openServer(url),千幸万苦，我们要开始启动Netty服务了，先休息一下，写的好累啊

启动Netty服务
---------

    private void openServer(URL url) {
            // find server.
            String key = url.getAddress();
            //client 也可以暴露一个只有server可以调用的服务。
            boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
            if (isServer) {
                ExchangeServer server = serverMap.get(key);
                if (server == null) {
                    serverMap.put(key, createServer(url));
                } else {
                    //server支持reset,配合override功能使用
                    server.reset(url);
                }
            }
        }
    复制代码

首先得到一个key=ip:端口，再调用createServer(),创建服务,开启心跳检测，默认使用 netty。组装 url

    private ExchangeServer createServer(URL url) {
            //默认开启server关闭时发送readonly事件
            url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
            //默认开启heartbeat
            url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
            String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);
    
            if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
                throw new RpcException("Unsupported server type: " + str + ", url: " + url);
    
            url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
            ExchangeServer server;
            try {
                server = Exchangers.bind(url, requestHandler);
            } catch (RemotingException e) {
                throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
            }
            str = url.getParameter(Constants.CLIENT_KEY);
            if (str != null && str.length() > 0) {
                Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
                if (!supportedTypes.contains(str)) {
                    throw new RpcException("Unsupported client type: " + str);
                }
            }
            return server;
        }
    复制代码

进入 Exchangers.bind(url, requestHandler)方法

    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
           ···
            url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
            return getExchanger(url).bind(url, handler);
        }
    复制代码

进入getExchanger(url)方法：

    public static Exchanger getExchanger(URL url) {
            String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
            return getExchanger(type);
    }
    public static Exchanger getExchanger(String type) {
    //此时type=header
        return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
    }
    复制代码

这里返回HeaderExchanger，进入HeaderExchanger.bind方法

    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
            return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
    复制代码

这里初始化一个new DecodeHandler(new HeaderExchangeHandler(handler))，进入Transporters.bind()方法,这里进入transporter层

    public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
            ···
            ChannelHandler handler;
            if (handlers.length == 1) {
                handler = handlers[0];
            } else {
                handler = new ChannelHandlerDispatcher(handlers);
            }
            return getTransporter().bind(url, handler);
        }
    复制代码

进入getTransporter()方法

    public static Transporter getTransporter() {
            return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
    复制代码

老规矩，先看下Transporter接口

    @SPI("netty")
    public interface Transporter {
        @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
        Server bind(URL url, ChannelHandler handler) throws RemotingException;
        
        @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
        Client connect(URL url, ChannelHandler handler) throws RemotingException;
    
    }
    复制代码

@SPI注解，默认netty，两个方法，一个bind方法，一个connect方法

获取自适应适配器扩展点Transporter![Adpative，进入Transporter](https://juejin.im/equation?tex=Adpative%EF%BC%8C%E8%BF%9B%E5%85%A5Transporter)Adpative.bind()方法，执行

    ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension("netty");
    复制代码

执行NettyTransport.bind方法

    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
            return new NettyServer(url, listener);
    }
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
            super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }
    public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
           ···
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
    复制代码

AbstractServer的构造方法中，进入doOpen();这个是一个抽象方法，具体实现再NettyTransport中，这里可以看到具体的实现都是在各自的扩展类中去实现，这些不同的扩展类会有一些公共的方法，就可以提取出来一个抽象类去实现

    protected void doOpen() throws Throwable {
            NettyHelper.setNettyLoggerFactory();
            ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
            ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
            ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
            bootstrap = new ServerBootstrap(channelFactory);
    
            final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
            channels = nettyHandler.getChannels();
            bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
                public ChannelPipeline getPipeline() {
                    NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                    ChannelPipeline pipeline = Channels.pipeline();
                    pipeline.addLast("decoder", adapter.getDecoder());
                    pipeline.addLast("encoder", adapter.getEncoder());
                    pipeline.addLast("handler", nettyHandler);
                    return pipeline;
                }
            });
            // bind
            channel = bootstrap.bind(getBindAddress());
        }
    复制代码

在这里，我们看到了熟悉的netty代码，设置 NioServerSocketChannelFactory boss worker的线程池 线程个数为3，再设置设置编解码和hander处理类， 回到new HeaderExchangeServer()方法中

    public HeaderExchangeServer(Server server) {
            if (server == null) {
                throw new IllegalArgumentException("server == null");
            }
            this.server = server;
            this.heartbeat = server.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
            this.heartbeatTimeout = server.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
            if (heartbeatTimeout < heartbeat * 2) {
                throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
            }
            //这是一个心跳定时器，采用了线程池，如果断开就心跳重连。
            startHeatbeatTimer();
    }
    复制代码

    private void startHeatbeatTimer() {
           stopHeartbeatTimer();
           if (heartbeat > 0) {
           //每隔 heartbeat 时间执行一次,此时默认为60000ms
               heatbeatTimer = scheduled.scheduleWithFixedDelay(
                       new HeartBeatTask(new HeartBeatTask.ChannelProvider() {
                           public Collection<Channel> getChannels() {
                           //获取channels
                               return Collections.unmodifiableCollection(
                                       HeaderExchangeServer.this.getChannels());
                           }
                       }, heartbeat, heartbeatTimeout),
                       heartbeat, heartbeat, TimeUnit.MILLISECONDS);
           }
       }
    复制代码

到此，Netty服务已经启动

连接注册中心
------

我们回到Registry.export()方法中，

    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
            //export invoker
            final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
            //registry provider  现在我们要从这里开始连接注册中心
            final Registry registry = getRegistry(originInvoker);
            final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
            registry.register(registedProviderUrl);
            // 订阅override数据
            // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
            final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
            final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
            overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
            registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
            //保证每次export都返回一个新的exporter实例
            return new Exporter<T>() {
                public Invoker<T> getInvoker() {
                    return exporter.getInvoker();
                }
    
                public void unexport() {
                    try {
                        exporter.unexport();
                    } catch (Throwable t) {
                        logger.warn(t.getMessage(), t);
                    }
                    try {
                        registry.unregister(registedProviderUrl);
                    } catch (Throwable t) {
                        logger.warn(t.getMessage(), t);
                    }
                    try {
                        overrideListeners.remove(overrideSubscribeUrl);
                        registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
                    } catch (Throwable t) {
                        logger.warn(t.getMessage(), t);
                    }
                }
            };
        }
    复制代码

进入getRegistry()方法,根据invoker的地址获取registry实例

    private Registry getRegistry(final Invoker<?> originInvoker) {
            URL registryUrl = originInvoker.getUrl();
            if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
                String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
                registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
            }
            return registryFactory.getRegistry(registryUrl);
    }
    复制代码

此时，registryUrl=zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&export=dubbo%3A%2F%2F192.168.31.132%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D1122%26side%3Dprovider%26timestamp%3D1538487268569&pid=1122&timestamp=1538487268461  
此时registryFactory=RegistryFactory$Adaptive ,执行getRegistry方法  
ExtensionLoader.getExtensionLoader(RegistryFactory.class).getExtension("zookeeper");这里得到ZookeeperRegistryFactory，执行ZookeeperRegistryFactory.getRegistry()

    public Registry getRegistry(URL url) {
            url = url.setPath(RegistryService.class.getName())
                    .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                    .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
            String key = url.toServiceString();
            // 锁定注册中心获取过程，保证注册中心单一实例
            LOCK.lock();
            try {
                Registry registry = REGISTRIES.get(key);
                if (registry != null) {
                    return registry;
                }
                registry = createRegistry(url);
                if (registry == null) {
                    throw new IllegalStateException("Can not create registry " + url);
                }
                REGISTRIES.put(key, registry);
                return registry;
            } finally {
                // 释放锁
                LOCK.unlock();
            }
        }
    
    复制代码

进入createRegistry()方法：

    public Registry createRegistry(URL url) {
           return new ZookeeperRegistry(url, zookeeperTransporter);
    }
    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
           super(url);
           if (url.isAnyHost()) {
               throw new IllegalStateException("registry address == null");
           }
           String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
           if (!group.startsWith(Constants.PATH_SEPARATOR)) {
               group = Constants.PATH_SEPARATOR + group;
           }
           //设置根节点
           this.root = group;
           zkClient = zookeeperTransporter.connect(url);
           zkClient.addStateListener(new StateListener() {
               public void stateChanged(int state) {
                   if (state == RECONNECTED) {
                       try {
                           recover();
                       } catch (Exception e) {
                           logger.error(e.getMessage(), e);
                       }
                   }
               }
           });
       }
    复制代码

首先AbstractRegistry抽象类中，调用loadProperties()目的把注册的服务缓存到本地

然后创建连接，zkClient = zookeeperTransporter.connect(url);进入ZkclientZookeeperTransporter.connect方法

    public ZookeeperClient connect(URL url) {
           return new ZkclientZookeeperClient(url);
    }
    public ZkclientZookeeperClient(URL url) {
           super(url);
           //创建zk连接
           client = new ZkClient(url.getBackupAddress());
           //订阅的目标：连接断开，重连
           client.subscribeStateChanges(new IZkStateListener() {
               public void handleStateChanged(KeeperState state) throws Exception {
                   ZkclientZookeeperClient.this.state = state;
                   if (state == KeeperState.Disconnected) {
                       stateChanged(StateListener.DISCONNECTED);
                   } else if (state == KeeperState.SyncConnected) {
                       stateChanged(StateListener.CONNECTED);
                   }
               }
    
               public void handleNewSession() throws Exception {
                   stateChanged(StateListener.RECONNECTED);
               }
           });
    }
    复制代码

创建zk连接后，返回到ZookeeperRegistry的构造方法中，

    zkClient.addStateListener(new StateListener() {
               public void stateChanged(int state) {
                   if (state == RECONNECTED) {
                       try {
                           recover();
                       } catch (Exception e) {
                           logger.error(e.getMessage(), e);
                       }
                   }
               }
           });
    复制代码

recover方法作用是连接失败 重连  
回到ResigstryProtocol.export()方法，执行registry.register(registedProviderUrl); 调用 FailbackRegistry 类中的 register. 为什么呢?因为 ZookeeperRegistry 这个类中并没有 register 这个方法，但是他的父类 FailbackRegistry 中存在 register 方法，而这个类又重写了 AbstractRegistry 类中的 register 方法。所以我们可以直接定位大 FailbackRegistry 这个类 中的 register 方法中

    public FailbackRegistry(URL url) {
           super(url);
           int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
           this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
               public void run() {
                   // 检测并连接注册中心
                   try {
                   //失败重连
                       retry();
                   } catch (Throwable t) { // 防御性容错
                       logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
                   }
               }
           }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
       }
    复制代码

FailbackRegistry，从名字上来看，是一个失败重试机制  
调用父类的register方法，讲当前url添加到缓存集合中  
调用 doRegister 方法，这个方法很明显，是一个抽象方法，会由ZookeeperRegistry 子类实现

    protected void doRegister(URL url) {
            try {
                zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
            } catch (Throwable e) {
                throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
            }
    }
    复制代码

在这里想zk注册服务，最终实现注册的是AbstractZookeeperClient.create方法

    public void create(String path, boolean ephemeral) {
            int i = path.lastIndexOf('/');
            if (i > 0) {
                create(path.substring(0, i), false);
            }
            if (ephemeral) {
                createEphemeral(path);
            } else {
                createPersistent(path);
            }
        }
    复制代码

后续的注册监听的代码就不分析了，服务端去zk注册一个监听，当zk节点发生变化时，通知服务端处理  
至此，Dubbo服务的发布源码分析完成
