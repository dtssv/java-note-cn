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

        // 从第4位开始设置请求id，占8字节
        Bytes.long2bytes(req.getId(), header, 4);

        // encode request data.
        int savedWriteIndex = buffer.writerIndex();
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        if (req.isEvent()) {
            encodeEventData(channel, out, req.getData());
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