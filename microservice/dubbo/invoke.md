### client发起请求
由服务引入可知，dubbo引入的类都是代理类，代理类内容大致如：  
```
package org.apache.dubbo.common.bytecode;

public class proxy0 implements ClassGenerator.DC,EchoService,XxxService {

    public static java.lang.reflect.Method[] methods;

    private java.lang.reflect.InvocationHandler handler;

    public proxy0() {
    }

    public proxy0(java.lang.reflect.InvocationHandler arg0) {
        handler = arg0;
    }

    public Object $echo(Object arg0){
        Object[] args = new Object[1];
        args[0] = (Object) arg0;
        Object ret = handler.invoke(this, methods[0], args);
        return (java.lang.String) ret;
    }

    public String demo(String arg0){
        Object[] args = new Object[1];
        args[0] = (Object) arg0;
        Object ret = handler.invoke(this, methods[1], args);
        return (java.lang.String) ret;
    }
    //...其他方法省略，逻辑如此
}

public class Proxy0 implement Proxy{
    public Proxy0(){

    }
    public Object newInstance(InvocationHandler h){
        return new com.alibaba.dubbo.common.bytecode.proxy0(h); 
    }
}
```
注入的```InvocationHandler```实现类为```InvokerInvocationHandler```，最终获取```XxxService```的代理类```proxy0```，所以我们调用```XxxService.demo```时实际上调用的是```proxy0.demo```，此处举例列一下请求的调用链后边具体分析每一步的操作：  
```
proxy0#demo(String)
  —> InvokerInvocationHandler#invoke(Object, Method, Object[])
    —> MockClusterInvoker#invoke(Invocation)
      —> AbstractClusterInvoker#invoke(Invocation)
        —> FailoverClusterInvoker#doInvoke(Invocation, List<Invoker<T>>, LoadBalance)
          —> Filter#invoke(Invoker, Invocation)  // 包含多个 Filter 调用
            —> ListenerInvokerWrapper#invoke(Invocation) 
              —> AbstractInvoker#invoke(Invocation) 
                —> DubboInvoker#doInvoke(Invocation)
                  —> ReferenceCountExchangeClient#request(Object, int)
                    —> HeaderExchangeClient#request(Object, int)
                      —> HeaderExchangeChannel#request(Object, int)
                        —> AbstractPeer#send(Object)
                          —> AbstractClient#send(Object, boolean)
                            —> NettyChannel#send(Object, boolean)
                              —> NioClientSocketChannel#write(Object)
```
以次分析请求调用过程如下：  
首先是```InvokerInvocationHandler```的invoke：  
```
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        //如果方法是toSting,hashCode,equals，则直接调用
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        //此处invoker=MockClusterInvoker
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
}
```
```MockClusterInvoker```是在集群环境下默认的invoker，如无特殊指定，会在它内部调用```FailoverClusterInvoker```：  
```
    //MockClusterInvoker
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            //不走mock
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            //直接调用mock
            result = doMockInvoke(invocation, null);
        } else {
            //服务调用失败再调用mock
            result = this.invoker.invoke(invocation);
            //....
        }
        return result;
    }
```
```FailoverClusterInvoker```的具体实现见[服务集群](cluster.md)，在此不过多关注，直接去看具体的invoker调用，也就是从```AbstractInvoker```开始：  
```
    public Result invoke(Invocation inv) throws RpcException {
        //...
        RpcInvocation invocation = (RpcInvocation) inv;
        invocation.setInvoker(this);
        if (attachment != null && attachment.size() > 0) {
            //设置attachment
            invocation.addAttachmentsIfAbsent(attachment);
        }
        Map<String, String> context = RpcContext.getContext().getAttachments();
        if (context != null) {
            //设置上下文中的contextAttachment
            invocation.addAttachmentsIfAbsent(context);
        }
        if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) {
            //设置异步调用
            invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
        }
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        try {
            //调用具体实现类方法
            return doInvoke(invocation);
        } catch (Exception e) { // biz exception
           //...异常处理
        }
    }
```
一般情况下我们调用的都是```DubboInvoker```的实现方法，逻辑如下：  
```
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);//是否异步调用
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);//是否单向通道
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
                //如果是单向通道调用，则不需要响应结果，直接返回
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                //如果是异步调用，设置future并返回结果
                ResponseFuture future = currentClient.request(inv, timeout);
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
                //同步调用
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (Exception e) {
            //异常处理
        }
    }
```
然后如果client是共享的话就是一路```ReferenceCountExchangeClient.request()```->```HeaderExchangeClient.request()```->```HeaderExchangeChannel->request()```，前两个方法只是对后边方法的调用，无其他逻辑处理，所以我们只看```HeaderExchangeChannel->request()```，逻辑如下：  
```
   public ResponseFuture request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // 构造请求
        Request req = new Request();
        req.setVersion("2.0.0");
        req.setTwoWay(true);
        req.setData(request);
        DefaultFuture future = new DefaultFuture(channel, req, timeout);
        try {
            channel.send(req);//此处channel为NettyClient
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```
```NettyClient```并未实现```send()```方法，而是由父类```AbstractPeer```实现，此处调用```AbstractClient.send()```，逻辑如下：  
```
    public void send(Object message, boolean sent) throws RemotingException {
        if (send_reconnect && !isConnected()) {
            connect();
        }
        Channel channel = getChannel();//获取一个channel，由NettyClient实现
        //...判断channel状态
        channel.send(message, sent);
    }
```
具体的请求发送由```NetyChannel.send()```实现，逻辑如下：  
```
    public void send(Object message, boolean sent) throws RemotingException {
        super.send(message, sent);

        boolean success = true;
        int timeout = 0;
        try {
            //由Netty的channel发送请求并返回一个Future
            ChannelFuture future = channel.write(message);
            if (sent) {
                timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
                success = future.await(timeout);
            }
            Throwable cause = future.getCause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
        }

        if (!success) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                    + "in timeout(" + timeout + "ms) limit");
        }
    }
```
到此，消费者发送请求逻辑就分析完毕。  
### server处理请求
