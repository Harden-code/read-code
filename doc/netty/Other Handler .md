####额外handler实现

在Netty中字节流的转换处理都是通过handler的事件进行扩展，在源代码库netty-handler模块中也有很多handler扩展。

常用的：

- io.netty.handler.timeout.IdleStateHandler
- io.netty.handler.logging.LoggingHandler
- io.netty.handler.ssl.SslHandler

####利用IdleStateHandler实现keepAlive功能

```java
public class MyChannelInitializer extends ChannelInitializer<Channel> {
  @Override
  public void initChannel(Channel channel) {
    channel.pipeline().addLast("idleStateHandler", new {@link IdleStateHandler}(60, 30, 0));
    channel.pipeline().addLast("myHandler", new MyHandler());
  }
}
// Handler should handle the {@link IdleStateEvent} triggered by {@link IdleStateHandler}.
//IdleStateHandler超时后会发出IdleStateEvent事件，所以需要用户自己实现一个处理IdleStateEvent的handler
public class MyHandler extends ChannelDuplexHandler {
  @Override
  public void userEventTriggered( ChannelHandlerContext ctx, Object evt) throws Exception {
    if (evt instanceof { IdleStateEvent}) {
      IdleStateEvente = (IdleStateEvent) evt;
      if (e.state() == IdleState.READER_IDLE) {
        ctx.close();
      } else if (e.state() ==  IdleState.WRITER_IDLE) {
        ctx.writeAndFlush(new PingMessage());
      }
    }
  }
```

原理：

因为netty都是通过事件的方式来进行处理，而事件的处理正是handler，所以在分析代码时应该注意hander事件处理逻辑。

```java
public class IdleStateHandler{
  //设置超时事件
  public IdleStateHandler(boolean observeOutput,
                          long readerIdleTime, long writerIdleTime, long allIdleTime,
                          TimeUnit unit) {...}
  //
  @Override
  public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    // channelActive() event has been fired already, which means this.channelActive() will
    if (ctx.channel().isActive() && ctx.channel().isRegistered()) {
      //初始化
      initialize(ctx);
    } else {
    }
  }
  private void initialize(ChannelHandlerContext ctx) {
    //初始化个定时任务
    ...
    if (readerIdleTimeNanos > 0) {
      readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                                   readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    ...
  }
//任务逻辑 如果超时就发出IdleStateHandler事件
  protected void run(ChannelHandlerContext ctx) {
    long nextDelay = readerIdleTimeNanos;
    //reading channelRead打开 channelReadComplete关闭
    if (!reading) {
      //计算是否idle的关键
      //现在时间-上次读的时间
      nextDelay -= ticksInNanos() - lastReadTime;
    }

    if (nextDelay <= 0) {
      //空闲了
      // Reader is idle - set a new timeout and notify the callback.
      readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

      boolean first = firstReaderIdleEvent;
      //firstReaderIdleEvent下个读来之前，第一次idle之后，可能触发多次，都属于非第一次idle.
      firstReaderIdleEvent = false;

      try {
        IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
        channelIdle(ctx, event);
      } catch (Throwable t) {
        ctx.fireExceptionCaught(t);
      }
    } else {
      //重新其一个监测task，用nextdelay时间
      // Read occurred before the timeout - set a new timeout with shorter delay.
      readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
  }
}
}

```

