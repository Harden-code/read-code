## Mapper动态代理实现
1. Mapper里的接口方法怎么跟xml或注解SQL打通的

在Mybatis里主要通过JDK动态代理实现,实现入口org.apache.ibatis.binding.MapperProxy类.具体实现看invoker方法.主要逻辑也在cachedInvoke创建的org.apache.ibatis.binding.MapperMethod.MapperMethod对象里.

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      }
      // 缓存 Map<Method, MapperMethodInvoker>
      // MapperMethodInvoker可以不用管主要封装,具体逻辑MapperMethod封装
      return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
}
```
MapperMethod对象是桥接Method和SQL语句的关键,里面包含了有两个属性也是最关键的.它们两由Configuration里存放的MappedStatement对象解析构成.SqlCommand执行命令和MethodSignature方法签名

> 1. MappedStatement里面放的xml里sql,这个对象在Configuration对象里的一个Map， key全限类名+方法名字
> 2. MappedStatement对象里org.apache.ibatis.mapping.SqlSource是存放的xml里sql,有不同的实现调用getBoundSql就返回最终执行SQL
> 3. org.apache.ibatis.builder.annotation.MapperAnnotationBuilder.parseStatement注解构造MappedStatement的方式，解析好后放入Configuration对象里


```java
public class MapperMethod {
    // 存放取MappedStatement的key,和sql类型
    private final SqlCommand command;
    // 方法前面用生成参数,和返回值
    private final MethodSignature method;
    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        // 根据类型选择对应的执行方法
        // 在后面的执行sql的接口里会看到很多Object param其实就是这里生成出来传递下去的
        switch (command.getType()) {
          case INSERT: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
          }
          case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
          }
          case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
          }
          case SELECT:
            if (method.returnsVoid() && method.hasResultHandler()) {
              executeWithResultHandler(sqlSession, args);
              result = null;
            } else if (method.returnsMany()) {
              result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
              result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
              result = executeForCursor(sqlSession, args);
            } else {
              Object param = method.convertArgsToSqlCommandParam(args);
              result = sqlSession.selectOne(command.getName(), param);
              if (method.returnsOptional() && (result == null || !method.getReturnType().equals(result.getClass()))) {
                result = Optional.ofNullable(result);
              }
            }
            break;
          case FLUSH:
            result = sqlSession.flushStatements();
            break;
          default:
            throw new BindingException("Unknown execution method for: " + command.getName());
        }
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
          throw new BindingException("Mapper method '" + command.getName()
              + "' attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
  }
}
```
