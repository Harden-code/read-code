### Spring Cloud 客户端负载均衡

#### Spring Cloud 负载均衡的模块

org.springframework.cloud.client.loadbalancer.LoadBalancerClient是Spring Cloud抽象出来的一个共有接口来实现客户端的的负载均衡。

在RestTemplet 通过把这个LoadBalancerClient接口放在ClientHttpRequestInterceptor过滤器中进行功能实现；

而在feign中它的客户端中进行实现是实现org.springframework.cloud.openfeign.loadbalancer.FeignBlockingLoadBalancerClient

```JAVA
@Override
public <T> ServiceInstance choose(String serviceId, Request<T> request) {
  ReactiveLoadBalancer<ServiceInstance> loadBalancer = loadBalancerClientFactory.getInstance(serviceId);
  if (loadBalancer == null) {
    return null;
  }
  Response<ServiceInstance> loadBalancerResponse = 
    //扩展接口ReactorLoadBalancer
    Mono.from(loadBalancer.choose(request)).block();
  if (loadBalancerResponse == null) {
    return null;
  }
  return loadBalancerResponse.getServer();
}
```

扩展自定义实现负载均衡： 可以直接实现org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer即可，便直接会在resttemplete和feign客户端中生效



LoadBalancerClientFactory 自动配置类 - org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration；

```JAVA
private final ObjectProvider<List<LoadBalancerClientSpecification>> configurations;

public LoadBalancerAutoConfiguration(ObjectProvider<List<LoadBalancerClientSpecification>> configurations) {
this.configurations = configurations;
}

//在@LoadBalancerClient注解上有个派生注解,会把该注解上的配置类生成一个LoadBalancerClientSpecification的bean对象。LoadBalancerClientFactory是装配不同客户端的配置类
@ConditionalOnMissingBean
@Bean
public LoadBalancerClientFactory loadBalancerClientFactory(LoadBalancerClientsProperties properties) {
LoadBalancerClientFactory clientFactory = new LoadBalancerClientFactory(properties);
clientFactory.setConfigurations(this.configurations.getIfAvailable(Collections::emptyList));
return clientFactory;
}

```



LoadBalancerClient自动配置类org.springframework.cloud.loadbalancer.config.BlockingLoadBalancerClientAutoConfiguration

默认是实现负载均衡器BlockingLoadBalancerClient

```
@Bean
@ConditionalOnBean(LoadBalancerClientFactory.class)
@ConditionalOnMissingBean
public LoadBalancerClient blockingLoadBalancerClient(LoadBalancerClientFactory loadBalancerClientFactory) {
return new BlockingLoadBalancerClient(loadBalancerClientFactory);
}
```



##### RestTemplet

实则是通过实现 org.springframework.http.client.ClientHttpRequestInterceptor 拦截器实现负载逻辑。

org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor。

```JAVA
private LoadBalancerClient loadBalancer;

@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
                                    final ClientHttpRequestExecution execution) throws IOException {
  final URI originalUri = request.getURI();
  String serviceName = originalUri.getHost();
  Assert.state(serviceName != null,
               "Request URI does not contain a valid hostname: " + originalUri);
  return this.loadBalancer.execute(serviceName,
                                   this.requestFactory.createRequest(request, body, execution));
}
```



##### feign

默认配置feign的配置类 org.springframework.cloud.openfeign.loadbalancer.DefaultFeignLoadBalancerConfiguration

```JAVA
@Bean
@ConditionalOnMissingBean
@Conditional(OnRetryNotEnabledCondition.class)
public Client feignClient(LoadBalancerClient loadBalancerClient,
                          LoadBalancerClientFactory loadBalancerClientFactory) {
  return new FeignBlockingLoadBalancerClient(new Client.Default(null, null), loadBalancerClient,
                                             loadBalancerClientFactory);
}
```

org.springframework.cloud.openfeign.loadbalancer.FeignBlockingLoadBalancerClient

```JAVA
@Override
public Response execute(Request request, Request.Options options) throws IOException {
  ...
  ServiceInstance instance = loadBalancerClient.choose(serviceId, lbRequest);
loadBalancerClientFactory.getProperties(serviceId);
  ...
  return executeWithLoadBalancerLifecycleProcessing(delegate, options, newRequest, lbRequest, lbResponse,supportedLifecycleProcessors,loadBalancerProperties.isUseRawStatusCodeInResponseData());
}
```



#### Ribbon

##### Ribbon整合RestTemplet

org.springframework.cloud.netflix.ribbon.RibbonClientHttpRequestFactory

##### Ribbon整合Feign

org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient 继承 feign.Client 包装Ribbon进去

##### Ribbon整合OpenFeign 

org.springframework.cloud.client.loadbalancer.LoadBalancerClient 通过spring cloud comment 抽象的LoadBalancerClient进行整合

org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient



Ribbon 自身有超时设置
2Feign Client 也有超时设置（使用 Ribbon ReadTimeout 和 ConnectTimeout）
aFeign Client 适配 HTTP Client 实现