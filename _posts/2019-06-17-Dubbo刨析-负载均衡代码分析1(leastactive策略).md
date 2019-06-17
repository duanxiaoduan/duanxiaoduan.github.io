---
layout:     post
title:      dubbo负载均衡代码分析1(leastactive策略)
subtitle:   dubbo
date:       2019-06-17
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - Dubbo
---

dubbo负载均衡代码分析1(leastactive策略)


既然有集群容错，自然会有负载均衡。dubbo通过spi默认实现了4种lb策略  
分别是  
权重随机（random），实现类RandomLoadBalance  
权重轮询(roundrobin)，实现类RoundRobinLoadBalance  
最少活跃（leastactive）负载策略，实现类LeastActiveLoadBalance  
一致性hash(consistenthash)实现类ConsistentHashLoadBalance  
类关系图：

![](https://static.oschina.net/uploads/space/2017/1113/155610_uha1_146130.jpg)

4种实现都扩展了抽象类AbstractLoadBalance，  
并实现了doSelect抽象方法，  
这点和集群容错结构使用了同样的设计模式，这个doSelect方法在AbstractLoadBalance的select方法中被调用，select方法也是接口LoadBalance的唯一方法，是负载均衡的实现方法。

代码如下：

     public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
            if (invokers == null || invokers.size() == 0)
                return null;
            if (invokers.size() == 1)
                return invokers.get(0);
            //回调子类的doSelect实现，实现具体的lb策略
            return doSelect(invokers, url, invocation);
        }

dubbo负载均衡，默认是随机（random）  
这个可通过[上篇](https://my.oschina.net/u/146130/blog/1569554)提到的AbstractClusterInvoker的invoke方法实现看到，代码：  
 

    public Result invoke(final Invocation invocation) throws RpcException {
    
            checkWhetherDestroyed();
    
            LoadBalance loadbalance;
    
            List<Invoker<T>> invokers = list(invocation);
            if (invokers != null && invokers.size() > 0) {
                //从url通过key "loadbalance" 取不到值，就取默认random随机策略
                loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                        .getMethodParameter(invocation.getMethodName(), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
            } else {
                //取默认random随机策略
                loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
            }
            RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
            return doInvoke(invocation, invokers, loadbalance);
        }

但是，这篇只说，最少活跃（leastactive）负载策略。

首先想说的是，要理解最少活跃数负载策略，就要先弄明白这里的最少活跃数，指的是什么数  
先看实现代码：

    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
            int length = invokers.size(); // 总个数
            int leastActive = -1; // 最小的活跃数
            int leastCount = 0; // 相同最小活跃数的个数
            int[] leastIndexs = new int[length]; // 相同最小活跃数的下标
            int totalWeight = 0; // 总权重
            int firstWeight = 0; // 第一个权重，用于于计算是否相同
            boolean sameWeight = true; // 是否所有权重相同
            for (int i = 0; i < length; i++) {
            	Invoker<T> invoker = invokers.get(i);
    	 // 活跃数是通过RpcStatus,getStatus(invoker.getUrl(), invocation.getMethodName()).getActive()获取的。
    	 // 可以先跳过去看下文的RpcStatus类解读
                int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive(); 
                int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT); // 权重
                if (leastActive == -1 || active < leastActive) { // 发现更小的活跃数，重新开始
                    leastActive = active; // 记录最小活跃数
                    leastCount = 1; // 重新统计相同最小活跃数的个数
                    leastIndexs[0] = i; // 重新记录最小活跃数下标
                    totalWeight = weight; // 重新累计总权重
                    firstWeight = weight; // 记录第一个权重
                    sameWeight = true; // 还原权重相同标识
                } else if (active == leastActive) { // 累计相同最小的活跃数
                    leastIndexs[leastCount ++] = i; // 累计相同最小活跃数下标
                    totalWeight += weight; // 累计总权重
                    // 判断所有权重是否一样
                    if (sameWeight && i > 0 
                            && weight != firstWeight) {
                        sameWeight = false;
                    }
                }
            }
            // assert(leastCount > 0)
            if (leastCount == 1) {
                // 如果只有一个最小则直接返回
                return invokers.get(leastIndexs[0]);
            }
            if (! sameWeight && totalWeight > 0) {
                // 如果权重不相同且权重大于0则按总权重数随机
                int offsetWeight = random.nextInt(totalWeight);
                // 并确定随机值落在哪个片断上
                for (int i = 0; i < leastCount; i++) {
                    int leastIndex = leastIndexs[i];
    		//这里getWeight得到权重，不一定就是配置的，它兼容了java的warmup问题，
    		//大概意思是，如果warmup时间设置为10分钟，权重配置为100，
    		//而当前服务只启动了1分钟，那么这个方法为计算出一个值为10的新权值
    		//这其实，这会有个小问题的，应为上面计算的totalWeight是没有按warmup降权的，
    		//所以，按目前落在哪个片段上的算法，有可能一个也选不到。特别是服务刚启动时。
                    offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                    if (offsetWeight <= 0)
                        return invokers.get(leastIndex);
                }
            }
            // 如果权重相同或权重为0则均等随机
            return invokers.get(leastIndexs[random.nextInt(leastCount)]);
        }

这个方法，就是把invokers里，有最小活跃数的invoker（一个或多个）的下标，记录到leastIndexs数组里。  
如果只有一个，就直接返回，不用选了。如果有多个，再计算这其中每个的invoker的权重。  
如果权重一样，就均等随机选一个。  
如果权重不一样，就再按权重随机（random策略）从中选一个。

RpcStatus类，它是url统计类，有以下属性

        //私有静态map,存调用统计信息用的
        private static final ConcurrentMap<String, RpcStatus> SERVICE_STATISTICS = new ConcurrentHashMap<String, RpcStatus>();
        private static final ConcurrentMap<String, ConcurrentMap<String, RpcStatus>> METHOD_STATISTICS = new ConcurrentHashMap<String, ConcurrentMap<String, RpcStatus>>();
        //具体代表各个调用指标统计值
        private final ConcurrentMap<String, Object> values = new ConcurrentHashMap<String, Object>();
        //活跃数
        private final AtomicInteger active = new AtomicInteger();
        private final AtomicLong total = new AtomicLong();
        private final AtomicInteger failed = new AtomicInteger();
        private final AtomicLong totalElapsed = new AtomicLong();
        private final AtomicLong failedElapsed = new AtomicLong();
        private final AtomicLong maxElapsed = new AtomicLong();
        private final AtomicLong failedMaxElapsed = new AtomicLong();
        private final AtomicLong succeededMaxElapsed = new AtomicLong();

  上文提到的连接数是通过下面方法得到  
  public int getActive() {  
        return active.get();  
  }  
  而能改变这个active值的只有下面两个方法  
   private static void beginCount(RpcStatus status) {  
    }  
  private static void endCount(RpcStatus status, long elapsed, boolean succeeded) {  
    }

这两个方法，什么时候被调用的呢  
通过源码find usages发现在ActiveLimitFilter和ExecuteLimitFilter两个过滤器中调用的。  
通过注解知道，ExecuteLimitFilter是服务端过滤器，ActiveLimitFilter是客户端过滤器（以后可以写专门介绍过滤器的）  
我们这边是调用方，应该用ActiveLimitFilter。而启用这个过滤器，则需要在调用方应用上配置filter="activelimit"  
由于dubbo默认调用是没有启用这个过滤器的，所以要想使用最少活跃（leastactive）负载策略，需要配置启用这个activelimit过滤器。看下过滤器，唯一一个方法：

     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
            URL url = invoker.getUrl();
            String methodName = invocation.getMethodName();
            int max = invoker.getUrl().getMethodParameter(methodName, Constants.ACTIVES_KEY, 0);
            RpcStatus count = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName());
            if (max > 0) {
                long timeout = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, 0);
                long start = System.currentTimeMillis();
                long remain = timeout;
                int active = count.getActive();
                if (active >= max) {
                    synchronized (count) {
                        while ((active = count.getActive()) >= max) {
                            try {
                                count.wait(remain);
                            } catch (InterruptedException e) {
                            }
                            long elapsed = System.currentTimeMillis() - start;
                            remain = timeout - elapsed;
                            if (remain <= 0) {
                                throw new RpcException("Waiting concurrent invoke timeout in client-side for service:  "
                                                       + invoker.getInterface().getName() + ", method: "
                                                       + invocation.getMethodName() + ", elapsed: " + elapsed
                                                       + ", timeout: " + timeout + ". concurrent invokes: " + active
                                                       + ". max concurrent invoke limit: " + max);
                            }
                        }
                    }
                }
            }
            try {
    	    //业务方法调用前，调用beginCoun
                long begin = System.currentTimeMillis();
                RpcStatus.beginCount(url, methodName);
                try {
                    Result result = invoker.invoke(invocation);
    		//调用成功后，返回后，调用endCount
                    RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, true);
                    return result;
                } catch (RuntimeException t) {
    	        //调用失败后结束统计
                    RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, false);
                    throw t;
                }
            } finally {
                if(max>0){
                    synchronized (count) {
                        count.notify();
                    } 
                }
            }
        }

 再回头看下两个方法的具体实现：

         /**
         * @param url
         */
        public static void beginCount(URL url, String methodName) {
        	//dubbo这里，把调用的url或方法名做key ,RpcStatus对象作为value是方法，静态map属性里
    	//通过这样把调用信息存起来。
    	//它可以统计一个url被调用的信息，也可以记录一个url里某个方法被调用的统计信息
            beginCount(getStatus(url));
            beginCount(getStatus(url, methodName));
        }
        
        private static void beginCount(RpcStatus status) {
            status.active.incrementAndGet();//active值加1
        }
    
       //beginCount的作用，可以理解某个方法调用前，它对应的active数目加1
    private static void endCount(RpcStatus status, long elapsed, boolean succeeded) {
            status.active.decrementAndGet();//某个方法正调用结束，它对应的active减一
            status.total.incrementAndGet();
            status.totalElapsed.addAndGet(elapsed);
            if (status.maxElapsed.get() < elapsed) {
                status.maxElapsed.set(elapsed);
            }
            if (succeeded) {
                if (status.succeededMaxElapsed.get() < elapsed) {
                    status.succeededMaxElapsed.set(elapsed);
                }
            } else {
                status.failed.incrementAndGet();
                status.failedElapsed.addAndGet(elapsed);
                if (status.failedMaxElapsed.get() < elapsed) {
                    status.failedMaxElapsed.set(elapsed);
                }
            }
        }
    //endCount的作用，可以理解某个方法调用结束后，它对应的active数目减1

所以，这个active数目就是表示，某个方法当前有多少正在执行（开始调用，但还没有返回）  
也可以说最少活跃（leastactive）负载策略，就选择那些返回比较快的主机，或者本机调用较少的主机。