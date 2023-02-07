####解码器

byte-->消息

io.netty.handler.codec.ByteToMessageDecoder

io.netty.handler.codec.ByteToMessageDecoder#decode

消息-->消息

io.netty.handler.codec.MessageToMessageDecoder

io.netty.handler.codec.MessageToMessageDecoder#decode

####编码器

消息-->byte

io.netty.handler.codec.MessageToByteEncoder

io.netty.handler.codec.MessageToByteEncoder#encode

消息-->消息

io.netty.handler.codec.MessageToMessageEncoder

io.netty.handler.codec.MessageToMessageEncoder#encode

####内嵌编解码处理器

DelimiterBasedFrameDecoder 使用自定义分割符

LineBasedFrameDecoder 行分割符

FixedLengthFrameDecoder 固定长度

LengthFieldBasedFrameDecoder 根据编码进帧头部中的长度值提取帧，该字段的偏移量以及长度在构造函数中指定

> io.netty.channel.ChannelDuplexHandler 入站和出站适配类 extends ChannelInboundHandlerAdapter implements ChannelOutboundHandler

