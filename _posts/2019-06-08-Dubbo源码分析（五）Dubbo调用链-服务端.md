---
layout:     post
title:      Dubbo源码分析（五）Dubbo调用链-服务端
subtitle:   dubbo
date:       2019-06-08
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - Dubbo
---

Dubbo源码分析（五）Dubbo调用链-服务端
========================

这篇来分析Dubbo服务端接收消费端请求调用的过程，先看一张调用链的整体流程图

上面绿色的部分为服务端接收请求部分，大体流程是从ThreadPool-->server-->Exporter-->Filter-->Invoker-->Impl，下面来看源码

源码入口
----

我们知道Dubbo默认是通过Netty进行网络传输，所以这里的源码入口我们应该找到NettyHandler的接收消息的方法

    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
            try {
                handler.received(channel, e.getMessage());
            } finally {
                NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
            }
        }
    复制代码

这里就是服务端接收消费端发送请求的地方，进入handler.received方法，在AbstractPeer类中

    public void received(Channel ch, Object msg) throws RemotingException {
            if (closed) {
                return;
            }
            handler.received(ch, msg);
        }
    复制代码

进入到handler.received,最终我们进入AllChannelHandler.received方法中

    public void received(Channel channel, Object message) throws RemotingException {
            ExecutorService cexecutor = getExecutorService();
            try {
                cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
            } catch (Throwable t) {
                throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
            }
        }
    复制代码

这里从线程池总来执行请求，我们看到ChannelEventRunnable类，这个类中一定有一个run方法，我们看一下

    public void run() {
            switch (state) {
                case CONNECTED:
                    try {
                        handler.connected(channel);
                    } catch (Exception e) {
                        logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
                    }
                    break;
                case DISCONNECTED:
                    try {
                        handler.disconnected(channel);
                    } catch (Exception e) {
                        logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
                    }
                    break;
                case SENT:
                    try {
                        handler.sent(channel, message);
                    } catch (Exception e) {
                        logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                                + ", message is " + message, e);
                    }
                    break;
                case RECEIVED:
                    try {
                        handler.received(channel, message);
                    } catch (Exception e) {
                        logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                                + ", message is " + message, e);
                    }
                    break;
                case CAUGHT:
                    try {
                        handler.caught(channel, exception);
                    } catch (Exception e) {
                        logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                                + ", message is: " + message + ", exception is " + exception, e);
                    }
                    break;
                default:
                    logger.warn("unknown state: " + state + ", message is " + message);
            }
        }
    复制代码

这里有很多种类型的请求，我们这次是RECEIVED请求，进入handler.received(channel, message)方法，此时的handler=DecodeHandler,先进行解码，这里的内容暂时不说，放在后面的解编码一起说，继续进入HeaderExchangeHandler.received方法，

    public void received(Channel channel, Object message) throws RemotingException {
            ···
                            Response response = handleRequest(exchangeChannel, request);
                            channel.send(response);
                        } else {
                            handler.received(exchangeChannel, request.getData());
                        }
                    }
                } ···
            } finally {
                HeaderExchangeChannel.removeChannelIfDisconnected(channel);
            }
        }
    复制代码

执行handleRequest(exchangeChannel, request)方法，这里是网络通信接收处理方法，继续走，进入DubboProtocol.reply方法

    public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
                if (message instanceof Invocation) {
                    Invocation inv = (Invocation) message;
                    Invoker<?> invoker = getInvoker(channel, inv);
                    ···
                    return invoker.invoke(inv);
                }
               ···
            }
    复制代码

进入getInvoker方法，

     Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
           ···
            DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);
           ···
            return exporter.getInvoker();
        }
    复制代码

从exporterMap中获取所需的exporter，还记不记得这个exporterMap是什么时候放入值的，在服务端暴露接口的时候，这个是在第二篇中有提到过

然后执行exporter.getInvoker()，现在我们要将exporter转化为invoeker对象，我们拿到这个invoker对象，并执行invoker.invoke(inv)方法，然后经过8个Filter，最终进入JavassistProxyFactory.AbstractProxyInvoker.doInvoke方法，

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

还记得这段代码吗？这个wrapper是什么？下面这个截图是在服务暴露的时候，

这个时候执行 wrapper.invokeMethod方法，此时的wrapper就是之前我们动态生成的一个wrapper包装类，然后进入真正的实现类DemoServiceImpl.sayHello方法，执行完成之后，回到HeaderExchangeHandler.received方法

    channel.send(response)
    复制代码

最终把结果通过ChannelFuture future = channel.write(message)发回consumer

消费端接收服务端发送的执行结果
---------------

我们先进入NettyHandler.messageReceived方法，再执行handler.received(channel, e.getMessage())，这个和上面的代码是一样的，继续进入MultiMessageHandler.received方法，继续进入ChannelEventRunnable线程池，继续进入DecodeHandler.received解码，最终进入DefaultFuture.doReceived方法

    private void doReceived(Response res) {
            lock.lock();
            try {
                response = res;
                if (done != null) {
                    done.signal();
                }
            } finally {
                lock.unlock();
            }
            if (callback != null) {
                invokeCallback(callback);
            }
        }
    复制代码

这里用到了一个Condition，就是这个done，这里的done.signal的作用是什么呢？既然有signal方法，一定还有done.await方法,我们看到这个get方法，

    public Object get(int timeout) throws RemotingException {
            if (timeout <= 0) {
                timeout = Constants.DEFAULT_TIMEOUT;
            }
            if (!isDone()) {
                long start = System.currentTimeMillis();
                lock.lock();
                try {
                    while (!isDone()) {
                        done.await(timeout, TimeUnit.MILLISECONDS);
                        if (isDone() || System.currentTimeMillis() - start > timeout) {
                            break;
                        }
                    }
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } finally {
                    lock.unlock();
                }
                if (!isDone()) {
                    throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
                }
            }
            return returnFromResponse();
        }
    复制代码

这个里面有一个done.await方法，貌似这个get方法有点熟悉，在哪里见过呢，在DubboInvoker.doInvoke()方法中的最后一行,贴代码

    return (Result) currentClient.request(inv, timeout).get();
    复制代码

这里拿到消费端发起请求之后，调用了get方法，这个get方法一直阻塞，直到服务端返回结果调用done.signal()，这个也是Dubbo的异步转同步机制的实现方式，至此，Dubbo的调用链就分析完了