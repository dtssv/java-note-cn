dubbo负载均衡由```LoadBalance```实现，这个类由SPI注解修饰，默认实现方式为```RandomLoadBalance```，这里对各个实现类分别分析一下。  
首先是```RandomLoadBalance```，顾名思义也就是随机选取一台机器，这是dubbo默认的负载均衡算法，实现逻辑如下：  
```
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // invoker数量
        int totalWeight = 0; // 总权重
        boolean sameWeight = true; // invoker权重是否一样
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            totalWeight += weight; // Sum
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // 在总权重中间随机取一个整数
            int offset = random.nextInt(totalWeight);
            // 挨个减去每个invoker的权重，知道offset<0则返回对应invoker
            for (int i = 0; i < length; i++) {
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // 如果权重都一样那就随便取一个
        return invokers.get(random.nextInt(length));
    }
```
权重计算逻辑：  
```
    protected int getWeight(Invoker<?> invoker, Invocation invocation) {
        //获取权重值，如果没设置则默认为100
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
        if (weight > 0) {
            //获取服务提供者启动时间戳
            long timestamp = invoker.getUrl().getParameter(Constants.REMOTE_TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
                int uptime = (int) (System.currentTimeMillis() - timestamp);
                //获取服务预热时间
                int warmup = invoker.getUrl().getParameter(Constants.WARMUP_KEY, Constants.DEFAULT_WARMUP);
                if (uptime > 0 && uptime < warmup) {
                    //如果服务启动时间小于预热时间则重新计算权重
                    weight = calculateWarmupWeight(uptime, warmup, weight);
                }
            }
        }
        return weight;
    }

    static int calculateWarmupWeight(int uptime, int warmup, int weight) {
        //启动时间/(预热时间/原始权重)取整
        int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
        //如果重新计算后的权重小于1则取1，否则和原始权重比较哪个小使用哪儿作为新的权重
        return ww < 1 ? 1 : (ww > weight ? weight : ww);
    }
```
```RoundRobinLoadBalance```为轮询模式，逻辑如下:  
```
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int length = invokers.size(); // Number of invokers
        int maxWeight = 0; // 最大权重值
        int minWeight = Integer.MAX_VALUE; // 最小权重值
        final LinkedHashMap<Invoker<T>, IntegerWrapper> invokerToWeightMap = new LinkedHashMap<Invoker<T>, IntegerWrapper>();
        int weightSum = 0;//总权重值
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            maxWeight = Math.max(maxWeight, weight); // 获取最大权重值
            minWeight = Math.min(minWeight, weight); // 获取最小权重值
            if (weight > 0) {
                invokerToWeightMap.put(invokers.get(i), new IntegerWrapper(weight));
                weightSum += weight;
            }
        }
        //获取方法的调用数量，没有就放一个新的，里边是一个AtomicInteger()
        AtomicPositiveInteger sequence = sequences.get(key);
        if (sequence == null) {
            sequences.putIfAbsent(key, new AtomicPositiveInteger());
            sequence = sequences.get(key);
        }
        int currentSequence = sequence.getAndIncrement();
        if (maxWeight > 0 && minWeight < maxWeight) {
            //当前调用次数和总权重取模
            int mod = currentSequence % weightSum;
            //轮询，直到模为0，或者某个invoker的权重为0则返回对应invoker
            for (int i = 0; i < maxWeight; i++) {
                for (Map.Entry<Invoker<T>, IntegerWrapper> each : invokerToWeightMap.entrySet()) {
                    final Invoker<T> k = each.getKey();
                    final IntegerWrapper v = each.getValue();
                    if (mod == 0 && v.getValue() > 0) {
                        return k;
                    }
                    if (v.getValue() > 0) {
                        v.decrement();
                        mod--;
                    }
                }
            }
        }
        // 如果总权重为0，则轮询取模
        return invokers.get(currentSequence % length);
    }
```
```LeastActiveLoadBalance```为最不活跃负载均衡算法，逻辑如下:  
```
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // invoker数量
        int leastActive = -1; // 最小的活跃数
        int leastCount = 0; // 有相同最小活跃数的invoker数量
        int[] leastIndexs = new int[length]; // 有相同最小活跃数的invoker下标
        int totalWeight = 0; // 总权重
        int firstWeight = 0; // 初始值，用来比较
        boolean sameWeight = true; // 是否所有invoker的权重都一样
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive(); // 获取活跃数，初次进来为0
            int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT); // 权重
            if (leastActive == -1 || active < leastActive) { // Restart, when find a invoker having smaller least active value.
                leastActive = active; // Record the current least active value
                leastCount = 1; // Reset leastCount, count again based on current leastCount
                leastIndexs[0] = i; // Reset
                totalWeight = weight; // Reset
                firstWeight = weight; // Record the weight the first invoker
                sameWeight = true; // Reset, every invoker has the same weight value?
            } else if (active == leastActive) { // If current invoker's active value equals with leaseActive, then accumulating.
                leastIndexs[leastCount++] = i; // Record index number of this invoker
                totalWeight += weight; // Add this invoker's weight to totalWeight.
                // If every invoker has the same weight?
                if (sameWeight && i > 0
                        && weight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // assert(leastCount > 0)
        if (leastCount == 1) {
            // If we got exactly one invoker having the least active value, return this invoker directly.
            return invokers.get(leastIndexs[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offsetWeight = random.nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexs[i];
                offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                if (offsetWeight <= 0)
                    return invokers.get(leastIndex);
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(leastIndexs[random.nextInt(leastCount)]);
    }
```
```ConsistentHashLoadBalance```为一致性hash负载均衡，逻辑如下:  
```
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int identityHashCode = System.identityHashCode(invokers);
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        if (selector == null || selector.identityHashCode != identityHashCode) {
            selectors.put(key, new ConsistentHashSelector<T>(invokers, invocation.getMethodName(), identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);
    }
```