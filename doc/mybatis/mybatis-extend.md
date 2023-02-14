### Mybatis 扩展接口

##### 扩展接口

- org.apache.ibatis.executor.parameter.ParameterHandler
- org.apache.ibatis.executor.resultset.ResultSetHandler
- org.apache.ibatis.executor.statement.StatementHandler



##### 拦截器扩展实现优化

org.apache.ibatis.plugin.Interceptor 该接口拦截方法(SQL)执前进行调用

通过org.apache.ibatis.plugin.Interceptor#plugin进行装配。该方法在这个接口有默认实现，通过动态代理的；也可通过重写该方法通过责任链的模式增强功能

动态代理拦截

```java
public static Object wrap(Object target, Interceptor interceptor) {
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  Class<?> type = target.getClass();
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
        type.getClassLoader(),
        interfaces,
        new Plugin(target, interceptor, signatureMap));
  }
  return target;
}

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
```