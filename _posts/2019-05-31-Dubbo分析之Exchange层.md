---
layout:     post
title:      Dubbo分析之Exchange层
subtitle:   dubbo
date:       2019-05-31
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - dubbo
---
前言
--

本文介绍Exchange层，此层官方介绍为信息交换层：封装请求响应模式，同步转异步，以 Request, Response 为中心，扩展接口为 Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer；下面分别进行介绍

Exchanger分析
-----------

Exchanger是此层的核心接口类，提供了connect()和bind()接口，分别返回ExchangeClient和ExchangeServer；dubbo提供了此接口的默认实现类HeaderExchanger，代码如下：

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

在实现类中在connect和bind中分别实例化了HeaderExchangeClient和HeaderExchangeServer，传入的参数是Transporters，可以认为这里就是Transport层的入口类；这里的ExchangeClient/ExchangeServer其实就是对Client/Server的包装，同时传入了自己的ChannelHandler；ChannelHandler已经在Transport层介绍过了，提供了连接建立，连接端口，发送请求，接受请求等接口；已默认使用的Netty为例，这里就是对NettyClient和NettyServer的包装，同时传入DecodeHandler，在NettyHandler中被调用；

ExchangeClient分析
----------------

ExchangeClient本身也继承于Client，同时也继承于ExchangeChannel：

    public interface ExchangeClient extends Client, ExchangeChannel {
     
    }
     
    public interface ExchangeChannel extends Channel {
     
        ResponseFuture request(Object request) throws RemotingException;
     
        ResponseFuture request(Object request, int timeout) throws RemotingException;
     
        ExchangeHandler getExchangeHandler();
     
        @Override
        void close(int timeout);
     
    }

ExchangeChannel负责将上层的data包装成Request，然后发送给Transport层；具体的逻辑在HeaderExchangeChannel中：

    public ResponseFuture request(Object request, int timeout) throws RemotingException {
           if (closed) {
               throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
           }
           // create request.
           Request req = new Request();
           req.setVersion(Version.getProtocolVersion());
           req.setTwoWay(true);
           req.setData(request);
           DefaultFuture future = new DefaultFuture(channel, req, timeout);
           try {
               channel.send(req);
           } catch (RemotingException e) {
               future.cancel();
               throw e;
           }
           return future;
       }

创建了一个Request，在构造器中同时会产生一个RequestId；设置了协议版本，是否双向通信，最后设置了真实的业务数据；接下来实例化了一个DefaultFuture类，此类实现了同步转异步的方式，channel调用send发送请求之后，不需要等待结果，直接将DefaultFuture返回给上层，上层可以通过调用DefaultFuture的get方法来获取响应，get方法会阻塞等待获取服务器的响应才会返回；Client接收消息在handler里面，比如Netty在NettyHandler里面messageReceived方法介绍响应消息，NettyHandler最终会调用上面传入的DecodeHandler，DecodeHandler会先判断一下是否已经解码，如果解码就直接调用HeaderExchangeHandler，默认已经设置了编码解码器，所以会直接调用HeaderExchangeHandler里面的received方法：

    public void received(Channel channel, Object message) throws RemotingException {
           channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
           ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
           try {
               if (message instanceof Request) {
                   // handle request.
                   Request request = (Request) message;
                   if (request.isEvent()) {
                       handlerEvent(channel, request);
                   } else {
                       if (request.isTwoWay()) {
                           Response response = handleRequest(exchangeChannel, request);
                           channel.send(response);
                       } else {
                           handler.received(exchangeChannel, request.getData());
                       }
                   }
               } else if (message instanceof Response) {
                   handleResponse(channel, (Response) message);
               } else if (message instanceof String) {
                   if (isClientSide(channel)) {
                       Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                       logger.error(e.getMessage(), e);
                   } else {
                       String echo = handler.telnet(channel, (String) message);
                       if (echo != null && echo.length() > 0) {
                           channel.send(echo);
                       }
                   }
               } else {
                   handler.received(exchangeChannel, message);
               }
           } finally {
               HeaderExchangeChannel.removeChannelIfDisconnected(channel);
           }
       }

服务端和客户端都会使用此方法，这里是客户端接受的是Response，直接调用handleResponse方法：

    static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            DefaultFuture.received(channel, response);
        }
    }

接收到响应之后，再去告诉DefaultFuture已经收到响应，DefaultFuture本身存放了requestId对应DefaultFuture的一个ConcurrentHashMap；具体怎么映射过去，Response也包含一个responseId，此responseId和requestId是相同的；

    private final Lock lock = new ReentrantLock();
    private final Condition done = lock.newCondition();
       
    public static void received(Channel channel, Response response) {
          try {
              DefaultFuture future = FUTURES.remove(response.getId());
              if (future != null) {
                  future.doReceived(response);
              } else {
                  logger.warn("The timeout response finally returned at "
                          + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                          + ", response " + response
                          + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                          + " -> " + channel.getRemoteAddress()));
              }
          } finally {
              CHANNELS.remove(response.getId());
          }
      }
       
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

通过responseId获取了之前请求时创建的DefaultFuture，然后再更新DefaultFuture内部的response对象，更新完之后在调用Condition的signal方法，用户唤起通过DefaultFuture的get方法获取响应的阻塞线程：

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

可以发现阻塞要么被获取被signal方法唤醒，要么等待超时；以上大致是客户端发送获取响应的流程，下面看看服务器端流程

ExchangeServer分析
----------------

ExchangeServer继承于Server，同时提供了两个包装服务端Channel的方法

    public interface ExchangeServer extends Server {
     
        Collection<ExchangeChannel> getExchangeChannels();
     
        ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress);
    }

服务器端主要用于接收Request消息，然后处理消息，最后把响应发送给客户端，相关接收消息已经在上面介绍过了，同样是在HeaderExchangeHandler里面的received方法中，只不过这里的消息类型为Request；

    Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
          Response res = new Response(req.getId(), req.getVersion());
          if (req.isBroken()) {
              Object data = req.getData();
     
              String msg;
              if (data == null) msg = null;
              else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
              else msg = data.toString();
              res.setErrorMessage("Fail to decode request due to: " + msg);
              res.setStatus(Response.BAD_REQUEST);
     
              return res;
          }
          // find handler by message class.
          Object msg = req.getData();
          try {
              // handle data.
              Object result = handler.reply(channel, msg);
              res.setStatus(Response.OK);
              res.setResult(result);
          } catch (Throwable e) {
              res.setStatus(Response.SERVICE_ERROR);
              res.setErrorMessage(StringUtils.toString(e));
          }
          return res;
      }

首先创建了一个Response，并且指定responseId为requestId，方便在客户端定位到具体的DefaultFuture；然后调用handler的reply方法处理消息，返回结果，如何处理的将在后面的protocol层介绍，大致就是通过Request的信息，反射调用Server端的服务，然后返回结果，然后将结果放入Response对象中，通过channel将消息发送客户端；

总结
--

本文介绍了Exchange层的大体流程，围绕Exchanger，ExchangeClient和ExchangeServer展开；请求封装成Request，响应封装成Response，客户端通过异步的方式接收服务器请求；

示例代码地址
------

[https://github.com/ksfzhaohui...](https://github.com/ksfzhaohui/blog)  
[https://gitee.com/OutOfMemory...](https://gitee.com/OutOfMemory/blog)
