服务集群，即是```Cluster```，在服务引入时有如下代码：  
```
    if (urls.size() == 1) {
        invoker = refprotocol.refer(interfaceClass, urls.get(0));
    } else {
        List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
        URL registryURL = null;
        for (URL url : urls) {
            invokers.add(refprotocol.refer(interfaceClass, url));
            if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                registryURL = url; // use last registry url
            }
        }
        if (registryURL != null) { // registry url is available
            // use AvailableCluster only when register's cluster is available
            URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
            invoker = cluster.join(new StaticDirectory(u, invokers));
        } else { // not a registry url
            invoker = cluster.join(new StaticDirectory(invokers));
        }
    }
```
如果服务列表的数量大于1，那么就会将服务构建为cluster invoker，dubbo提供了多种方式保证集群容错，实现和功能入下表：  
|容错方式|实现类|功能描述|使用场景|
|----|----|----|----|
|failover|FailoverClusterInvoker|失败自动切换|     |
|failfast|FailfastClusterInvoker|快速失败|幂等|
|failsafe|FailsafeClusterInvoker|安全失败|记录日志等|
|failback|FailbackClusterInvoker|失败自动修复|消息通知|
|forking|ForkingClusterInvoker|并行调用多个服务提供者|读请求|
|available|AvailableClusterInvoker|调用第一个可用的服务提供者|    |
|mergeable|MergeableClusterInvoker|   |    |
|broadcast|BroadcastClusterInvoker|广播|通知所有服务更新信息|
从上到下以次分析如何实现。  
```FailoverClusterInvoker```（默认）:  
```
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyinvokers = invokers;
        checkInvokers(copyinvokers, invocation);
        //获取借口重试次数，默认是3次
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //...
            //失败后重新选择一个invoker再次发起调用，直到达到重试次数
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                Result result = invoker.invoke(invocation);
                //...
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        //...
    }
```
```FailfastClusterInvoker```:  
```
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
            //选择一个invoker，失败则抛出异常
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            //.....
        }
    }
```
```FailsafeClusterInvoker```:  
```
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            //选择一个invoker，失败则忽略异常
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            return new RpcResult(); // ignore
        }
    }
```
```FailbackClusterInvoker```:   
```
    private void addFailed(Invocation invocation, AbstractClusterInvoker<?> router) {
        if (retryFuture == null) {
            synchronized (this) {
                if (retryFuture == null) {
                    retryFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {
                        //启动一个线程重试失败请求
                        public void run() {
                            // collect retry statistics
                            try {
                                retryFailed();
                            } catch (Throwable t) { // Defensive fault tolerance
                                logger.error("Unexpected error occur at collect statistic", t);
                            }
                        }
                    }, RETRY_FAILED_PERIOD, RETRY_FAILED_PERIOD, TimeUnit.MILLISECONDS);
                }
            }
        }
        failed.put(invocation, router);
    }

    void retryFailed() {
        if (failed.size() == 0) {
            return;
        }
        for (Map.Entry<Invocation, AbstractClusterInvoker<?>> entry : new HashMap<Invocation, AbstractClusterInvoker<?>>(
                failed).entrySet()) {
            Invocation invocation = entry.getKey();
            Invoker<?> invoker = entry.getValue();
            try {
                invoker.invoke(invocation);
                failed.remove(invocation);
            } catch (Throwable e) {
                logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
            }
        }
    }

    @Override
    protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            //调用失败将请求放入失败map
            addFailed(invocation, this);
            return new RpcResult(); // ignore
        }
    }
```
```ForkingClusterInvoker```:   
```
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        final List<Invoker<T>> selected;
        final int forks = getUrl().getParameter(Constants.FORKS_KEY, Constants.DEFAULT_FORKS);
        final int timeout = getUrl().getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        //获取选中的invoker列表
        if (forks <= 0 || forks >= invokers.size()) {
            selected = invokers;
        } else {
            selected = new ArrayList<Invoker<T>>();
            for (int i = 0; i < forks; i++) {
                // TODO. Add some comment here, refer chinese version for more details.
                Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                if (!selected.contains(invoker)) {//Avoid add the same invoker several times.
                    selected.add(invoker);
                }
            }
        }
        RpcContext.getContext().setInvokers((List) selected);
        final AtomicInteger count = new AtomicInteger();
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<Object>();
        //线程池启动多个线程同时调用所有服务
        for (final Invoker<T> invoker : selected) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Result result = invoker.invoke(invocation);
                        ref.offer(result);//将结果放到阻塞队列
                    } catch (Throwable e) {
                        int value = count.incrementAndGet();
                        //只有所有请求都失败才把异常放到队列里，否则有一个成功即认为请求成功
                        if (value >= selected.size()) {
                            ref.offer(e);
                        }
                    }
                }
            });
        }
        try {
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);//获取结果
            if (ret instanceof Throwable) {
                Throwable e = (Throwable) ret;
                throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
            }
            return (Result) ret;
        } catch (InterruptedException e) {
            throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
        }
    }
```
```AvailableClusterInvoker```:  
```
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {//调用第一个可用的服务
                return invoker.invoke(invocation);
            }
        }
        throw new RpcException("No provider available in " + invokers);
    }
```
```BroadcastClusterInvoker```:  
```
    //轮询调用所有服务，如果有一个失败则抛出异常，都成功则认为成功
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        RpcContext.getContext().setInvokers((List) invokers);
        RpcException exception = null;
        Result result = null;
        for (Invoker<T> invoker : invokers) {
            try {
                result = invoker.invoke(invocation);
            } catch (RpcException e) {
                exception = e;
                logger.warn(e.getMessage(), e);
            } catch (Throwable e) {
                exception = new RpcException(e.getMessage(), e);
                logger.warn(e.getMessage(), e);
            }
        }
        if (exception != null) {
            throw exception;
        }
        return result;
    }
```
以上即为常用cluster集群配置，```MergeableClusterInvoker```此处暂不分析。  