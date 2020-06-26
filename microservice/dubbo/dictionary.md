#### 服务字典
本来服务字典相关逻辑分析是计划在服务引入之后进行，但是在服务引入分析过程中，发现服务字典和负载均衡是一个绕不过的坎，所以就先在服务引入分析到一半的情况下，改为先解读服务字典相关，然后再接着之前的继续往下走。  
服务字典```Directory```有两个实现类，```RegistryDirectory```和```StaticDirectory```，从字面意思可以理解为前者是动态服务字典，后者为静态服务字典，大部分情况下我们使用的都是```RegistryDirectory```，所以我们着重分析，而```StaticDirectory```我们就简单带过。  
首先我们先简单分析```StaticDirectory```：  
```
public class StaticDirectory<T> extends AbstractDirectory<T> {

    private final List<Invoker<T>> invokers;

    public StaticDirectory(List<Invoker<T>> invokers) {
        this(null, invokers, null);
    }

    public StaticDirectory(List<Invoker<T>> invokers, List<Router> routers) {
        this(null, invokers, routers);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers) {
        this(url, invokers, null);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers, List<Router> routers) {
        super(url == null && invokers != null && !invokers.isEmpty() ? invokers.get(0).getUrl() : url, routers);
        if (invokers == null || invokers.isEmpty())
            throw new IllegalArgumentException("invokers == null");
        this.invokers = invokers;
    }

    @Override
    public Class<T> getInterface() {
        return invokers.get(0).getInterface();//获取invoker interface
    }

    //判断是否可用，如果有一个invoker可用则字典可用
    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                return true;
            }
        }
        return false;
    }
    //销毁字典
    @Override
    public void destroy() {
        if (isDestroyed()) {
            return;
        }
        super.destroy();
        for (Invoker<T> invoker : invokers) {
            invoker.destroy();
        }
        invokers.clear();
    }

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {

        return invokers;
    }

}
```
逻辑相对简单，无深入分析的必要性，下面我们着重开始分析```RegistryDirectory```，首先我们先看一下这个类的定义：
```
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener
```
```RegistryDirectory```初始化完成并且赋值之后首先调用了```notify()```方法，即是```NotifyListener```接口的实现类如下：  
```
    @Override
    public synchronized void notify(List<URL> urls) {
        List<URL> invokerUrls = new ArrayList<URL>();
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            //大多情况下未配置，走默认为providers
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category)
                    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                //也就是此处providers
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // configurators
        if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
            this.configurators = toConfigurators(configuratorUrls);
        }
        // routers
        if (routerUrls != null && !routerUrls.isEmpty()) {
            List<Router> routers = toRouters(routerUrls);
            if (routers != null) { // null - do nothing
                setRouters(routers);
            }
        }
        List<Configurator> localConfigurators = this.configurators; // local reference
        // merge override parameters
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && !localConfigurators.isEmpty()) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }
        // providers
        refreshInvoker(invokerUrls);
    }
```