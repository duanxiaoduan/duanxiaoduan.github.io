---
layout:     post
title:      路由代码分析4（script路由器file路由器）
subtitle:   dubbo
date:       2019-06-23
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - dubbo
---

 dubbo路由代码分析4（script路由器file路由器）
 
 
 这篇分析下，script类型和file类型路由器。  
 目前，script类型和file路由规则，还不能通过dubbo的admin管理页面添加。可以通过java api添加。具体[看这里](https://my.oschina.net/u/146130/blog/1585537)  
 先说，script路由器，它由ScriptRouterFactory路由工厂创建如下：
 
     public class ScriptRouterFactory implements RouterFactory {
     
         public static final String NAME = "script";
     
         public Router getRouter(URL url) {
             return new ScriptRouter(url);
         }
     
     }
 
 dubbo的脚本路由器，是通过执行一段脚本逻辑来执行路由规则，  
 它能定制出比condition路由规则更加灵活的路由规则。  
 先看下它接受的路由规则形式，如下：
 
     URL SCRIPT_URL = URL.valueOf("script://javascript?type=javascript&rule=function route(op1,op2){return op1} route(invokers)");
     
 
 这个url中，type=javascript，表示脚本的语言使用javascript。  
 rule=function route(op1,op2){return op1} route(invokers)，表示具体的脚本内容。route(invokers)表示立即执行route函数。
 
 dubbo脚本路由实现，依赖jdk对脚本引擎的实现。题外话，  
 从jdk1.6,根据JSR223，引入脚本引擎，目前jdk 用java只实现了一个叫Rhino的javasrcipt脚本引擎。  
 实际根据JSR 223标准，任何实现了jdk里，AbstractScriptEngine抽象类等配套接口的脚本引擎，都可以集成到java程序中来,被jvm加载执行。  
 Rhino脚本引擎真的比较神奇，看duboo官方给的一个路由函数：
 
     function route(invokers) {
     	var result = new java.util.ArrayList(invokers.size());
     	for (i = 0; i < invokers.size(); i ++) {
     	if ("10.20.153.10".equals(invokers.get(i).getUrl().getHost())) {
     	result.add(invokers.get(i));
            }
     }
     return result;
     } (invokers); // 表示立即执行方法
 
 里面还能有java语法对象。
 
 下面看下脚本路由器具体实现代码：
 
     public class ScriptRouter implements Router {
     
         private static final Logger logger = LoggerFactory.getLogger(ScriptRouter.class);
     
         private static final Map<String, ScriptEngine> engines = new ConcurrentHashMap<String, ScriptEngine>();
     
         private final ScriptEngine engine;
     
         private final int priority;
     
         private final String rule;
     
         private final URL url;
     
         public ScriptRouter(URL url) {
             this.url = url;
             //通过type key，获取脚本的语言类型，是用来初始化脚本引擎的。
             String type = url.getParameter(Constants.TYPE_KEY);
             //获取优先级，路由之间排序用
             this.priority = url.getParameter(Constants.PRIORITY_KEY, 0);
             //通过 rule key,获取具体的脚本函数字符串
             String rule = url.getParameterAndDecoded(Constants.RULE_KEY);
             //type 没取到值，默认是javascript类型
             if (type == null || type.length() == 0) {
                 type = Constants.DEFAULT_SCRIPT_TYPE_KEY;
             }
             //没有具体的规则，则抛出异常
             if (rule == null || rule.length() == 0) {
                 throw new IllegalStateException(new IllegalStateException("route rule can not be empty. rule:" + rule));
             }
             //根据type，获取java 的脚本类型。这里用了map做缓存。
             ScriptEngine engine = engines.get(type);
             if (engine == null) {
                 //根据type获取java 的脚本类型，这块得熟悉下，java对脚本的支持,
                 engine = new ScriptEngineManager().getEngineByName(type);
     
                 if (engine == null) {
                     throw new IllegalStateException(new IllegalStateException("Unsupported route rule type: " + type + ", rule: " + rule));
                 }
                 engines.put(type, engine);
             }
             this.engine = engine;
             this.rule = rule;
         }
     
         public URL getUrl() {
             return url;
         }
         /***
         *
         *执行路由规则的逻辑
         */
         @SuppressWarnings("unchecked")
         public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
             try {
                 //copy一份，原始invokers
                 List<Invoker<T>> invokersCopy = new ArrayList<Invoker<T>>(invokers);
                 Compilable compilable = (Compilable) engine;
                 Bindings bindings = engine.createBindings();
                 //绑定3个参数，也是在rule规则字串最后，调用函数时，传递的参数名称。
                 bindings.put("invokers", invokersCopy);
                 bindings.put("invocation", invocation);
                 bindings.put("context", RpcContext.getContext());
                 //根据rule 规则字串，编译脚本
                 CompiledScript function = compilable.compile(rule);
                 //执行脚本
                 Object obj = function.eval(bindings);
                 //把结果转型，返回
                 if (obj instanceof Invoker[]) {
                     invokersCopy = Arrays.asList((Invoker<T>[]) obj);
                 } else if (obj instanceof Object[]) {
                     invokersCopy = new ArrayList<Invoker<T>>();
                     for (Object inv : (Object[]) obj) {
                         invokersCopy.add((Invoker<T>) inv);
                     }
                 } else {
                     invokersCopy = (List<Invoker<T>>) obj;
                 }
                 return invokersCopy;
             } catch (ScriptException e) {
                 //fail then ignore rule .invokers.
                 logger.error("route error , rule has been ignored. rule: " + rule + ", method:" + invocation.getMethodName() + ", url: " + RpcContext.getContext().getUrl(), e);
                 return invokers;
             }
         }
     
         /***
          * 路由排序，实现Comparable接口方法
          * @param o
          * @return
          */
         public int compareTo(Router o) {
             if (o == null || o.getClass() != ScriptRouter.class) {
                 return 1;
             }
             ScriptRouter c = (ScriptRouter) o;
             return this.priority == c.priority ? rule.compareTo(c.rule) : (this.priority > c.priority ? 1 : -1);
         }
     
     }
 
 接下来看下，file类型路由器。  
 file路由器，使dubbo可以读取使用放在文件里的路由脚本逻辑。  
 这样用户可以把路由脚本放在文件中，由于路由逻辑在consumer方执行，所以文件要放在consumer能读取的路径里。  
 看看它的代码实现原理。  
 file路由器由FileRouterFactory路由工厂构造。  
 先看下file路由规则形式。如下  
 URL FILE_URL = URL.valueOf("file:///d:/path/to/route.js?router=script");  
 可以看到构造的url数据结构内容如下图：
 
 ![](https://static.oschina.net/uploads/space/2017/1228/132502_bCzU_146130.png)
 
 源码解析：
 
     public class FileRouterFactory implements RouterFactory {
     
         public static final String NAME = "file";
     
         private RouterFactory routerFactory;
     
         public void setRouterFactory(RouterFactory routerFactory) {
             this.routerFactory = routerFactory;
         }
     
         public Router getRouter(URL url) {
             try {
                 // File URL 转换成 其它Route URL，然后Load
                 // file:///d:/path/to/route.js?router=script ==> script:///d:/path/to/route.js?type=js&rule=<file-content>
                 //通过router 获取路由类型， 默认是script类型路由
                 String protocol = url.getParameter(Constants.ROUTER_KEY, ScriptRouterFactory.NAME); // 将原类型转为协议
                 String type = null; // 使用文件后缀做为类型
                 //获取路由规则文件路径
                 String path = url.getPath();
                 if (path != null) {
                     int i = path.lastIndexOf('.');
                     if (i > 0) {
                         type = path.substring(i + 1);
                     }
                 }
                 //通过路径，读取脚本文件的内容
                 String rule = IOUtils.read(new FileReader(new File(url.getAbsolutePath())));
                 //设置路由类型到protocol里，这里protocol就是script。把规则字串放在url里的rule key里，
                 URL script = url.setProtocol(protocol).addParameter(Constants.TYPE_KEY, type).addParameterAndEncoded(Constants.RULE_KEY, rule);
                 //这里routerFactory，其实是dubbo根据spi机制生成的自适应类对象，
                 //routerFactory实现的getRouter方法，会根据协议类型，自动构造相应类型路由器，下面有dubbo spi机制动态构造生成的RouterFactory接口实现类
                 //这里protocol是script
                 return routerFactory.getRouter(script);
             } catch (IOException e) {
                 throw new IllegalStateException(e.getMessage(), e);
             }
         }
     
     }
 
 spi机制动态构造生成的RouterFactory接口实现类源码：
 
     package com.alibaba.dubbo.rpc.cluster;
     
     import com.alibaba.dubbo.common.extension.ExtensionLoader;
     public class RouterFactory$Adpative implements com.alibaba.dubbo.rpc.cluster.RouterFactory {
         public com.alibaba.dubbo.rpc.cluster.Router getRouter(com.alibaba.dubbo.common.URL arg0) {
             if (arg0 == null) throw new IllegalArgumentException("url == null");
             com.alibaba.dubbo.common.URL url = arg0;
     	//根据协议类型，获取路由类型工厂类型
             String extName = url.getProtocol();
             if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.cluster.RouterFactory) name from url(" + url.toString() + ") use keys([protocol])");
             com.alibaba.dubbo.rpc.cluster.RouterFactory extension = (com.alibaba.dubbo.rpc.cluster.RouterFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.RouterFactory.class).getExtension(extName);
             return extension.getRouter(arg0);
         }
     }
