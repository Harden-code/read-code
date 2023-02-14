#### Java监控指标的负载均衡

##### 通用的监控要素：

指标数据：JVM Memory（Heap Non-Heap)，操作系统（CPU，网络，内存）,容器（CPU 内存）

指标来源：JMX，JNI，FIle System

指标收集工具：Jolokia - JMX HTTP Brige，Netflix Servo，Micrometer，Spring Metrics，Promethues，Pinpoint，Skywalking，Zipkins

指标存储：ElasticSearch，Promethues Server，Open TSDB，InfluxDB，Redis，HBase，MySQL



##### 监控指标获取方式

通过JMX获取，相关接口信息

- java.lang.management.OperatingSystemMXBean
- java.lang.management.MemoryPoolMXBean
- java.lang.management.MemoryManagerMXBean
- java.lang.management.GarbageCollectorMXBean
- java.lang.management.ClassLoadingMXBean
- java.lang.management.MemoryMXBean
- java.lang.management.ThreadMXBean
- java.lang.management.RuntimeMXBean
- java.lang.management.CompilationMXBean

Hotspot JVM扩展JMX接口

- com.sun.management.OperatingSystemMXBean


- - java.lang.management.OperatingSystemMXBean


- com.sun.management.ThreadMXBean


- - java.lang.management.ThreadMXBean


- com.sun.management.GarbageCollectorMXBean


- - java.lang.management.GarbageCollectorMXBean


- com.sun.management.GarbageCollectorMXBean


- - java.lang.management.GarbageCollectorMXBean

> 获取操作系统指标的接口com.sun.management.OperatingSystemMXBean
>
> CPU利用率方法：com.sun.management.OperatingSystemMXBean#getProcessCpuLoad（Returns the "recent cpu usage" for the Java Virtual Machine process）



##### 传统负载均衡算法不足

Spring Cloud的负载均算法，比如 RoundRobinLoadBalance 通过注册中心的服务个数进行调用次数取余选择，未能考虑新节点预热情况，因此未预热的节点选择后，由于处于预热阶段所执行的效率肯定会低于其他已预热节点。这就会导致伪负载。解决方案：通过各个服务的启动时间或者系统指标进行动态的权重划分

比如 Dubbo的动态负载均衡算法

org.apache.dubbo.rpc.cluster.loadbalance.AbstractLoadBalance#getWeight

```JAVA
@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
    ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
    int totalWeight = 0;
    long maxCurrent = Long.MIN_VALUE;
    long now = System.currentTimeMillis();
    Invoker<T> selectedInvoker = null;
    WeightedRoundRobin selectedWRR = null;
    for (Invoker<T> invoker : invokers) {
      String identifyString = invoker.getUrl().toIdentityString();
      /**
       * org.apache.dubbo.rpc.cluster.loadbalance.AbstractLoadBalance#getWeight
       *
       * 计算权重核心逻辑
       * 运行时间/预热时间/权重
       * 预热10分钟:invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
       * 权重100:url.getMethodParameter(invocation.getMethodName(), WEIGHT_KEY,DEFAULT_WEIGHT);
       *
       * static int calculateWarmupWeight(int uptime, int warmup, int weight) {
       * 	int ww = (int) ( uptime / ((float) warmup / weight));
       *	return ww < 1 ? 1 : (Math.min(ww, weight));
       * }
       *
       */
      int weight = getWeight(invoker, invocation);
      WeightedRoundRobin weightedRoundRobin = map.computeIfAbsent(identifyString, k -> {
        WeightedRoundRobin wrr = new WeightedRoundRobin();
        wrr.setWeight(weight);
        return wrr;
      });

      if (weight != weightedRoundRobin.getWeight()) {
        //weight changed cur也会修改
        weightedRoundRobin.setWeight(weight);
      }
      //没次加
      long cur = weightedRoundRobin.increaseCurrent();
      weightedRoundRobin.setLastUpdate(now);
      //选取权重最大的
      if (cur > maxCurrent) {
        maxCurrent = cur;
        selectedInvoker = invoker;
        selectedWRR = weightedRoundRobin;
      }
      totalWeight += weight;
    }
    if (invokers.size() != map.size()) {
      map.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
    }
    if (selectedInvoker != null) {
      //减total 这样就成最小的
      selectedWRR.sel(totalWeight);
      return selectedInvoker;
    }
    // should not happen here
    return invokers.get(0);
    }
```



##### 在Spring Cloud中实现

在注册中心和客户端发生心跳时，向本地实例添加元信息（CPU 利用率  ）发送至注册中心。然后其他客户端同理在心跳时拉服务的元信息来作为动态路由的依据

- ​

##### 通过Netflix Eureka实现

基本特征：全量订阅 周期性轮训获取应用信息（registryFetchIntervalSeconds，默认 30 秒）



API：<https://github.com/Netflix/eureka/wiki/Eureka-REST-operations>

Netflix Eureka Server 原生 REST URI 前缀 ： /eureka/v2/apps/

Spring Cloud Eureka Server  REST URI 前缀 ：/eureka/apps/



Eureka 应用元信息：

```XML
<application>
    <name>BIZ-WEB</name>
    <instance>
        <instanceId>windows10.microdone.cn:biz-web:8080</instanceId>
        <hostName>windows10.microdone.cn</hostName>
        <app>BIZ-WEB</app>
        <ipAddr>192.168.0.107</ipAddr>
        <status>UP</status>
        <overriddenstatus>UNKNOWN</overriddenstatus>
        <port enabled="true">8080</port>
        <securePort enabled="false">443</securePort>
        <countryId>1</countryId>
        <dataCenterInfo class="com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo">
            <name>MyOwn</name>
        </dataCenterInfo>
        <leaseInfo>
            <renewalIntervalInSecs>30</renewalIntervalInSecs>
            <durationInSecs>90</durationInSecs>
            <registrationTimestamp>1667400440368</registrationTimestamp>
            <lastRenewalTimestamp>1667400890602</lastRenewalTimestamp>
            <evictionTimestamp>0</evictionTimestamp>
            <serviceUpTimestamp>1667400439844</serviceUpTimestamp>
        </leaseInfo>
        <metadata>
            <management.port>8080</management.port>
        </metadata>
        <homePageUrl>http://windows10.microdone.cn:8080/</homePageUrl>
        <statusPageUrl>http://windows10.microdone.cn:8080/actuator/info</statusPageUrl>
        <healthCheckUrl>http://windows10.microdone.cn:8080/actuator/health</healthCheckUrl>
        <vipAddress>biz-web</vipAddress>
        <secureVipAddress>biz-web</secureVipAddress>
        <isCoordinatingDiscoveryServer>false</isCoordinatingDiscoveryServer>
        <lastUpdatedTimestamp>1667400440368</lastUpdatedTimestamp>
        <lastDirtyTimestamp>1667400439792</lastDirtyTimestamp>
        <actionType>ADDED</actionType>
    </instance>
</application>
```

##### 自定义实现接口

客户端实现接口

- org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer 

注册服务端实现（上次元信息到注册中心）

- com.netflix.discovery.EurekaClient
- com.netflix.appinfo.ApplicationInfoManager 通过EurekaClient获取

```JAVA
 public void upload() {
        InstanceInfo instanceInfo = applicationInfoManager.getInfo();
        Map<String, String> metadata = instanceInfo.getMetadata();
        metadata.put("timestamp", String.valueOf(System.currentTimeMillis()));
        metadata.put("cpu-usage", String.valueOf(getCpuUsage()));
       //com.netflix.discovery.InstanceInfoReplicator#run 定时同步注册中心
        instanceInfo.setIsDirty();
        logger.info("Upload Eureka InstanceInfo's metadata");
    }
```

