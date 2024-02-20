数据库的事务,解决数据的并发读写问题.因此分类4种隔离级别来,级别划分主要是划分数据一致性和性能提升所带来的平衡.他们的区别在控制读写锁或加锁位置的不同.由于锁对并发来说都是很重的东西,所有现在的事务实现通过一种并发版本控制mvcc,避免锁带来的开销.

###MyBatis里的事务
org.apache.ibatis.transaction.jdbc.JdbcTransaction
主要就是在getConnection的时候设置自动提交属性,因此它的事务范围是connection

```
  public void openConnection() throws SQLException {
    ...
    connection = dataSource.getConnection();
    ...
    setDesiredAutoCommit(autoCommit);
  }

```


###Spring里的事务
Spring里的事务就相对会复杂一点,它主要提供了跨**方法**的事务范围,比MyBatis的粒度更细.

####前置知识

#####JDBC规范:
java.sql.Savepoint - 事务回滚点记录,Connection.setSavepoint()方法创建,
Connection.rollback()方法进行回滚

The representation of a savepoint, which is a point within
the current transaction that can be referenced from the
Connection.rollback method. When a transaction
is rolled back to a savepoint all changes made after that
savepoint are undone.

```
 // process
  Savepoint a=connection.setSavepoint();
  ...
  connection.rollback(Savepoint savepoint)
```
#####Spring动态代理:
低配模式:
把要提升的方法,比如前置,后置处理器会变成对应的Handler在最后在代理方法中进行实现
```java
Object proxy = Proxy.newProxyInstance(classLoader, new Class[]{EchoService.class}, new InvocationHandler() {
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (EchoService.class.isAssignableFrom(method.getDeclaringClass())) {
        // 前置拦截器
        BeforeInterceptor beforeInterceptor = new BeforeInterceptor() {
            @Override
            public Object before(Object proxy, Method method, Object[] args) {
                return System.currentTimeMillis();
            }
        };
        Long startTime = 0L;
        Long endTime = 0L;
        Object result = null;
        try {
            // 前置拦截
            startTime = (Long) beforeInterceptor.before(proxy, method, args);
            EchoService echoService = new DefaultEchoService();
            result = echoService.echo((String) args[0]); // 目标对象执行
            // 方法执行后置拦截器
            AfterReturnInterceptor afterReturnInterceptor = new AfterReturnInterceptor() {
                @Override
                public Object after(Object proxy, Method method, Object[] args, Object returnResult) {
                    return System.currentTimeMillis();
                }
            };
            // 执行 after
            endTime = (Long) afterReturnInterceptor.after(proxy, method, args, result);
        } catch (Exception e) {
            // 异常拦截器（处理方法执行后）
            ExceptionInterceptor interceptor = (proxy1, method1, args1, throwable) -> {
            };
        } finally {
            // finally 后置拦截器
            FinallyInterceptor interceptor = new TimeFinallyInterceptor(startTime, endTime);
            Long costTime = (Long) interceptor.finalize(proxy, method, args, result);
            System.out.println("echo 方法执行的实现：" + costTime + " ms.");
        }

    }
    return null;
}
```


直接给出结论,像Spring内置AOP注解都有回应的Interceptor实例(见org.aopalliance.intercept.MethodInterceptor),
里面就是增强逻辑实现,比如TransactionInterceptor[Transactional],MethodValidationInterceptor[Validated]
#####在不同方法之间隐式的传参
threadlocal


####Spring事务实现:

##### PlatformTransactionManager
```
//事务管理接口
public interface PlatformTransactionManager extends TransactionManager{
    // TransactionDefinition =>注解里填写的属性,返回事务信息[里面有Savepoint对象]
    // 因此跨方法的事务传播就是通过Savepoint对象实现
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;
}
```

#####低配伪代码

```java
// MethodInvocation 方法执行上下文对象
// 一个方法对应一个savepoint 很关键
public Object invoke(MethodInvocation invocation) throws Throwable {

    try{
        // 检查事务信息,通过threadlocal,创建或者生成savepoint
        return invocation.process();
    }catch(e){
        // 回滚操作
    }finnaly{
        // 清除threadlocal
    }

}

```


#####实战分析
1. TransactionInterceptor
  这个类没什么实现,主要适配动态代理接口MethodInterceptor实现invoke方法,主要的逻辑都在父类TransactionAspectSupport的
  invokeWithinTransaction方法上
```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
        ...
    	@Override
    	@Nullable
    	public Object invoke(MethodInvocation invocation) throws Throwable {
    		// Work out the target class: may be {@code null}.
    		// The TransactionAttributeSource should be passed the target class
    		// as well as the method, which may be from an interface.
    		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
    		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
    	}
    	...
}
```


2. TransactionAspectSupport
```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

       @Nullable
       	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
       			final InvocationCallback invocation) throws Throwable {
            ...
            // 根据这段话判断下面if...else...里的逻辑,if执行有事务方法,else非事方法[猜测需要调试]
       		// If the transaction attribute is null, the method is non-transactional.
       		TransactionAttributeSource tas = getTransactionAttributeSource();
       		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
       		// 事务管理器
       		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
       		// savepoint 名字
       		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

       		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
       			// Standard transaction demarcation with getTransaction and commit/rollback calls.
       			// 创建savepoint和threadlocal在这里面
       			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

       			Object retVal;
       			try {
       				// This is an around advice: Invoke the next interceptor in the chain.
       				// This will normally result in a target object being invoked.
       				retVal = invocation.proceedWithInvocation();
       			}
       			catch (Throwable ex) {
       				// target invocation exception
       				completeTransactionAfterThrowing(txInfo, ex);
       				throw ex;
       			}
       			finally {
       				cleanupTransactionInfo(txInfo);
       			}

       			if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
       				// Set rollback-only in case of Vavr failure matching our rollback rules...
       				TransactionStatus status = txInfo.getTransactionStatus();
       				if (status != null && txAttr != null) {
       					retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
       				}
       			}

       			commitTransactionAfterReturning(txInfo);
       			return retVal;
       		}
       		else {
       			...
       		}
       	}

}
```

