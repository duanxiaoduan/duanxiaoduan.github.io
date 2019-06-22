---
layout:     post
title:      路由代码分析3（condition路由器）
subtitle:   dubbo
date:       2019-06-22
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - dubbo
---

 
dubbo路由代码分析3（condition路由器）


这篇说说，dubbo condition类型路由器的路由解析和执行过程  
由[https://my.oschina.net/u/146130/blog/1581069](https://my.oschina.net/u/146130/blog/1581069)这篇我们可以看到  
condition类型路由器是由condition路由工厂获取，源码如下

    public class ConditionRouterFactory implements RouterFactory {
    
        public static final String NAME = "condition";
        //返回ConditionRouter类型路由器
        public Router getRouter(URL url) {
            return new ConditionRouter(url);
        }
    
    }

ConditionRouterFactory是RouterFactory 通过spi机制的一种实现。具体看下，condition路由器的源码,这里先贴出两个个方法，一个构造方法，一个是路由方法

    /**
     * ConditionRouter 类生命
     * 实现了Comparable接口，是为了路由排序用的
     * @author william.liangf
     */
    public class ConditionRouter implements Router, Comparable<Router>{}
     
    

属性

        private static final Logger logger = LoggerFactory.getLogger(ConditionRouter.class);
        private static Pattern ROUTE_PATTERN = Pattern.compile("([&!=,]*)\\s*([^&!=,\\s]+)");
        //路由器的信息来源:url
        private final URL url;
        //路由器优先级，在多个路由排序用的
        private final int priority;
        //是否强制执行路由规则，哪怕没有合适的invoker
        private final boolean force;
        //存放consumer路由规则
        private final Map<String, MatchPair> whenCondition;
        //存放provider路由规则
        private final Map<String, MatchPair> thenCondition;
    /***
         * pulibc 构造函数，为各个属性赋值
         * @param url
         */
        public ConditionRouter(URL url) {
            this.url = url;
            this.priority = url.getParameter(Constants.PRIORITY_KEY, 0);
            this.force = url.getParameter(Constants.FORCE_KEY, false);
            try {
                //通过rule key获取路由规则字串
                String rule = url.getParameterAndDecoded(Constants.RULE_KEY);
                if (rule == null || rule.trim().length() == 0) {
                    throw new IllegalArgumentException("Illegal route rule!");
                }
                //把字符串里的"consumer." "provider." 替换掉，方便解析
                rule = rule.replace("consumer.", "").replace("provider.", "");
                int i = rule.indexOf("=>");
                //以"=>"为分割线，前面是consumer规则，后面是provider 规则
                String whenRule = i < 0 ? null : rule.substring(0, i).trim();
                String thenRule = i < 0 ? rule.trim() : rule.substring(i + 2).trim();
                //parseRule 方法解析规则，放在Map<String, MatchPair>里
                Map<String, MatchPair> when = StringUtils.isBlank(whenRule) || "true".equals(whenRule) ? new HashMap<String, MatchPair>() : parseRule(whenRule);
                Map<String, MatchPair> then = StringUtils.isBlank(thenRule) || "false".equals(thenRule) ? null : parseRule(thenRule);
                // NOTE: When条件是允许为空的，外部业务来保证类似的约束条件
                //解析构造的规则放在condition变量里
                this.whenCondition = when;
                this.thenCondition = then;
            } catch (ParseException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
    
        //代码略...
           /***
         * 路由方法,执行路由规则
         * @param invokers
         * @param url        refer url
         * @param invocation
         * @param <T>
         * @return
         * @throws RpcException
         */
        public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation)
                throws RpcException {
            if (invokers == null || invokers.size() == 0) {
                return invokers;
            }
            try {
                //前置条件不匹配，说明consumer不在限制之列。说明，路由不针对当前客户，这样就全部放行，所有提供者都可以调用。
                //这是consumer的url
                if (!matchWhen(url, invocation)) {
                    return invokers;
                }
                List<Invoker<T>> result = new ArrayList<Invoker<T>>();
                //thenCondition为null表示拒绝一切请求
                if (thenCondition == null) {
                    logger.warn("The current consumer in the service blacklist. consumer: " + NetUtils.getLocalHost() + ", service: " + url.getServiceKey());
                    return result;
                }
                for (Invoker<T> invoker : invokers) {
                    //调用放匹配才放行。服务提供者，只要符合路由才，能提供服务，这里的url改为invoker.getUrl()
                    //都不匹配，这里result可能是空的。
                    if (matchThen(invoker.getUrl(), url)) {
                        result.add(invoker);
                    }
                }
                if (result.size() > 0) {
                    return result;
                } else if (force) {
                    //force强制执行路由。哪怕result是空的，也要返回给上层方法。如果为false,最后放回所有的invokers，等于不执行路由
                    logger.warn("The route result is empty and force execute. consumer: " + NetUtils.getLocalHost() + ", service: " + url.getServiceKey() + ", router: " + url.getParameterAndDecoded(Constants.RULE_KEY));
                    return result;
                }
            } catch (Throwable t) {
                logger.error("Failed to execute condition router rule: " + getUrl() + ", invokers: " + invokers + ", cause: " + t.getMessage(), t);
            }
            return invokers;
        }

 总的来说，构造函数，初始化路由规则。路由方法，根据路由规则对，调用方（一个）和服务提供方（多个）执行路由规则。  
 让符合规则的调用方，可以调用，  
 让不符合规则的调用方不能调用。  
 让符合规则的服务提供方，留着服务提供者列表。  
 让不符合路由规则的服务提供方，从服务者列表中除去。

 先看下，存放路由规则的数据结构。也就是下面量个属性  
     //存放consumer路由规则  
    private final Map<String, MatchPair> whenCondition;  
    //存放provider路由规则  
    private final Map<String, MatchPair> thenCondition;  
 我以，如下的路由规则为例

     "host !=4.4.4.4 & host = 2.2.2.2,1.1.1.1,3.3.3.3 &method =sayHello => host = 1.2.3.4&host !=4.4.4.4"

  
 看到经过规则解析后，存放在上面两个对象的内容如下图：

![](https://static.oschina.net/uploads/space/2017/1219/175449_B5qe_146130.png)

 可以看到，路由条件，可分为host和method的两类。  
 每一类，又可分为允许类(match)和不允许(diamatch)两类规则，  
 每一类规则有，可以包含多条路由信息。 路由规则存放，通过，Map结合MatchPair数据结构完成。  
MatchPair结构如下：

    /***
         * MatchPiar 数据结构，它包含类具体匹配方法isMatch
         * 是个私有，静态类，为什么静态？？
         */
        private static final class MatchPair {
            //允许列表,内部用Set集合类型，防止重复。
            final Set<String> matches = new HashSet<String>();
            //拒绝规则
            final Set<String> mismatches = new HashSet<String>();
            //具体执行匹配规则的方法
            private boolean isMatch(String value, URL param) {
                //只有允许项目
                if (matches.size() > 0 && mismatches.size() == 0) {
                    for (String match : matches) {
                        //value只要满足一项，允许，就为匹配，通过
                        if (UrlUtils.isMatchGlobPattern(match, value, param)) {
                            return true;
                        }
                    }
                    return false;
                }
                  //只有不允许项目
                if (mismatches.size() > 0 && matches.size() == 0) {
                    //value是要满足一项，不允许。就是不匹配
                    for (String mismatch : mismatches) {
                        if (UrlUtils.isMatchGlobPattern(mismatch, value, param)) {
                            return false;
                        }
                    }
                    //必须全部不满足不匹配。才通过
                    return true;
                }
    
                if (matches.size() > 0 && mismatches.size() > 0) {
                    //when both mismatches and matches contain the same value, then using mismatches first
                    //当匹配项和不匹配项包含同样的值，不匹配项优先
                    for (String mismatch : mismatches) {
                        if (UrlUtils.isMatchGlobPattern(mismatch, value, param)) {
                            return false;
                        }
                    }
                    for (String match : matches) {
                        if (UrlUtils.isMatchGlobPattern(match, value, param)) {
                            return true;
                        }
                    }
                    return false;
                }
                return false;
            }
        }
    

路由规则的解析，由parseRule方法完成。把规则字符串放到condition对象里。

    /**
         * 解析规则字串解析后放到map里
         * 这里有个数据结构类MatchPair 用set放着，允许规则matches，和不允许规则mismatches，他们都是set结构
         * @param rule
         * @return
         * @throws ParseException
         */
        private static Map<String, MatchPair> parseRule(String rule)
                throws ParseException {
            Map<String, MatchPair> condition = new HashMap<String, MatchPair>();
            if (StringUtils.isBlank(rule)) {
                return condition;
            }
            // 匹配或不匹配Key-Value对
            MatchPair pair = null;
            // 多个Value值
            Set<String> values = null;
            //用java的正则表达式Pattern的matcher去分割字符串到Matcher类，然后逐个匹配。
            //这个要熟悉正则，和Matcher，Pattern
            final Matcher matcher = ROUTE_PATTERN.matcher(rule);
            while (matcher.find()) { //逐个匹配
                String separator = matcher.group(1);
                String content = matcher.group(2);
                // 表达式开始
                if (separator == null || separator.length() == 0) {
                    pair = new MatchPair();
                    condition.put(content, pair);
                }
                // KV开始
                else if ("&".equals(separator)) {
                    if (condition.get(content) == null) {
                        pair = new MatchPair();
                        condition.put(content, pair);
                    } else {
                        pair = condition.get(content);
                    }
                }
                // KV的Value部分开始
                else if ("=".equals(separator)) {
                    if (pair == null)
                        throw new ParseException("Illegal route rule \""
                                + rule + "\", The error char '" + separator
                                + "' at index " + matcher.start() + " before \""
                                + content + "\".", matcher.start());
    
                    values = pair.matches;
                    values.add(content);
                }
                // KV的Value部分开始
                else if ("!=".equals(separator)) {
                    if (pair == null)
                        throw new ParseException("Illegal route rule \""
                                + rule + "\", The error char '" + separator
                                + "' at index " + matcher.start() + " before \""
                                + content + "\".", matcher.start());
    
                    values = pair.mismatches;
                    values.add(content);
                }
                // KV的Value部分的多个条目
                else if (",".equals(separator)) { // 如果为逗号表示
                    if (values == null || values.size() == 0)
                        throw new ParseException("Illegal route rule \""
                                + rule + "\", The error char '" + separator
                                + "' at index " + matcher.start() + " before \""
                                + content + "\".", matcher.start());
                    values.add(content);
                } else {
                    throw new ParseException("Illegal route rule \"" + rule
                            + "\", The error char '" + separator + "' at index "
                            + matcher.start() + " before \"" + content + "\".", matcher.start());
                }
            }
            return condition;
        }
    

然后是，理由规则的执行。

    /***
         * 对服务调用方的匹配
         * @param url
         * @param invocation
         * @return
         */
        boolean matchWhen(URL url, Invocation invocation) {
            return whenCondition == null || whenCondition.isEmpty() || matchCondition(whenCondition, url, null, invocation);
        }
    
        /***
         * 对服务提供者的匹配
         * @param url
         * @param param
         * @return
         */
        private boolean matchThen(URL url, URL param) {
            return !(thenCondition == null || thenCondition.isEmpty()) && matchCondition(thenCondition, url, param, null);
        }
    
        /***
         * 具体执行匹配规则，遍历condition里所有的规则，执行规则。
         * @param condition
         * @param url
         * @param param
         * @param invocation
         * @return
         */
        private boolean matchCondition(Map<String, MatchPair> condition, URL url, URL param, Invocation invocation) {
            Map<String, String> sample = url.toMap();
            boolean result = false;
            for (Map.Entry<String, MatchPair> matchPair : condition.entrySet()) {
                String key = matchPair.getKey();
                String sampleValue;
                //get real invoked method name from invocation
                //路由规则支持到方法级别
                if (invocation != null && (Constants.METHOD_KEY.equals(key) || Constants.METHODS_KEY.equals(key))) {
                    sampleValue = invocation.getMethodName();//获取方法名
                } else {//key 是host  获取host值
                    sampleValue = sample.get(key);
                }
                if (sampleValue != null) {
                    //调用MatchPair的isMatch方法
                    if (!matchPair.getValue().isMatch(sampleValue, param)) {
                        return false;
                    } else {
                        result = true;
                    }
                } else {
                    //not pass the condition
                    //如果sampleValue没有值，但匹配项有值。不通过
                    if (matchPair.getValue().matches.size() > 0) {
                        return false;
                    } else {
                        result = true;
                    }
                }
            }
            return result;
        }
    

最后，是路由器的排序

     /***
         * 路由器的排序
         * @param o
         * @return
         */
        public int compareTo(Router o) {
            if (o == null || o.getClass() != ConditionRouter.class) {
                return 1;
            }
            ConditionRouter c = (ConditionRouter) o;
            //如果优先级相同，通过rul字符串比较
            return this.priority == c.priority ? url.toFullString().compareTo(c.url.toFullString()) : (this.priority > c.priority ? 1 : -1);
        }