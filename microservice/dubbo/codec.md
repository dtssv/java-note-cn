在服务引用初始化客户端时，我们可以看到如下逻辑：  
```
    //DubboProtocol
    private ExchangeClient initClient(URL url) {

        //....
        //此处制定codec=dubbo，也即是DubboCodec
        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        //....
    }
```
初次之外，codec还有其他实现类，如：```thrift、telnet等```，但是此处我们只关注```DubboCodec```。   
首先查看```DubboCodec```的定义可知```public class DubboCodec extends ExchangeCodec implements Codec2```它继承了```ExchangeCodec```，重写了```protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException```，```protected void encodeRequestData(Channel channel, ObjectOutput out, Object data) throws IOException```，```protected void encodeResponseData(Channel channel, ObjectOutput out, Object data) throws IOException```三个方法，下面我们从上到下分析编码和解码流程。  
首先我们先看编码方法，也就是```ExchangeCodec.encode```：  
```
    public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
        if (msg instanceof Request) {
            encodeRequest(channel, buffer, (Request) msg);//请求
        } else if (msg instanceof Response) {
            encodeResponse(channel, buffer, (Response) msg);//响应
        } else {
            super.encode(channel, buffer, msg);//
        }
    }
```
首先先看一下```encodeRequest```：  
```
    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        //默认为hessian2
        Serialization serialization = getSerialization(channel);
        // 构建长度为16的header数组
        byte[] header = new byte[HEADER_LENGTH];
        // 魔法数，占两个字节header[0]=高八位，header[1]=低八位
        Bytes.short2bytes(MAGIC, header);

        // 设置序列化方式标识和请求标识
        header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());
        //设置单双向请求标识
        if (req.isTwoWay()) header[2] |= FLAG_TWOWAY;
        //设置事件请求标识
        if (req.isEvent()) header[2] |= FLAG_EVENT;
        //header第3位为请求/响应标识，0：请求，1：响应
        // 从第4位开始设置请求id，占8字节
        Bytes.long2bytes(req.getId(), header, 4);

        // encode request data.
        int savedWriteIndex = buffer.writerIndex();
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        if (req.isEvent()) {
            encodeEventData(channel, out, req.getData());//请求是一个事件，直接序列化写入
        } else {
            encodeRequestData(channel, out, req.getData());
        }
        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }
        bos.flush();
        bos.close();
        int len = bos.writtenBytes();
        checkPayload(channel, len);
        //从第12位开始设置请求体长度，占4字节
        Bytes.int2bytes(len, header, 12);

        // write
        buffer.writerIndex(savedWriteIndex);
        buffer.writeBytes(header); // write header.
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
    //DubboCodec
    protected void encodeRequestData(Channel channel, ObjectOutput out, Object data) throws IOException {
        RpcInvocation inv = (RpcInvocation) data;

        out.writeUTF(inv.getAttachment(Constants.DUBBO_VERSION_KEY, DUBBO_VERSION));//写入dubbo版本号，此处2.6.2
        out.writeUTF(inv.getAttachment(Constants.PATH_KEY));//写入path
        out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));//写入version

        out.writeUTF(inv.getMethodName());//写入方法名
        out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));//写入参数类型
        Object[] args = inv.getArguments();
        if (args != null)
            for (int i = 0; i < args.length; i++) {
                out.writeObject(encodeInvocationArgument(channel, inv, i));//写入参数
            }
        out.writeObject(inv.getAttachments());//写入附件
    }
```
以上就是请求的序列化过程，下面分析响应的序列化过程，响应序列化由```encodeResponse```实现，源码如下：  
```
    protected void encodeResponse(Channel channel, ChannelBuffer buffer, Response res) throws IOException {
        int savedWriteIndex = buffer.writerIndex();
        try {
            Serialization serialization = getSerialization(channel);
            // 响应头
            byte[] header = new byte[HEADER_LENGTH];
            // 设置魔法数
            Bytes.short2bytes(MAGIC, header);
            // 设置序列化标识，如果是心跳响应设置事件标识
            header[2] = serialization.getContentTypeId();
            if (res.isHeartbeat()) header[2] |= FLAG_EVENT;
            // 设置响应标识
            byte status = res.getStatus();
            header[3] = status;
            // 设置响应id
            Bytes.long2bytes(res.getId(), header, 4);

            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
            ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
            ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
            // 对响应体进行编码，如果是心跳则直接序列化，否则调用dubboCodec的encodeResponseData方法进行序列化
            if (status == Response.OK) {
                if (res.isHeartbeat()) {
                    encodeHeartbeatData(channel, out, res.getResult());
                } else {
                    encodeResponseData(channel, out, res.getResult());
                }
            } else out.writeUTF(res.getErrorMessage());
            out.flushBuffer();
            if (out instanceof Cleanable) {
                ((Cleanable) out).cleanup();
            }
            bos.flush();
            bos.close();

            int len = bos.writtenBytes();
            checkPayload(channel, len);
            //设置响应体长度
            Bytes.int2bytes(len, header, 12);
            // write
            buffer.writerIndex(savedWriteIndex);
            buffer.writeBytes(header); // write header.
            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
        } catch (Throwable t) {
            // clear buffer
            buffer.writerIndex(savedWriteIndex);
            // send error message to Consumer, otherwise, Consumer will wait till timeout.
            if (!res.isEvent() && res.getStatus() != Response.BAD_RESPONSE) {
                Response r = new Response(res.getId(), res.getVersion());
                r.setStatus(Response.BAD_RESPONSE);

                if (t instanceof ExceedPayloadLimitException) {
                    logger.warn(t.getMessage(), t);
                    try {
                        r.setErrorMessage(t.getMessage());
                        channel.send(r);
                        return;
                    } catch (RemotingException e) {
                        logger.warn("Failed to send bad_response info back: " + t.getMessage() + ", cause: " + e.getMessage(), e);
                    }
                } else {
                    // FIXME log error message in Codec and handle in caught() of IoHanndler?
                    logger.warn("Fail to encode response: " + res + ", send bad_response info instead, cause: " + t.getMessage(), t);
                    try {
                        r.setErrorMessage("Failed to send response: " + res + ", cause: " + StringUtils.toString(t));
                        channel.send(r);
                        return;
                    } catch (RemotingException e) {
                        logger.warn("Failed to send bad_response info back: " + res + ", cause: " + e.getMessage(), e);
                    }
                }
            }

            // Rethrow exception
            if (t instanceof IOException) {
                throw (IOException) t;
            } else if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else if (t instanceof Error) {
                throw (Error) t;
            } else {
                throw new RuntimeException(t.getMessage(), t);
            }
        }
    }
    //DubboCodec
    protected void encodeResponseData(Channel channel, ObjectOutput out, Object data) throws IOException {
        Result result = (Result) data;

        Throwable th = result.getException();
        //如果有异常设置响应标识为异常，且序列化异常信息，如果无异常且无返回值，设置响应标识为无数据，有响应则设置响应标识为有数据响应，并且序列化响应数据
        if (th == null) {
            Object ret = result.getValue();
            if (ret == null) {
                out.writeByte(RESPONSE_NULL_VALUE);
            } else {
                out.writeByte(RESPONSE_VALUE);
                out.writeObject(ret);
            }
        } else {
            out.writeByte(RESPONSE_WITH_EXCEPTION);
            out.writeObject(th);
        }
    }
```
以上是请求和响应的编码流程，接下来分析解码流程，解码入口是模板方法```decode()```，源码如下：  
```
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        int readable = buffer.readableBytes();
        byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
        buffer.readBytes(header);
        return decode(channel, buffer, readable, header);
    }

    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        // 魔法数校验，如果校验不通过则调用TelnetCodec进行解码
        if (readable > 0 && header[0] != MAGIC_HIGH
                || readable > 1 && header[1] != MAGIC_LOW) {
            int length = header.length;
            if (header.length < readable) {
                header = Bytes.copyOf(header, readable);
                buffer.readBytes(header, length, readable - length);
            }
            for (int i = 1; i < header.length - 1; i++) {
                if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                    buffer.readerIndex(buffer.readerIndex() - header.length + i);
                    header = Bytes.copyOf(header, i);
                    break;
                }
            }
            return super.decode(channel, buffer, readable, header);
        }
        // 如果可读长度小于header长度，返回无数据错误
        if (readable < HEADER_LENGTH) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        //检测消息体长度，如超出允许长度则抛异常
        int len = Bytes.bytes2int(header, 12);
        checkPayload(channel, len);
        //如果可读长度小于请求体长度，返回无数据错误
        int tt = len + HEADER_LENGTH;
        if (readable < tt) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        // limit input stream.
        ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);

        try {
            return decodeBody(channel, is, header);
        } finally {
            if (is.available() > 0) {
                try {
                    if (logger.isWarnEnabled()) {
                        logger.warn("Skip input stream " + is.available());
                    }
                    StreamUtils.skipUnusedStream(is);
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        }
    }
```
消息体的解码由```DubboCodec.decodeBody()```实现，逻辑如下：  
```
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        //flag=状态标识位
        //proto=序列化标识位
        byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
        //根据序列化标识位获取序列化实现
        Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);
        // 获取请求id
        long id = Bytes.bytes2long(header, 4);
        //位运算获取请求标识，如果是1则是响应
        if ((flag & FLAG_REQUEST) == 0) {
            // decode response.
            Response res = new Response(id);
            //获取事件标识，如果是事件，就设置事件标识为心跳
            if ((flag & FLAG_EVENT) != 0) {
                res.setEvent(Response.HEARTBEAT_EVENT);
            }
            // 第三位为响应结果位，获取响应结果
            byte status = header[3];
            res.setStatus(status);
            if (status == Response.OK) {
                try {
                    Object data;
                    if (res.isHeartbeat()) {
                        //对心跳包进行解码
                        data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
                    } else if (res.isEvent()) {
                        //对事件进行解码
                        data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
                    } else {
                        DecodeableRpcResult result;
                        //在当前IO线程上进行解码
                        if (channel.getUrl().getParameter(
                                Constants.DECODE_IN_IO_THREAD_KEY,
                                Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                            result = new DecodeableRpcResult(channel, res, is,
                                    (Invocation) getRequestData(id), proto);
                            result.decode();
                        } else {
                            result = new DecodeableRpcResult(channel, res,
                                    new UnsafeByteArrayInputStream(readMessageData(is)),
                                    (Invocation) getRequestData(id), proto);
                        }
                        data = result;
                    }
                    res.setResult(data);
                } catch (Throwable t) {
                    if (log.isWarnEnabled()) {
                        log.warn("Decode response failed: " + t.getMessage(), t);
                    }
                    res.setStatus(Response.CLIENT_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
            } else {
                res.setErrorMessage(deserialize(s, channel.getUrl(), is).readUTF());
            }
            return res;
        } else {
            // 请求解码和响应解码大致一致，除了version和status方面，其他医院
            Request req = new Request(id);
            req.setVersion("2.0.0");
            req.setTwoWay((flag & FLAG_TWOWAY) != 0);
            if ((flag & FLAG_EVENT) != 0) {
                req.setEvent(Request.HEARTBEAT_EVENT);
            }
            try {
                Object data;
                if (req.isHeartbeat()) {
                    data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
                } else if (req.isEvent()) {
                    data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
                } else {
                    DecodeableRpcInvocation inv;
                    if (channel.getUrl().getParameter(
                            Constants.DECODE_IN_IO_THREAD_KEY,
                            Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                        inv = new DecodeableRpcInvocation(channel, req, is, proto);
                        inv.decode();
                    } else {
                        inv = new DecodeableRpcInvocation(channel, req,
                                new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                    }
                    data = inv;
                }
                req.setData(data);
            } catch (Throwable t) {
                if (log.isWarnEnabled()) {
                    log.warn("Decode request failed: " + t.getMessage(), t);
                }
                // bad request
                req.setBroken(true);
                req.setData(t);
            }
            return req;
        }
    }
```