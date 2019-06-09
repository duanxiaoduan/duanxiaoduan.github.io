---
layout:     post
title:      Dubbo源码分析（六）Dubbo通信的编码解码机制
subtitle:   dubbo
date:       2019-06-09
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - Dubbo
---

Dubbo源码分析（六）Dubbo通信的编码解码机制
==========================

TCP的粘包拆包问题
----------

我们知道Dubbo的网络通信框架Netty是基于TCP协议的，TCP协议的网络通信会存在粘包和拆包的问题，先看下为什么会出现粘包和拆包

*   当要发送的数据大于TCP发送缓冲区剩余空间大小，将会发生拆包
*   待发送数据大于MSS（最大报文长度），TCP在传输前将进行拆包
*   要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据一次发送出去，将会发生粘包
*   接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包

以上四点基本上是出现粘包和拆包的原因，业界的解决方法一般有以下几种：

*   将每个数据包分为消息头和消息体，消息头中应该至少包含数据包的长度，这样接收端在接收到数据后，就知道每一个数据包的实际长度了（Dubbo就是这种方案）
*   消息定长，每个数据包的封装为固定长度，不够补0
*   在数据包的尾部设置特殊字符，比如FTP协议

Dubbo消息协议头规范
------------

在dubbo.io官网上找到一张图，协议头约定

dubbo的消息头是一个定长的 16个字节的数据包：

*   magic High & Magic Low：2byte：0-7位 8-15位:类似java字节码文件里的魔数，用来判断是不是dubbo协议的数据包，就是一个固定的数字
*   Serialization id:1byte:16-20位：序列id,21 event,22 two way 一个标志位，是单向的还是双向的,23 请求或响应标识，
*   status:1byte: 24-31位:状态位,设置请求响应状态，request为空，response才有值
*   Id(long)：8byte:32-95位:每一个请求的唯一识别id（由于采用异步通讯的方式，用来把请求request和返回的response对应上）
*   data length:4byte:96-127位:消息体长度，int 类型

看完这张图，大致可以理解Dubbo通信协议解决的问题，Dubbo采用消息头和消息体的方式来解决粘包拆包，并在消息头中放入了一个唯一Id来解决异步通信关联request和response的问题，下面以一次调用为入口分为四个部分来看下源码具体实现

Comsumer端请求编码
-------------

    private class InternalEncoder extends OneToOneEncoder {
            @Override
            protected Object encode(ChannelHandlerContext ctx, Channel ch, Object msg) throws Exception {
                com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer =
                        com.alibaba.dubbo.remoting.buffer.ChannelBuffers.dynamicBuffer(1024);
                NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
                try {
                    codec.encode(channel, buffer, msg);
                } finally {
                    NettyChannel.removeChannelIfDisconnected(ch);
                }
                return ChannelBuffers.wrappedBuffer(buffer.toByteBuffer());
            }
        }
    复制代码

这个InternalEncoder是一个NettyCodecAdapter的内部类，我们看到codec.encode(channel, buffer, msg)这里，这个时候codec=DubboCountCodec，这个是在构造方法中传入的，DubboCountCodec.encode-->ExchangeCodec.encode-->ExchangeCodec.encodeRequest

    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
            //获取序列化方式，默认是Hessian序列化
            Serialization serialization = getSerialization(channel);
            // new了一个16位的byte数组，就是request的消息头
            byte[] header = new byte[HEADER_LENGTH];
            // 往消息头中set magic数字，这个时候header中前2个byte已经填充
            Bytes.short2bytes(MAGIC, header);
    
            // set request and serialization flag.第三个byte已经填充
            header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());
    
            if (req.isTwoWay()) header[2] |= FLAG_TWOWAY;
            if (req.isEvent()) header[2] |= FLAG_EVENT;
    
            // set request id.这个时候是0
            Bytes.long2bytes(req.getId(), header, 4);
    
            // 编码 request data.
            int savedWriteIndex = buffer.writerIndex();
            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
            ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
            //序列化
            ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
            if (req.isEvent()) {
                encodeEventData(channel, out, req.getData());
            } else {
                //编码消息体数据
                encodeRequestData(channel, out, req.getData());
            }
            out.flushBuffer();
            bos.flush();
            bos.close();
            int len = bos.writtenBytes();
            checkPayload(channel, len);
            //在消息头中设置消息体长度
            Bytes.int2bytes(len, header, 12);
    
            // write
            buffer.writerIndex(savedWriteIndex);
            buffer.writeBytes(header); // write header.
            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
        }
    复制代码

就是这方法，对request请求进行了编码操作，具体操作我写在代码的注释中，就是刚刚我们分析的消息头的代码实现

Provider端请求解码
-------------

看到NettyCodecAdapter中的InternalDecoder这个类的messageReceived方法，这里就是Provider端对于Consumer端的request请求的解码

    public void messageReceived(ChannelHandlerContext ctx, MessageEvent event) throws Exception {
                ···
                try {
                    // decode object.
                    do {
                        saveReaderIndex = message.readerIndex();
                        try {
                            msg = codec.decode(channel, message);
                        } catch (IOException e) {
                            buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
                            throw e;
                        }
                ···
    复制代码

进入DubboCountCodec.decode--ExchangeCodec.decode

    // 检查 magic number.
            if (readable > 0 && header[0] != MAGIC_HIGH
               ···
            }
            // check 长度如果小于16位继续等待
            if (readable < HEADER_LENGTH) {
                return DecodeResult.NEED_MORE_INPUT;
            }
            // get 消息体长度
            int len = Bytes.bytes2int(header, 12);
            checkPayload(channel, len);
            //消息体长度+消息头的长度
            int tt = len + HEADER_LENGTH;
            //如果总长度小于tt，那么返回继续等待
            if (readable < tt) {
                return DecodeResult.NEED_MORE_INPUT;
            }
    
            // limit input stream.
            ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);
    
            try {
                //解析消息体内容
                return decodeBody(channel, is, header);
            } finally {
               ···
        }
    复制代码

这里对于刚刚的request进行解码操作，具体操作步骤写在注释中了

Provider端响应编码
-------------

当服务端执行完接口调用，看下服务端的响应编码，和消费端不一样的地方是，服务端进入的是ExchangeCodec.encodeResponse方法

    try {
                //获取序列化方式 默认Hession协议
                Serialization serialization = getSerialization(channel);
                // 初始化一个16位的header
                byte[] header = new byte[HEADER_LENGTH];
                // set magic 数字
                Bytes.short2bytes(MAGIC, header);
                // set request and serialization flag.
                header[2] = serialization.getContentTypeId();
                if (res.isHeartbeat()) header[2] |= FLAG_EVENT;
                // set response status.这里返回的是OK
                byte status = res.getStatus();
                header[3] = status;
                // set request id.
                Bytes.long2bytes(res.getId(), header, 4);
    
                int savedWriteIndex = buffer.writerIndex();
                buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
                ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
                ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
                // 编码返回消息体数据或者错误数据
                if (status == Response.OK) {
                    if (res.isHeartbeat()) {
                        encodeHeartbeatData(channel, out, res.getResult());
                    } else {
                        encodeResponseData(channel, out, res.getResult());
                    }
                } else out.writeUTF(res.getErrorMessage());
                out.flushBuffer();
                bos.flush();
                bos.close();
    
                int len = bos.writtenBytes();
                checkPayload(channel, len);
                Bytes.int2bytes(len, header, 12);
                // write
                buffer.writerIndex(savedWriteIndex);
                buffer.writeBytes(header); // write header.
                buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
            } catch (Throwable t) {
                // 发送失败信息给Consumer，否则Consumer只能等超时了
                if (!res.isEvent() && res.getStatus() != Response.BAD_RESPONSE) {
                    try {
                        // FIXME 在Codec中打印出错日志？在IoHanndler的caught中统一处理？
                        logger.warn("Fail to encode response: " + res + ", send bad_response info instead, cause: " + t.getMessage(), t);
    
                        Response r = new Response(res.getId(), res.getVersion());
                        r.setStatus(Response.BAD_RESPONSE);
                        r.setErrorMessage("Failed to send response: " + res + ", cause: " + StringUtils.toString(t));
                        channel.send(r);
    
                        return;
                    } catch (RemotingException e) {
                        logger.warn("Failed to send bad_response info back: " + res + ", cause: " + e.getMessage(), e);
                    }
                }
                // 重新抛出收到的异常
                ···
        }
    复制代码

基本上和消费方请求编码一样，多了一个步骤，一个是在消息头中加入了一个状态位，第二个是如果发送有异常，则继续发送失败信息给Consumer，否则Consumer只能等超时了

Conmsuer端响应解码
-------------

和上面的解码一样，具体操作是在ExchangeCodec.decode--DubboCodec.decodeBody中