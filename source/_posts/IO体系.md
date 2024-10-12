---
title: IO体系
date: 2024-09-20 14:06:23
tags: [ "Java基础", "IO", "NIO" ]
---

> 基于Java 17 整理

# 知识体系

**同步IO**
![](同步IO.png)
异步IO

<!-- more -->

## NIO概述

Java NIO（New IO，即非阻塞IO）是从Java 1.4版本开始引入的一套新的IO API，可以替代标准的Java IO
API。NIO支持面向缓冲区的（Buffer-oriented）、基于通道的（Channel-based）IO操作。NIO提供了更高的性能，特别是在需要高速读写、大量并发访问IO操作时。

## 核心类解析

### Buffer

在NIO中，所有数据都是用缓冲区处理的。缓冲区实质上是一个数组。常用的缓冲区有`ByteBuffer`、`CharBuffer`、`IntBuffer`等。

### Channel

通道是双向的，既可以用来进行读操作，也可以用来进行写操作。常见的通道有`FileChannel`、`DatagramChannel`、`SocketChannel`
和`ServerSocketChannel`等。

### Selector

选择器用于使用单个线程处理多个 Channel。如果你的应用打开了多个连接（通道），但每个连接的流量都很低，使用选择器就会非常方便。

## 重点源码说明

- `ByteBuffer`的`flip()`方法：切换读写模式。
- `Channel`的`read()`和`write()`方法：从通道读取数据和向通道写入数据。
- `Selector`的`select()`方法：检查一个或多个通道是否在你注册的事件上有所更新。

## 使用说明

### ByteBuffer使用示例

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.put((byte) 97); // 写入数据
        buffer.flip(); // 切换模式

byte b = buffer.get(); // 读取数据
```

### FileChannel使用示例

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buf); // 读取数据到Buffer
```

### Selector使用示例

```java
Selector selector = Selector.open();
channel.

configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
while(selector.

select() >0){
Set<SelectionKey> selectedKeys = selector.selectedKeys();
Iterator<SelectionKey> it = selectedKeys.iterator();
    while(it.

hasNext()){
SelectionKey myKey = it.next();
// 处理IO事件
    }
            }
```

**实现一个NettyServer**

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(workerGroup instanceof EpollEventLoopGroup ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
                    .childOption(ChannelOption.TCP_NODELAY, true)
                    .childOption(ChannelOption.SO_REUSEADDR, true)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline()
                                    .addLast("decoder", new HttpRequestDecoder())
                                    .addLast("encoder", new HttpResponseEncoder())
                                    .addLast("handler", new NettyHttpServerHandler(httpHandler, httpServer));
                        }
                    }).option(ChannelOption.SO_BACKLOG, 128);

            ChannelFuture f = b.bind(8080).sync();
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```