### Mybatis Mapper Register

#### Mapper接口注册过程

Mapper接口用于执行SQL语句相关的方法，方法名一般和Mapper XML配置文件种<select|update|delete|insert>标签的id属性相同，接口的完全限定名一般对应Mapper XML配置文件的命名空间。

MapperRegistry对象存放Mapper的容器，属性里有个knownMappers的Map集合用于存放Mapper，其中key为Class对象，value生成的代理对象工MapperProxyFactory，用来生成MapperProxy对象，JDK动态代理拦截类

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
}
```

org.apache.ibatis.binding.MapperRegistry#addMapper 

#### MappedStatement 注册过程

Configuration类中有一个mappedStatements属性，该属性用于注册MyBatis中所有的MappedStatement对象

```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
```

mappedStatements属性是一个Map对象，它的Key为Mapper SQL配置的Id，如果SQL是通过XML配置的，则Id为命名空间加上<select|update|delete|insert>标签的Id，如果SQL通过Java注解配置，则Id为Mapper接口的完全限定名（包括包名）加上方法名称。

注册逻辑方法：org.apache.ibatis.builder.xml.XMLStatementBuilder#parseStatementNode

#### Mapper调用

Mybatis中MapperProxy实现InvocationHandler接口，用于实现动态相关逻辑，所以直接查看invoke方法

```java
//MapperProxy中创建MapperMethod 执行方法
private MapperMethod cachedMapperMethod(Method method) {
  MapperMethod mapperMethod = methodCache.get(method);
  if (mapperMethod == null) {
    mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
    methodCache.put(method, mapperMethod);
  }
  return mapperMethod;
}
//最终执行方法
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
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
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

MapperMethod：

```JAVA
public class MapperMethod {
	..
  private final SqlCommand command;
  private final MethodSignature method;
    ..
}
```

SqlCommand:

```java
public static class SqlCommand {

  private final String name;
  //执行类型 updata、select，delete
  private final SqlCommandType type;
  }
```

