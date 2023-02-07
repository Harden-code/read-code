NioServerSocketChannel处理新建连接绑定逻辑：

1. ServerBootstrap初始化逻辑

   ```java
   //io.netty.bootstrap.ServerBootstrap#doBind
   private ChannelFuture doBind(final SocketAddress localAddress) {
      		//初始化+注册
           final ChannelFuture regFuture = initAndRegister();
           final Channel channel = regFuture.channel();
           if (regFuture.cause() != null) {
               return regFuture;
           }
           //不能肯定register完成，因为register是丢到nio event loop里面执行去了。
           if (regFuture.isDone()) {
               // At this point we know that the registration was complete and successful.
               ChannelPromise promise = channel.newPromise();
               doBind0(regFuture, channel, localAddress, promise);
               return promise;
           } else {
               // Registration future is almost always fulfilled already, but just in case it's not.
               final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
               //等着register完成来通知再执行bind
               regFuture.addListener(new ChannelFutureListener() {
                   @Override
                   public void operationComplete(ChannelFuture future) throws Exception {
                       Throwable cause = future.cause();
                       if (cause != null) {
                           promise.setFailure(cause);
                       } else {                
                           promise.registered();
                           doBind0(regFuture, channel, localAddress, promise);
                       }
                   }
               });
               return promise;
           }
       }
   #bind 具体实现在Unsafe=>io.netty.channel.AbstractChannel.AbstractUnsafe#bind
   ```

2. 向eventLoop注册channel，会从eventLoopGroup里找出一个eventLoop进行注册

   ```java
   //io.netty.bootstrap.ServerBootstrap#initAndRegister
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

3. 注册逻辑包含：1.会创建一个异步注册的句柄；2.unsafe()返回的是包装原生nio操作的对象（AbstractUnsafe）绑定channel的执行器，让；3.register0注册channe感兴趣事件

   `selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);`

   > 需要注意地方：SelectionKey register(Selector sel, int ops, Object att)，att存放的是Channel对象，当在循环事件select放回后，可以通过返回值取到这个出入的Channel.
   >
   > ```java
   > Set<SelectionKey> selectedKeys=selector.selectedKeys()；
   > Iterator<SelectionKey> i = selectedKeys.iterator();
   > while(i.hasNext()){
   > 	SelectionKey key=i.next();
   >   //selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
   >   final Object a = k.attachment();
   >   //移除selection key : 上次调用next（）结果
   >   i.remove();
   >   ...
   > }
   > ```

   ```java
   //io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)
   @Override
   public ChannelFuture register(Channel channel) {
     return register(new DefaultChannelPromise(channel, this));
   }
   //重载方法
   @Override
   public ChannelFuture register(final ChannelPromise promise) {
     ObjectUtil.checkNotNull(promise, "promise");
     promise.channel().unsafe().register(this, promise);
     return promise;
   }
   //io.netty.channel.AbstractChannel.AbstractUnsafe#register
   @Override
   public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    	...
     AbstractChannel.this.eventLoop = eventLoop;//绑定channel的执行器
      //判断是否是当前线程是否是eventLoop执行器上的线程
     if (eventLoop.inEventLoop()) {
       //selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
       register0(promise);
     } else {
       try {
         eventLoop.execute(new Runnable() {
           @Override
           public void run() {
             register0(promise);
           }
         });
       } catch (Throwable t) {
         ...
         closeForcibly();
         closeFuture.setClosed();
         safeSetFailure(promise, t);
       }
     }
   }
   ```

4. 在ServerBootstrap创建NioServerSocketChannel后，初始化时会添加一个ChannelHandler到pipeline链表中，该handler就是处理客服端创建的新连接，并注册在eventLoop中。

   ```JAVA
   //io.netty.bootstrap.ServerBootstrap#init
   void init(Channel channel) {
     ...
       ChannelPipeline p = channel.pipeline();
     ...
       //ChannelInitializer一次性、初始化handler:
       //负责添加一个ServerBootstrapAcceptor handler，添加完后，自己就移除了:
       //ServerBootstrapAcceptor handler： 负责接收客户端连接创建连接后，对连接的初始化工作。
       p.addLast(new ChannelInitializer<Channel>() {
         @Override
         public void initChannel(final Channel ch) {
           final ChannelPipeline pipeline = ch.pipeline();
           ChannelHandler handler = config.handler();
           if (handler != null) {
             pipeline.addLast(handler);
           }
           //添加pipeline中
           //eventLoop开始轮询
           ch.eventLoop().execute(new Runnable() {
             @Override
             public void run() {
               pipeline.addLast(new ServerBootstrapAcceptor(
                 ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
             }
           });
         }
       });
   }
   ```

5. ServerBootstrapAcceptor#channelRead触发逻辑触发：在io.netty.channel.nio.NioEventLoop#run会一直循环select等待事件的到来，通过attachment获取Channel对象进行读写操作也就是。读写操作通过事件的方式进行传播

   ```JAVA
   //有简化的
   protected void run() {
     for (;;) {
         ...
           Iterator<SelectionKey> i = selector.selectedKeys().iterator();
           for (;;) {
               final SelectionKey k = i.next();
               //AbstractNioChannel 对象里有原生Channel
               //selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
               final Object a = k.attachment();
               //移除selection key : 上次调用next（）结果
               i.remove();
               if (a instanceof AbstractNioChannel) {
                   processSelectedKey(k, (AbstractNioChannel) a);
               } 
               if (!i.hasNext()) {
                   break;
               }
           }
         ...      
   }
   ```

6. 通过NioMessageUnsafe创建Channel，NioServerSocketChannel通过实现doReadMessages抽象完成完成

   ```
    protected int doReadMessages(List<Object> buf) throws Exception {
           //接受新连接创建SocketChannel
           SocketChannel ch = SocketUtils.accept(javaChannel());
           try {
               if (ch != null) {
                   buf.add(new NioSocketChannel(this, ch));
                   return 1;
               }
           } catch (Throwable t) {
               logger.warn("Failed to create a new channel from an accepted socket.", t);
               try {
                   ch.close();
               } catch (Throwable t2) {
                   logger.warn("Failed to close a socket.", t2);
               }
           }
           return 0;
       }
   ```

   ​


