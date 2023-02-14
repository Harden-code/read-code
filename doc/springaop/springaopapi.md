



#### Advisor，PointCut， Advice关系

Advisor -> PointCut + Advice

1.Advice 是一个通知的行为具体要做什么动作

2.PointCut 增强点的判断（判断对象或者方法是否匹配）ClassFilters，MethodMatchers

3.Advisor 为两个对象组合构成，即是一个完成的通知

接口实现类：

PointCut->StaticMethodMatcherPointcut 方法上里的切入点
Advice->MethodInterceptor 方法拦截器
Advisor->DefaultPointcutAdvisor





### Advice

- JoinPoint ->MethodInvocation 方法级别的接入点，方法调用点抽象
- Around Advice -Interceptor
  - 方法拦截器 - MethodInterceptor
  - 构造器拦截器 - ConstructorInterceptor
  - 前置动作
    - BeforeAdvice
    - MethodBeforeAdvice
  - 后置动作
    - AfterAdvice
    - AfterReturningAdvice
    - ThrowsAdvice



### PointCut

Pointcut->判断当前方法,是否匹配（适合符合动态代理条件 -- 切点的判断）

有两个判断接口抽象一个是类型判断匹配ClassFilters，一个是方法判断匹配MethodMatcher

ComposablePointcut是一个组合Pointcut

- ComposablePointcut组合判断
  - ClassFilter工具类 ClassFilters
  - MethodMatcher工具类 MethodMatchers
  - PointCut工具类 PointCuts
- 静态方法PointCut - StaticMethodMatcherPointCut
- 正则表达式PointCut - JdkRegexpMethodPointcut
- 控制流PointCut - ControlFlowPointcut
- AspectJ表达式 - AspectJExpressionPointCut
  - PointcutExpression

Api使用demo

```java
//两个自定义Pointcut
EchoServicePointcut echoServicePointcut = new EchoServicePointcut("echo", EchoService.class);
EchoServiceEchoMethodPointcut echoServiceEchoMethodPointcut = new EchoServiceEchoMethodPointcut();
ComposablePointcut pointcut = new ComposablePointcut();
// 组合实现
pointcut.intersection(echoServicePointcut.getClassFilter());
pointcut.intersection(echoServicePointcut.getMethodMatcher());
pointcut.intersection(echoServiceEchoMethodPointcut);
// 将 Pointcut 适配成 Advisor
DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new EchoServiceMethodInterceptor());
DefaultEchoService defaultEchoService = new DefaultEchoService();
ProxyFactory proxyFactory = new ProxyFactory(defaultEchoService);
// 添加 Advisor
proxyFactory.addAdvisor(advisor);
// 获取代理对象
EchoService echoService = (EchoService) proxyFactory.getProxy();
System.out.println(echoService.echo("Hello,World"));
```



### Advice容器接口-Advisor

- 接口 -Advisor
  - 实现 DefaultPointCutAdvisor



### PointCut与Advice连接器 - PointCutAdvisor

- 接口 - PointCutAdvisor
  - 通用实现
    - DefaultPointCutAdvisor
  - AspectJ实现
    - AspectJExpressionPointCutAdvisor
    - AspectJPointCutAdvisor
  - 静态方法实现
    - StaticMethodMatcherPointCutAdvisor
  - Ioc容器实现
    - AbstractBeanFactoryPointCutAdvisor   



### Introduction与Advice连接器

- 接口 IntroductionAdvisor
  - 元信息 IntroductionInfo
  - 通用实现 DefaultIntroductionAdvisor
  - AspectJ实现 DeclareParentsAdvisor

IntroductionAdvisor 与 Advisor的区别 ：

```
Advisor=PointCut(ClassFilter+MethodMatcher) +  Advice(执行动作)
IntroductionAdvisor=IntroductionInfo(Class)+Advice
```



### Advisor的Interceptor适配器

- AdvisorAdapter
  - MethodBeforeAdvice - MethodBeforeAdviceAdapter
  - AfterReturningAdvice - AfterReturningAdviceAdapter
  - ThrowsAdvice - ThrowsAdviceAdapter



### JointPoint Before Advice

- 接口 
  - 接口：BeforeAdvice
  - 方法级别：MethodBeforeAdvice
- 实现
  - MethodBeforeAdviceInterceptor
  - AspectJ : AspectJMethodBeforeAdvice

### JointPoint After Advice

- 接口
  - AfterAdvice
  - AfterReturningAdvice
  - ThrowAdvice
- 实现
  - ThrowsAdviceInterceptor
  - AfterReturningAdviceInterceptor
  - AspectJ :
    - AspectJAfterAdvice
    - AspectJAfterRetuningAdvice
    - AspectJAfterThrowingAdvice



### AOP代理接口 - AopProxy

- JDK动态代理：JdkDynamicAopProxy

- CGLIB字节码提升：CglibAopProxy  

  ```
  ObjenesisCglibAopProxy
  ```

- AopProxy工厂接口与实现

  - 接口 AopProxyFactory
  - 默认实现 DefaultAopProxyFactory
  - 返回类型 
    - JdkDynamicAopProxy
    - CglibAopProxy
    - ObjenesisCglibAopProxy

### AopProxyFactory配置管理器

- AdvisedSupport  (增加Advice,Advisor)

  - 基类 ProxyConfig
  - 接口 Advised实现
  - 接口 AopProxy实现   

- AdvisedSupport 子类实现
  ProxyCreateSupport 

  ```
  语义 - 代理对象创建基类
  ```

### AdvisorChainFactory

```
- 特殊实现 InterceptorAndDynamicMethodMatcher  //匹配转换
- DefaultAdvisorFactory
```



### 目标对象来源接口实现

```
- TargetSource
    - 实现 ：HotSwappableTargetSource
            AbstractPoolingTargetSource
            ProtopypeTargetSource
            ThreadLocalTargetSource
            SingletonTargetSource
```



### AdvisedSupport事件监听器

- AdvisedSupportListener

  - 事件对象：AdvisedSupport
  - 事件来源：ProxyCreatorSupport
  - 激活事件触发：ProxyCreateSupport#createAopProxy
  - 变更事件触发：代理接口变化，Advisor变化，配置复制

  ​

### AdvisedSupport事件监听器

- AdvisedSupportListener

  - AdvisedSupport
  - ProxyCreatorSupport
  - ProxyCreatorSupport#createAopProxy
  - 变更事件触发,代理接口发生变化时,Advisor变化时,配置复制

  ​

### ProxyCreatorSupport IOC容器实现

- 核心API ProxyFactoryBean

  - 主要基类 ProxyFactorySupport

  ​

#### IOC容器自动代理抽象 - AbstractAutoProxyCreator

- 特点
  - 与Spring Bean声明周期整合 SmartInstantiationAwareBeanPostProcessor
- IoC整合
  - BeanClassLoaderAware
  - BeanFactoryAware
- 特性增强 FactoryBean



### AspectJ实现

- AspectJProxyFactory

  - 基类 ProxyCreatorSupport

  - 特点 AspectJ注解整合

  - 相关 API 

    - AspectMetadata
    - AspectJAdvisorFac

    ​

### 创建Proxy代理工程

ProxyCreatorSupport
  ProxyFactory
  AspectJProxyFactory
  ProxyFactoryBean



### 自动动态代理

 BeanNameAutoProxyCreator
 DefaultAdvisorAutoProxyCreator
 InfrastructureAdvisorAutoProxyCreator ---BeanDef里role 有关

- AspectJ容器自动代理实现

 基类：AspectJAwareAdvisorAutoProxyCreator

 AnnotationAwareAspectJAutoProxyCreator



### Infrastructure Bean接口 Spring Aop基础设施

 排除非Spring基础设施的bean，(bean is part of Spring's AOP infrastructure)

- AopInfrastructureBean
  - aop基础Bean标记接口
  - 实现
    - AbstractAutoProxyCreator
    - ScopedProxyFactoryBean
  - 判断逻辑
    - AbstractAutoProxyCreator#isInfrastuctureClass
    - ConfigurationClassUtils#checkConfigurationClassCandidate



### AOP上下文辅助类

- AopContext：ThreadLocal扩展，临时存储AOP对象

- AopProxyUtils

  - getSingletonTarget 从实例中获取单例对象
  - ultimateTargetClass 从实例中获取最终目标类
  - completeProxiedInterfaces 计算AdvisedSupport配置中所有被代理的接口
  - proxiedUserInterfaces 从代理对象中获取代理接口 

- AopUtils

  - isAopProxy 判断对象是否代理
  - isJdkDynamicProxy 判断对象是否jdk动态代理对象
  - isCglibProxy 判断对象是否是Cglib
  - getTargetClass 从对象中获取目标类型
  - invokeJoinPointUsingReflection 使用java反射调用JoinPoint目标方法

  ​

### AspectJ Enable模块驱动

- 注解 - EnableAspectJAutoProxy
- 属性方法 
  - proxyTargetClass 是否已类型代理
  - exposeProxy 是否将代理暴露在AopContext中
- 设计模型@Enable
  - 实现 ImportBeanDefinitionRegistrar -> AspectJAutoProxyRegistrar
- 底层实现 AnnotationAwareAspectJAutoProxyCreator 



### AspectJ XML 配置驱动实现

- XML元素 <aop:aspectj-autoproxy/>
- 属性
  - proxyTargetClass 是否已类型代理
  - exposeProxy 是否将代理暴露在AopContext中 
- 底层实现
  - AspectJAutoProxyBeanDefinitionParser

### AOP配置

- XML <aop:config/>
- 属性
  - proxyTargetClass 是否已类型代理
  - exposeProxy 是否将代理暴露在AopContext中
- 嵌套元素
  - pointcut
  - advisor
  - aspect
- 底层实现 
  - ConfigBeanDefinitionParser