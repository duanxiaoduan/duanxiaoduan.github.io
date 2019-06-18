---
layout:     post
title:      Dubbo源码分析（一）Dubbo的扩展点机制
subtitle:   dubbo
date:       2019-06-04
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - dubbo
---

Dubbo源码分析（一）Dubbo的扩展点机制
=======================

写在前面的话
======

自己用Dubbo也有几年时间，一直没有读过Dubbo的源码，现在来读一读Dubbo的源码，分析一下Dubbo的几个核心，并写一个Dubbo的源码专题来记录一下学习过程，供大家参考，写的不好的地方，欢迎拍砖  


**PS：读源码前先掌握以下基础**

1.  JDK的SPI
2.  Java多线程/线程池基础
3.  Javasissit基础（动态编译）
4.  Netty基础
5.  Zookeeper基础，zkClient客户端API
6.  工厂模式，装饰模式，模板模式，单例模式，动态代理模式
7.  Spring的schema自定义扩展
8.  序列化

**PS：读源码前的建议**

1.  代码量很大，各个地方都有关联，慢慢读，不要着急，一遍不行就两遍，两遍不行就三遍，总有看懂的时候
2.  带着问题去看，先想想这段代码的目的是什么，解决了什么问题
3.  沿着一条主线读，不影响流程走向的代码可以略过

Dubbo的扩展点
=========

为什么先读扩展点
--------

之所以选择先从Dubbo的扩展点机制入手，因为Dubbo的整体架构设计上，都是通过扩展点去实现，先了解清楚这块内容，才能读懂代码。

Dubbo扩展点规范
----------

*   如果要扩展自定义的SPI，可以在resources目录下配置三种目录，分别是：META-INF/dubbo/ 或者 META-INF/services/ 或者 META-INF/dubbo/internal/
*   文件名称和接口名称保持一致，文件内容为key=vaule的形式xxx=com.alibaba.xxx.XxxProtocol  
    
*   举个栗子：如果我们要扩展Dubbo的一个协议，在META-INF/dubbo/com.alibaba.dubbo.rpc.Protocol这个文件里面增加一行自己的扩展：xxx=com.alibaba.xxx.XxxProtocol,在Dubbo的配置文件<dubbo:protocol name="xxx" />，这样就可以实现自定义的Dubbo协议  
    

Dubbo的扩展点和JDK的SPI的区别
--------------------

Dubbo的扩展点（Extension）在JDK的SPI思想的基础上做了一些改进：

*   Dubbo的设计中用到了大量的全局缓存，所有的Extension都缓存在cachedInstances中，该对象类型为ConcurrentMap<String, Holder>
*   Dubbo的扩展点支持默认实现，比如，Protocol中的@SPI("dubbo")默认为DubboProtocol，默认扩展点可以通过ExtensionLoader.getExtensionLoader(Protocol.class).getDefaultExtension()获取
*   Dubbo的扩展点可以动态获取扩展对象，比如：ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName)来获取扩展对象，我觉得这是Dubbo扩展点设计的很有意思的地方，非常的灵活方便，代码中大量的用到了这个方法
*   Dubbo的扩展点提供了AOP功能，在cachedWrapperClasses中，在原来的SPI的类上包装了XxxxFilterWrapper XxxxListenerWrapper
*   Dubbo扩展点提供了IOC功能，通过构造函数注入所需的实例，后面源码会分析

读源码
---

*   先想想Dubbo的SPI的目的是什么？获取一个我们所需要的指定的对象
*   怎么获取呢？ExtensionLoader.getExtension(String name)  
    我们先从这段代码入手，这个代码的作用就是获取一个自适应的扩展类，我们看下这段代码的整个执行流程：

    ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    复制代码

getExtensionLoader方法中传入的了一个Protocol，我们看下Protocol长啥样

    @SPI("dubbo")
    public interface Protocol {
       //获取缺省端口，当用户没有配置端口时使用。
       int getDefaultPort();
    
      // 暴露远程服务：<br>
       @Adaptive
       <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    
       //引用远程服务：<br>
       @Adaptive
       <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
    
       //释放协议：<br>
       void destroy();
    复制代码

这是一个协议接口，在类上有@SPI注解，有个默认值dubbo，在方法上有@Adaptive注解，这两个注解是什么作用呢？我们继续往下看getExtensionLoader方法：

    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    ·····
           //先从缓存中取值，为null，则去new一个ExtensionLoader
           ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
           if (loader == null) {
               EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
               loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
           }
           return loader;
       }
    复制代码

继续进入new ExtensionLoader(type)方法：

    private ExtensionLoader(Class<?> type) {
           this.type = type;
           objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
       }
    复制代码

给type变量赋值，objectFactory赋值，此时传入的type是Protcol，继续执行ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()方法，这里我们分为两个步骤：  
步骤一：ExtensionLoader.getExtensionLoader(ExtensionFactory.class)，此时type为ExtensionFactory.class,这段代码我们得到一个ExtensionLoader实例,这个ExtensionLoader实例中，objetFactory为null， 步骤二：getAdaptiveExtension()方法，为cachedAdaptiveInstance赋值，我们来看这个方法：

      public T getAdaptiveExtension() {
           Object instance = cachedAdaptiveInstance.get();
           if (instance == null) {
               if (createAdaptiveInstanceError == null) {
                   synchronized (cachedAdaptiveInstance) {
                       instance = cachedAdaptiveInstance.get();
                       if (instance == null) {
                           try {
                               instance = createAdaptiveExtension();
                               cachedAdaptiveInstance.set(instance);
                           } catch (Throwable t) {
                               createAdaptiveInstanceError = t;
                               throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                           }
                       }
                   }
               } else {
                   throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
               }
           }
    复制代码

双重检查锁判断缓存，如果没有则进入createAdaptiveExtension()方法，这个方法有两部分，一个是injectExtension，一个是getAdaptiveExtensionClass().newInstance()

    private T createAdaptiveExtension() {                                                                                            
     try {                                                                                                                        
         return injectExtension((T) getAdaptiveExtensionClass().newInstance());                                                   
     } catch (Exception e) {                                                                                                      
         throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);         
     }                                                                                                                            
    }                                                                                                                                
    复制代码

先看getAdaptiveExtensionClass()，获取一个适配器扩展点的类

    private Class<?> getAdaptiveExtensionClass() {                                                      
     getExtensionClasses();                                                                          
     if (cachedAdaptiveClass != null) {                                                              
         return cachedAdaptiveClass;                                                                 
     }                                                                                               
     return cachedAdaptiveClass = createAdaptiveExtensionClass();                                    
    }                                                                                                   
    复制代码

这里我们进入getExtensionClasses()方法，双重检查锁判断，如果没有，继续

    private Map<String, Class<?>> getExtensionClasses() {
         Map<String, Class<?>> classes = cachedClasses.get();
         if (classes == null) {
             synchronized (cachedClasses) {
                 classes = cachedClasses.get();
                 if (classes == null) {
                     classes = loadExtensionClasses();
                     cachedClasses.set(classes);
                 }
             }
         }
         return classes;
     }
    复制代码

进入loadExtensionClasses()方法

        // 此方法已经getExtensionClasses方法同步过。
        private Map<String, Class<?>> loadExtensionClasses() {
            final SPI defaultAnnotation = type.getAnnotation(SPI.class);
            if (defaultAnnotation != null) {
                String value = defaultAnnotation.value();
                if (value != null && (value = value.trim()).length() > 0) {
                    String[] names = NAME_SEPARATOR.split(value);
                    if (names.length > 1) {
                        throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                                + ": " + Arrays.toString(names));
                    }
                    if (names.length == 1) cachedDefaultName = names[0];
                }
            }
    
            Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
            loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
            loadFile(extensionClasses, DUBBO_DIRECTORY);
            loadFile(extensionClasses, SERVICES_DIRECTORY);
            return extensionClasses;
        }
    复制代码

这里有个type.getAnnotation(SPI.class)，这个type就是刚刚再初始化ExtensionLoader的时候传入的，我们先看type=ExtensionFactory.class的情况，ExtensionFactory接口类上有@SPI注解，但是value为空，然后三次调用loadFile方法，分别对应Dubbo扩展点的三个配置文件路径，在源码中我们可以找到ExtensionFactory对应的文件，

通过loadFile方法，最终extensionClasses返回SpringExtensionFactory和SpiExtensionFactory 缓存到cachedClasses中，为什么只返回了2个类呢，AdaptiveExtensionFactory为什么没有返回呢，因为在loadFile中AdaptiveExtensionFactory因为类上有@Adaptive注解，所以直接缓存到cachedAdaptiveClass中(此时，我们要思考，@Adaptive注解放在类上和放在方法上有什么区别)，我们看下loadFile中的关键代码

    private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
    ····
           // 1.判断当前class类上面有没有Adaptive注解，如果有，则直接赋值给cachedAdaptiveClass
           if (clazz.isAnnotationPresent(Adaptive.class)) {
               if (cachedAdaptiveClass == null) {
                   cachedAdaptiveClass = clazz;
               } else if (!cachedAdaptiveClass.equals(clazz)) {
                   throw new IllegalStateException("More than 1 adaptive class found: "
                           + cachedAdaptiveClass.getClass().getName()
                           + ", " + clazz.getClass().getName());
               }
           } else {
               //2.如果没有类注解，那么判断该class中没有参数是type的构造方法，如果有，则把calss放入cachedWrapperClasses中
               try {
                   clazz.getConstructor(type);
                   Set<Class<?>> wrappers = cachedWrapperClasses;
                   if (wrappers == null) {
                       cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                       wrappers = cachedWrapperClasses;
                   }
                   wrappers.add(clazz);
               } catch (NoSuchMethodException e) {
                   //3.判断是否有默认构造方法
                   clazz.getConstructor();
                   if (name == null || name.length() == 0) {
                       name = findAnnotationName(clazz);
                       if (name == null || name.length() == 0) {
                           if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                   && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                               name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                           } else {
                               throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
                           }
                       }
                   }
                   String[] names = NAME_SEPARATOR.split(name);
                   if (names != null && names.length > 0) {
                       //4.判断class是否有@Activate注解，如果有，则放入cachedActivates
                       Activate activate = clazz.getAnnotation(Activate.class);
                       if (activate != null) {
                           cachedActivates.put(names[0], activate);
                       }
                       for (String n : names) {
                           //5.缓存calss到cachedNames中
                           if (!cachedNames.containsKey(clazz)) {
                               cachedNames.put(clazz, n);
                           }
                           Class<?> c = extensionClasses.get(n);
                           if (c == null) {
                               extensionClasses.put(n, clazz);
                           } else if (c != clazz) {
                               throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                           }
                       }
                   }
               }
           }
       }
       }
    复制代码

至此，我们已经拿到了extensionClasses，并缓存到了cachedClasses中，回到getAdaptiveExtensionClass()方法中

    private Class<?> getAdaptiveExtensionClass() {
           getExtensionClasses();
           if (cachedAdaptiveClass != null) {
               return cachedAdaptiveClass;
           }
           return cachedAdaptiveClass = createAdaptiveExtensionClass();
       }
    复制代码

如果cachedAdaptiveClass不为空，那么就返回cachedAdaptiveClass，刚刚我们在loadFile()方法中讲过，@Adaptive注解在类上，那么就会缓存到cachedAdaptiveClass中，这个时候cachedAdaptiveClass有值，为AdaptiveExtensionFactory，所以这里直接返回AdaptiveExtensionFactory，继续返回createAdaptiveExtension()方法，刚刚我们只是走完了createAdaptiveExtension()方法中的一个部分，还有injectExtension方法，这个方法是干什么的，在type=ExtensionFactory.class流程中，这个方法的作用没有体现，先不看injectExtension，我们放在后面的流程去看，然后继续返回到getAdaptiveExtension方法中，把实例AdaptiveExtensionFactory缓存到cachedAdaptiveInstance中，继续返回到ExtensionLoader方法中

    private ExtensionLoader(Class<?> type) {
           this.type = type;
           objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
       }
    复制代码

这个时候，objectFactory已经有值了，就是AdaptiveExtensionFactory，继续返回getExtensionLoader方法

    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
           ····
           //EXTENSION_LOADERS判断是否有type，ConcurrentMap<Class<?>, ExtensionLoader<?>>
           ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
           if (loader == null) {
               EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
               loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
           }
           return loader;
       }
    复制代码

我们把返回的ExtensionLoader实例缓存到EXTENSION_LOADERS中，此时type=Protocol

    ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    复制代码

至此，我们已经执行完了ExtensionLoader.getExtensionLoader(Protocol.class)，得到了ExtensionLoader实例，继续执行getAdaptiveExtension()方法,这个方法在上面已经分析过了，我们再看下跟type=ExtensionFactory的时候有什么区别，先看下com.alibaba.dubbo.rpc.Protocol文件中有哪些扩展点(这个文件在源码中是分散的，可以在Dubbo的jar包中找，jar包中是合并的)

一共有13个扩展点，其中有2个Wrapper包装类，我们直接看loadFile方法，extensionClasses返回了11条记录

这个时候再看下当前内存中的数据

cachedNames中有11条，cachedWrapperClasses中有2条分别是ProtocolListenerWrapper 和 ProtocolFilterWrapper ，cachedClasses中有11条，回到getAdaptiveExtensionClass()方法中，我们在上面说过，Protocol的类上有 @SPI("dubbo")注解，export和refer上有@Adaptive注解，所以此时cachedAdaptiveClass是null， ，进入createAdaptiveExtensionClass()方法，这个方法的目的是自动生成和编译一个动态代理适配器类，名字叫Protocol$Adaptive， 这里又用到了一个Compile扩展点，可以看到，这里用到了ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension()，有木有很熟悉，这里得到一个AdaptiveCompiler（因为AdaptiveCompiler类上有@Adaptive注解）

    private Class<?> createAdaptiveExtensionClass() {
       //生成字节码文件
       String code = createAdaptiveExtensionClassCode(); 
       //获得类加载器
       ClassLoader classLoader = findClassLoader(); 
       com.alibaba.dubbo.common.compiler.Compiler compiler = 
       ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
       //动态编译
       return compiler.compile(code, classLoader);                         
    }                                                                                                                                           
    复制代码

执行compiler.compile(code, classLoader)，先看下AdaptiveCompiler类

    @Adaptive
    public class AdaptiveCompiler implements Compiler {
    
       private static volatile String DEFAULT_COMPILER;
    
       public static void setDefaultCompiler(String compiler) {
           DEFAULT_COMPILER = compiler;
       }
    
       public Class<?> compile(String code, ClassLoader classLoader) {
           Compiler compiler;
           ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
           String name = DEFAULT_COMPILER; // copy reference
           if (name != null && name.length() > 0) {
               compiler = loader.getExtension(name);
           } else {
               compiler = loader.getDefaultExtension();
           }
           return compiler.compile(code, classLoader);
       }
    }
    复制代码

这里的DEFAULT_COMPILER值为JavassistCompiler，执行loader.getExtension(name)，这个方法这里暂时不展开，结果是得到JavassistCompiler实例，这里是一个装饰模式的设计，最终调用JavassistCompiler.compile()方法得到Protocol$Adpative，

回到我们最初的代码的入口

    ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    复制代码

这句代码就最终的返回结果就是Protocol$Adpative，我们把这个代理类拿出来看一下

    package com.alibaba.dubbo.rpc;
    import com.alibaba.dubbo.common.extension.ExtensionLoader;
    public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
      public void destroy() {throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
      }
      public int getDefaultPort() {throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
      }
      public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
          if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
          if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
          String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
          if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
          com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
          return extension.export(arg0);
      }
      public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
          if (arg1 == null) throw new IllegalArgumentException("url == null");
          com.alibaba.dubbo.common.URL url = arg1;
          String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
          if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
          com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
          return extension.refer(arg0, arg1);
      }
    }
    
    复制代码

这个时候如果执行Protocol$Adpative.export方法，我们看下这个适配器代理类里面的export()方法，通过url来获取extName，所以Dubbo是基于URL来驱动的， 看到Protocol extension = (Protocol)ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName)这个方法，这个方法是不是又又又很熟悉，接下来我们来分析getExtension(String name)方法，假设此时extName=dubbo

    public T getExtension(String name) {
            if (name == null || name.length() == 0)
                throw new IllegalArgumentException("Extension name == null");
            if ("true".equals(name)) {
                return getDefaultExtension();
            }
            Holder<Object> holder = cachedInstances.get(name);
            if (holder == null) {
                cachedInstances.putIfAbsent(name, new Holder<Object>());
                holder = cachedInstances.get(name);
            }
            Object instance = holder.get();
            if (instance == null) {
                synchronized (holder) {
                    instance = holder.get();
                    if (instance == null) {
                        instance = createExtension(name);
                        holder.set(instance);
                    }
                }
            }
            return (T) instance;
        }
    复制代码

进入createExtension()方法

    private T createExtension(String name) {
            //1.通过name获取ExtensionClasses，此时为DubboProtocol
            Class<?> clazz = getExtensionClasses().get(name);
            if (clazz == null) {
                throw findException(name);
            }
            try {
                //2.获取DubboProtocol实例
                T instance = (T) EXTENSION_INSTANCES.get(clazz);
                if (instance == null) {
                    EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                    instance = (T) EXTENSION_INSTANCES.get(clazz);
                }
                //3.dubbo的IOC反转控制，就是从spi和spring里面提取对象赋值。
                injectExtension(instance);
                Set<Class<?>> wrapperClasses = cachedWrapperClasses;
                if (wrapperClasses != null && wrapperClasses.size() > 0) {
                    for (Class<?> wrapperClass : wrapperClasses) {
                        //4.如果是包装类
                        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                    }
                }
                return instance;
            } catch (Throwable t) {
                throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                        type + ")  could not be instantiated: " + t.getMessage(), t);
            }
        }
    
    复制代码

第三步injectExtension(instance)，看一下代码：

    private T injectExtension(T instance) {
            try {
                if (objectFactory != null) {
                    //1.拿到所有的方法
                    for (Method method : instance.getClass().getMethods()) {
                        //判断是否是set方法
                        if (method.getName().startsWith("set")
                                && method.getParameterTypes().length == 1
                                && Modifier.isPublic(method.getModifiers())) {
                            Class<?> pt = method.getParameterTypes()[0];
                            try {
                                String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                                //从objectFactory中获取所需要注入的实例
                                Object object = objectFactory.getExtension(pt, property);
                                if (object != null) {
                                    method.invoke(instance, object);
                                }
                            } catch (Exception e) {
                                logger.error("fail to inject via method " + method.getName()
                                        + " of interface " + type.getName() + ": " + e.getMessage(), e);
                            }
                        }
                    }
                }
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
            return instance;
        }
    复制代码

这个方法就是Dubbo完成依赖注入的地方，到这里关于Dubbo的扩展点机制的代码就分析完成了。

总结
==

*   为什么要设计Adaptive？  
    Adaptive设计的目的是为了识别固定已知类和扩展未知类
*   注解在类上和注解在方法上的区别？  
    1.注解在类上：代表人工实现，实现一个装饰类（设计模式中的装饰模式），它主要作用于固定已知类， 目前整个系统只有2个，AdaptiveCompiler、AdaptiveExtensionFactory。
    *   为什么AdaptiveCompiler这个类是固定已知的？因为整个框架仅支持Javassist和JdkCompiler。
    *   为什么AdaptiveExtensionFactory这个类是固定已知的？因为整个框架仅支持2个objFactory,一个是spi,另一个是spring  
        2.注解在方法上：代表自动生成和编译一个动态的Adpative类，它主要是用于SPI，因为spi的类是不固定、未知的扩展类，所以设计了动态XXX$Adaptive类. 例如 Protocol的spi类有 injvm dubbo registry filter listener等等 很多扩展未知类， 它设计了Protocol$Adaptive的类，通过ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(spi类);来提取对象