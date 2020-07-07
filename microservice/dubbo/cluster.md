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
|-aa-|-aa-|
|-aa-|-aa-|