Channel，回调，Future，事件和ChannelHandler

#### 核心类

##### EventLoop

> Will handle all the I/O operations for a {@link Channel} once registered.

实现类：NioEventLoop

NioEventLoop类的层次结构：

```JAVA
AbstractScheduledEventExecutor
SingleThreadEventExecutor
SingleThreadEventLoop
NioEventLoop
```

关键属性：

```java
NioEventLoop{
  //AbstractScheduledEventExecutor 循环任务队列
  PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue;
  //SingleThreadEventExecutor 执行状态
  private static final int ST_NOT_STARTED = 1;
  private static final int ST_STARTED = 2;
  private static final int ST_SHUTTING_DOWN = 3;
  private static final int ST_SHUTDOWN = 4;
  private static final int ST_TERMINATED = 5;
  //任务队列
  private final Queue<Runnable> taskQueue;
  //执行器 线程对象 它有个方法包装		io.netty.util.internal.ThreadExecutorMap#apply(java.lang.Runnable, io.netty.util.concurrent.EventExecutor) 设置当前执行器的线程
  private final Executor executor;

  @Override
  public void execute(Runnable task) {
    if (task == null) {
      throw new NullPointerException("task");
    }
    //当前线程跟执行器线程是否一样
    //初始化标记 doStartThread方法进行设置
    //thread == this.thread
    boolean inEventLoop = inEventLoop();
    //
    addTask(task);
    //未初始化执行
    if (!inEventLoop) {
      //NioEventLoop#run方法（循环select）
      startThread();
      if (isShutdown()) {
        ...
      }
    }
    if (!addTaskWakesUp && wakesUpForTask(task)) {
      wakeup(inEventLoop);
    }
  }
  //startThread 启动
  private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
      @Override
      public void run() {
 //executor 线程通过ThreadFactory（ThreadPerTaskExecutor）创建的
        //NioEventLoopGroup#newChild方法
        thread = Thread.currentThread();
        if (interrupted) {
          thread.interrupt();
        }
        updateLastExecutionTime();
        ...
          SingleThreadEventExecutor.this.run();
        ...
          cleanup();//关闭select
      }
    }
}                                       
```

EventLoop，不仅是一个IO操作的handle，而且它还有线程的任务调度功能。里面有个执行线程一直循环处理任务io.netty.channel.nio.NioEventLoop#run逻辑。调用execute方法实则是把任务加到队列里面，然后执行线程执行。



##### EventLoopGroup

EventLoopGroup管理EventLoop

关键属性：

```JAVA
//管理的EventLoop
private final EventExecutor[] children;
//选择器，配合next方法调用
private final EventExecutorChooserFactory.EventExecutorChooser chooser;
//初始化EventLoop方法
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
  EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
  return new NioEventLoop(this, executor, (SelectorProvider) args[0],
                          ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
@Override
public EventExecutor next() {
  return chooser.next();
}
```



##### Channel 

> A nexus to a network socket or a component which is capable of I/O
> operations such as read, write, connect, and bind.

生命周期：unregistered，registered，active，inactive（没有连接远程节点）

实现类：NioServerSocketChannel，NioSocketChannel

NioServerSocketChannel类的层次结构：

```
AbstractChannel
AbstractNioChannel
//NioByteUnsafe读取字节流
AbstractNioByteChannel
NioSocketChannel
```

NioSocketChannel类的层次结构：

```
AbstractChannel
AbstractNioChannel
//NioMessageUnsafe读取操作信息
AbstractNioMessageChannel
NioServerSocketChannel
```

NioServerSocketChannel和NioSocketChannel区别：

NioServerSocketChannel代表原生nio中ServerSocketChannel，NioSocketChannel代表原生nio中的SocketChannel。

ServerSocketChannel通过`java.nio.channels.ServerSocketChannel#accept`方法监听socket当有新的一个连接时，会创建一个SocketChannel。SocketChannel则代表一个连接可在上面进行读写操作，反之不行。因此它们具体关注AbstractNioMessageChannel#NioMessageUnsafe和AbstractNioByteChannel#NioByteUnsafe的read方法上

NioSocketChannel关键属性：

```JAVA
NioSocketChannel{
  //原生操作对象
  private final Unsafe unsafe;
  //原生channel
  private final SelectableChannel ch;
    /**
     * The future of the current connection attempt.  If not null, subsequent
     * connection attempts will fail.
     */
  private ChannelPromise connectPromise;
  //IO handler 
  //初始化注册EventLoopGroup#register(Channel) 
  //NioServerSocketChannel和NioSocketChannel绑定或者连接触发
  //  public ChannelFuture register(final ChannelPromise promise) {
  //      ObjectUtil.checkNotNull(promise, "promise");
  //      promise.channel().unsafe().register(this, promise);
  //     return promise;
  // }
  //NioSocketChannel#connect
  //ServerBootstrap#bind 
  //ServerBootstrapAcceptor#channelRead方法
  private volatile EventLoop eventLoop;
} 
```

EventLoop与Channel绑定：

Channel=>连接（操作实体），EventLoop=>处理IO事件的线程对象（事件循环处理器）

当一个新的Channel生成时，就会调用EventLoop#register让Channel与EventLoop进行绑定（Bootstrap配置的EventLoopGroup对象），也就是在Channel设置EventLoop，绑定后EventLoop与Channel的关系将不会被改变；

> Channel注册时机：
>
> 1.NioSocketChannel#connect ；2.ServerBootstrap#bind ；3.ServerBootstrapAcceptor#channelRead



##### Unsafe 

封装一系列原生操作nio的方法

> Unsafe operations that should <em>never</em> be called from user-code. These methods
> are only provided to implement the actual transport, and must be invoked from an I/O thread except for the
> following methods:
>
> {@link #localAddress()}，{@link #remoteAddress()}
> {@link #closeForcibly()}，{@link #register(EventLoop, ChannelPromise)}，{@link #deregister(ChannelPromise)}
> ，{@link #voidPromise()}



##### ChannelFuture

>  All I/O operations in Netty are asynchronous.  It means any I/O calls will return immediately with no guarantee that the requested I/O operation has been completed at the end of the call.  Instead, you will be returned with a {@link ChannelFuture} instance which gives you the information about the result or status of the I/O operation.

Java中的Future模式：

可以让任务异步执行，当future调用get获取执行结果，如果任务没有执行完毕将会阻塞等到执行完毕才返回。

比如FutureTask

```java
public class FutureTask<V> implements RunnableFuture<V> {...}
```

Netty在此基础上增加了addListener使其避免调用get阻塞方法。ChannelFuture实则提供了一种在操作完成时通知应用程序的方式，这个对象看作是一个异步操作的占位符，比如：

io.netty.bootstrap.AbstractBootstrap#initAndRegister

regFuture就代表一个异步注册事件的占位符

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
 		..
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    //开始register
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

AbstractChannel的closeFuture表示等待连接关闭的占位符

```java
//AbstractChannel的closeFuture，
private final CloseFuture closeFuture = new CloseFuture(this);
//等待连接关闭
ChannelFuture f = b.connect(HOST, PORT).sync();
// Wait until the connection is closed.
f.channel().closeFuture().sync();
```



##### ChannelHandler 

> Handles an I/O event or intercepts an I/O operation, and forwards it to its next handler in its

主要充当所有处理入站出战数据的handler，ChannelInboundHandler，ChannelOutboundHandler；

自定义入站操作时，可以继承io.netty.channel.SimpleChannelInboundHandler，该类会帮忙关闭缓存

```JAVA
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
  boolean release = true;
  try {
    if (acceptInboundMessage(msg)) {
      @SuppressWarnings("unchecked")
      I imsg = (I) msg;
      channelRead0(ctx, imsg);
    } else {
      release = false;
      ctx.fireChannelRead(msg);
    }
  } finally {
    if (autoRelease && release) {
      ReferenceCountUtil.release(msg);
    }
  }
}
```

操作事件方法：

ChannelInboundHandler，ChannelOutboundHandler分别增加了入站和出战的事件操作

```JAVA
void handlerAdded(ChannelHandlerContext ctx) throws Exception;
void handlerRemoved(ChannelHandlerContext ctx) throws Exception;
@Deprecated
void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
```



##### ChannelPipeline

> A list of {@link ChannelHandler}s which handles or intercepts inbound events and outbound operations of a {@link Channel}.  {@link ChannelPipeline} implements an advanced form of the Intercepting Filter pattern to give a user full control over how an event is handled and how the {@link ChannelHandler}s in a pipeline* interact with each other.

```JAVA
public class DefaultChannelPipeline implements ChannelPipeline {
	 final AbstractChannelHandlerContext head;
    final AbstractChannelHandlerContext tail;

    private final Channel channel;
}
```

是ChannelHandler的一个容器。每一个新的channel都会分配ChannelPipeline。通过调用ChannelHandlerContext实现，转发到ChannelPipeline的下一个ChannelHandler。

需要注意：ChannelPipeline每个ChannelHandler同时通过eventLoop来处理传递事件的，所以不要阻塞该线程



##### ChannelHandlerContext

> Enables a {@link ChannelHandler} to interact with its {@link ChannelPipeline}
> and other handlers. Among other things a handler can notify the next {@link ChannelHandler} in the
> {@link ChannelPipeline} as well as modify the {@link ChannelPipeline} it belongs to dynamically.

```JAVA
final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext{\
    volatile AbstractChannelHandlerContext next;
    volatile AbstractChannelHandlerContext prev;
    private final ChannelHandler handler;
}
```

每当有ChannelHandler添加到ChannelPipeline都会创建一个ChannelHandlerContext对象

注意:ChannelHandlerContext和ChannelPipeline触发事件的区别，ChannelHandlerContext调用事件只会触发它的下个handler，而ChannelPipeline调用触发事件是从头开始触发

```JAVA
//ChannelPipeline
public final ChannelPipeline fireChannelReadComplete() {
    AbstractChannelHandlerContext.invokeChannelReadComplete(head);
    return this;
}
//ChannelHandlerContext
static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
  EventExecutor executor = next.executor();
  if (executor.inEventLoop()) {
    next.invokeChannelRegistered();
  } else {
    executor.execute(new Runnable() {
      @Override
      public void run() {
        next.invokeChannelRegistered();
      }
    });
  }
}
```





##### ByteBufAllocator

内存分配器



##### ByteBufHolder

提供了访问底层数据和引用计数的方法



