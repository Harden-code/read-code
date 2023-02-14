###Spring 事物

事物的传播行为解决业务方法之前相互调用的影响

####事物的传播级别：

```java
PROPAGATION_REQUIRED:Support a current transaction; create a new one if none exists

PROPAGATION_SUPPORTS:Support a current transaction; execute non-transactionally if none exists.

PROPAGATION_MANDATORY:Support a current transaction; throw an exception if no current transaction

PROPAGATION_REQUIRES_NEW: Create a new transaction, suspending the current transaction if one exists.

PROPAGATION_NOT_SUPPORTED: Do not support a current transaction; rather(宁愿) always execute non-transactionally.

PROPAGATION_NEVER:Do not support a current transaction; throw an exception if a current transaction

PROPAGATION_NESTED:Execute within a nested transaction if a current transaction exists,behave like PROPAGATION_REQUIRED 
```

#####实现细节：

注解装配 ProxyTransactionManagementConfiguration

通过@EnableTransactionManagement的@Import导入

#####前置知识：

AbstractAutoProxyCreator的作用生成AOP代理对象，利用spring生命周期的回调接口`SmartInstantiationAwareBeanPostProcessor`，该接口`postProcessBeforeInstantiation`方法会在bean初始化之前直接生成代理对象返回

```java
//org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args){
  ..
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  ..
}
//AbstractAutoProxyCreator
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
  Object cacheKey = getCacheKey(beanClass, beanName);

  TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
  if (targetSource != null) {
    if (StringUtils.hasLength(beanName)) {
      this.targetSourcedBeans.add(beanName);
    }
    //暴露给子类实现，查找Advisors(Interceptor=增强执行动作)
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
    Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }
  return null;
}
```

里面的createProxy方法通过ProxyFactory生成代理对象。代理对象分为jdk和cglib两种方式存在，其本质逻辑都是，把Interceptor和被代理对象构建成一个MethodInvocation调用进行调用。

```java
//JdkDynamicAopProxy 继承jdk动态代理InvocationHandler接口，getProxy返回Proxy.newProxyInstance生成的对象
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		...
      // Get the interception chain for this method.
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
      if (chain.isEmpty()) {
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {    
         MethodInvocation invocation =
               new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain); 
         retVal = invocation.proceed();
      }
   ...
}
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
  if (logger.isTraceEnabled()) {
    logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
  }
  Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
  findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
  return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}

```

#####核心实现（缓存，校验都差不多）：

BeanFactoryTransactionAttributeSourceAdvisor 查找的@Transaction的PointCut

TransactionInterceptor 增强逻辑执行

org.springframework.transaction.interceptor.TransactionInterceptor#invoke(很多细节劝退)

嵌套事物的实现逻辑：

通过threadlocal存储当前事物的传播级别和其他信息+嵌套回滚通过JDBC的API

```JAVA
Savepoint setSavepoint() //记录点
void rollback(Savepoint savepoint) //回滚只回滚到setSavepoint的点
```







