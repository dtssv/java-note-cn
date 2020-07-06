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
```RoundRobinLoadBalance```为加权轮询模式，逻辑如下:  
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
            //轮询，直到模为0，且某个invoker的权重大于0则返回对应invoker
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
            if (leastActive == -1 || active < leastActive) { // 如果发现更小的活跃数就重新开始
                leastActive = active; // 记录最小的活跃数
                leastCount = 1; // 重置leastCount
                leastIndexs[0] = i; // 重置
                totalWeight = weight; // 重置
                firstWeight = weight; // 记录第一个invoker的权重
                sameWeight = true; // 重置每一个invoker有相同的权重
            } else if (active == leastActive) { // 当前invoker的下标和最小活跃数相同
                leastIndexs[leastCount++] = i; // 记录当前invoker的下标
                totalWeight += weight; // 权重相加
                // 判断是否有相同的权重
                if (sameWeight && i > 0
                        && weight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // assert(leastCount > 0)
        if (leastCount == 1) {
            // 如果只有一个最小活跃数invoker那么直接返回
            return invokers.get(leastIndexs[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            // 如果权重不都相当，那么基于总权重选择一个invoker
            int offsetWeight = random.nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexs[i];
                offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                if (offsetWeight <= 0)
                    return invokers.get(leastIndex);
            }
        }
        // 如果所有invoker的权重相同，随机返回一个
        return invokers.get(leastIndexs[random.nextInt(leastCount)]);
    }
```
```ConsistentHashLoadBalance```为一致性hash负载均衡，为了避免节点过少导致的hash不均匀而导致某些节点负载过高，dubbo会为每个节点构建默认160个虚拟节点，逻辑如下:  
```
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int identityHashCode = System.identityHashCode(invokers);
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        //选择器，如果不存在则构建一个新的选择器
        if (selector == null || selector.identityHashCode != identityHashCode) {
            selectors.put(key, new ConsistentHashSelector<T>(invokers, invocation.getMethodName(), identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);//选择invoker
    }
    //选择器构造函数
    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            this.identityHashCode = identityHashCode;
            URL url = invokers.get(0).getUrl();
            this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);//虚拟节点数，默认160，为了避免节点过少造成hash分布不均匀，为每个节点创建160个虚拟节点
            //使用那些参数作为hashKey，默认为第0个，可以设置如0,1;0,2,4这样
            String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                for (int i = 0; i < replicaNumber / 4; i++) {
                    //对invoker+i进行md5运算，获取16位的数组
                    byte[] digest = md5(address + i);
                    for (int h = 0; h < 4; h++) {
                        // h=0时对0，1，2，3四位进行位运算
                        // h=1时对4，5，6，7四位进行位运算
                        // h=2,3时一次类推
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }

        public Invoker<T> select(Invocation invocation) {
            String key = toKey(invocation.getArguments());//构建key
            byte[] digest = md5(key);//计算key MD5
            return selectForKey(hash(digest, 0));//根据hash结果选择离得最近的后一个节点
        }

        private String toKey(Object[] args) {
            //根据argumentIndex构建key
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

        private Invoker<T> selectForKey(long hash) {
            Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();//选择离得最近的后一个节点
        	if (entry == null) {
        		entry = virtualInvokers.firstEntry();//如果为空就选择第一个
        	}
        	return entry.getValue();
        }

        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }

        private byte[] md5(String value) {
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.reset();
            byte[] bytes;
            try {
                bytes = value.getBytes("UTF-8");
            } catch (UnsupportedEncodingException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.update(bytes);
            return md5.digest();
        }
```