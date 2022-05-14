+++ 
date = 2022-05-14T22:01:17+08:00
title = "java网络编程101"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

# 摘要

通过hello world HTTP server示例，对比go、java io、java nio、netty网络编程，简单入个门

# Hello World

本节通过hello world http server代码示例对比四者的开发效率、性能

## 代码实现

```go
// Go
package main

import (
	"fmt"
	"net/http"
)

func main(){
	http.ListenAndServe(":8080", http.HandlerFunc(hello))
}

func hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "hello, world\n")
}
```

```jsx
// java io
public class IOHelloWorldServer {

    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
        server.createContext("/javaio", new MyHandler());
        server.setExecutor(null); // creates a default executor
        server.start();
    }

    static class MyHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange t) throws IOException {
            String response = "hello world";
            t.sendResponseHeaders(200, response.length());
            OutputStream os = t.getResponseBody();
            os.write(response.getBytes());
            os.close();
        }
    }
}
```

```jsx
// java nio

public class NioHelloWorldServer {
    private static ServerSocketChannel serverChannel;
    private static Selector selector;

    public static void main(String[] args) throws Exception {

        selector = Selector.open();
        serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.socket().bind(new InetSocketAddress(InetAddress.getByName("localhost"), 8080));
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("Server is now listening on port: 8080");

        while (true) {
            int readyNum = selector.select();
            if (readyNum == 0) {
                continue;
            }

            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                keyIterator.remove();
                try {
                    if (!key.isValid()) {
                        continue;
                    }

                    if (key.isAcceptable()) {
                        accept();
                    } else if (key.isReadable()) {
                        read(key);
                    } else if (key.isWritable()) {
                        write(key);
                    }
                } catch (Exception e) {
                    System.out.printf("Error occurred during handling key %s. Closing connection\n", e.getMessage());
                }
            }
        }
    }

    private static void accept() throws IOException {
        SocketChannel clientChannel = serverChannel.accept();
        if (clientChannel == null) {
            System.out.printf("No connection is available. Skipping selection key\n");
            return;
        }

        clientChannel.configureBlocking(false);
        clientChannel.register(selector, SelectionKey.OP_READ);
    }

    private static void read(SelectionKey key) throws IOException {
        // switch to write mode
        key.interestOps(SelectionKey.OP_WRITE);
    }

    private static void write(SelectionKey key) throws IOException {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.wrap("HTTP/1.1 200 OK\r\nConnection: close\r\n\r\nhello world\n".getBytes());
        clientChannel.write(buffer);
        clientChannel.close();
    }
}
```

```java
// netty

// file HelloWorldServer
public final class HelloWorldServer {
    public static void main(String[] args) throws Exception {
        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.option(ChannelOption.SO_BACKLOG, 1024);
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new HttpHelloWorldServerInitializer());

            Channel ch = b.bind(8080).sync().channel();

            System.err.println("Open your web browser and navigate to http://localhost:8080");

            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

//file HttpHelloWorldServerHandler
public class HttpHelloWorldServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    private static final byte[] CONTENT = { 'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd' };

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
        if (msg instanceof HttpRequest) {
            HttpRequest req = (HttpRequest) msg;

            boolean keepAlive = HttpUtil.isKeepAlive(req);
            FullHttpResponse response = new DefaultFullHttpResponse(req.protocolVersion(), OK,
                    Unpooled.wrappedBuffer(CONTENT));
            response.headers()
                    .set(CONTENT_TYPE, TEXT_PLAIN)
                    .setInt(CONTENT_LENGTH, response.content().readableBytes());

            if (keepAlive) {
                if (!req.protocolVersion().isKeepAliveDefault()) {
                    response.headers().set(CONNECTION, KEEP_ALIVE);
                }
            } else {
                // Tell the client we're going to close the connection.
                response.headers().set(CONNECTION, CLOSE);
            }

            ChannelFuture f = ctx.write(response);

            if (!keepAlive) {
                f.addListener(ChannelFutureListener.CLOSE);
            }
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}

// file HttpHelloWorldServerInitializer
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpServerExpectContinueHandler());
        p.addLast(new HttpHelloWorldServerHandler());
    }
}
```

## 开发效率

- Go版本无疑是最简洁的，核心原因在于Go `net/http` 包封装了大量的细节，让程序员的开发心智极大降低，如果看 `net/http` 包的实现，可以看到大量HTTP协议细节，以及各类优化
- 而在Java世界中，笔者能找到最接近Go `net/http` 包封装的就是 `com.sun.net.httpserver.HttpServer` 包，近似达到Go中 `net/http` 包的效果，也非常简洁；与Go版本差距在于依旧需要关心一些 executor、OutputStream等概念
- 对比之下 `java nio` 的实现可谓是细节满满，拉回到了socket编程，抽象层级和其它三者而言完全不在一个层级之上
- `netty` 的实现则引入了一些新的概念以补足 `java nio` 的缺陷，隐藏起一些底层实现的细节；相比于 `java nio` 的实现而言简单易懂很多，但也引入了 channel、channelFuture 等新的概念

# 性能

- 性能测试工具 https://github.com/rakyll/hey
- 本文都是未经过优化的代码，全靠各自语言runtime以及包的默认配置&优化，结果仅做参考，欢迎评论
- 本文中java nio实现性能有很大的提高空间，侧面反映出java nio the right way真的挺难

## 简化对比表格

|  |  QPS | P99 Latency |
| --- | --- | --- |
| Go | 51044.4546 | 99% in 0.0025 secs |
| java io | 26102.2234 | 99% in 0.0039 secs |
| java nio | 5379.3392 | 99% in 0.0153 secs |
| netty | 44739.2500 | 99% in 0.0020 secs |

## 性能对比详细数据

```jsx
hey -n 100000 http://localhost:8080/go

Summary:
  Total:        1.9591 secs
  Slowest:      0.0369 secs
  Fastest:      0.0001 secs
  Average:      0.0010 secs
  Requests/sec: 51044.4546

  Total data:   1300000 bytes
  Size/request: 13 bytes

Response time histogram:
  0.000 [1]     |
  0.004 [99612] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.007 [251]   |
  0.011 [77]    |
  0.015 [20]    |
  0.019 [11]    |
  0.022 [12]    |
  0.026 [4]     |
  0.030 [2]     |
  0.033 [5]     |
  0.037 [5]     |

Latency distribution:
  10% in 0.0006 secs
  25% in 0.0008 secs
  50% in 0.0009 secs
  75% in 0.0010 secs
  90% in 0.0012 secs
  95% in 0.0015 secs
  99% in 0.0025 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0001 secs, 0.0369 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0028 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0314 secs
  resp wait:    0.0009 secs, 0.0001 secs, 0.0337 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0357 secs

Status code distribution:
  [200] 100000 responses
```

```jsx
hey -n 100000 http://localhost:8080/javaio

Summary:
  Total:        3.8311 secs
  Slowest:      0.0307 secs
  Fastest:      0.0001 secs
  Average:      0.0019 secs
  Requests/sec: 26102.2234

  Total data:   1100000 bytes
  Size/request: 11 bytes

Response time histogram:
  0.000 [1]     |
  0.003 [96205] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.006 [3664]  |■■
  0.009 [90]    |
  0.012 [15]    |
  0.015 [3]     |
  0.018 [4]     |
  0.022 [4]     |
  0.025 [9]     |
  0.028 [2]     |
  0.031 [3]     |

Latency distribution:
  10% in 0.0011 secs
  25% in 0.0015 secs
  50% in 0.0019 secs
  75% in 0.0022 secs
  90% in 0.0027 secs
  95% in 0.0030 secs
  99% in 0.0039 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0001 secs, 0.0307 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0021 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0007 secs
  resp wait:    0.0015 secs, 0.0001 secs, 0.0251 secs
  resp read:    0.0004 secs, 0.0000 secs, 0.0022 secs

Status code distribution:
  [200] 100000 responses
```

```jsx
hey -n 100000 http://localhost:8080/javanio

Summary:
  Total:        18.5896 secs
  Slowest:      0.1030 secs
  Fastest:      0.0007 secs
  Average:      0.0093 secs
  Requests/sec: 5379.3392

Response time histogram:
  0.001 [1]     |
  0.011 [91109] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.021 [8229]  |■■■■
  0.031 [11]    |
  0.042 [0]     |
  0.052 [50]    |
  0.062 [400]   |
  0.072 [0]     |
  0.083 [0]     |
  0.093 [34]    |
  0.103 [166]   |

Latency distribution:
  10% in 0.0074 secs
  25% in 0.0079 secs
  50% in 0.0087 secs
  75% in 0.0096 secs
  90% in 0.0108 secs
  95% in 0.0118 secs
  99% in 0.0153 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0061 secs, 0.0007 secs, 0.1030 secs
  DNS-lookup:   0.0004 secs, 0.0000 secs, 0.0065 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0033 secs
  resp wait:    0.0031 secs, 0.0001 secs, 0.0967 secs
  resp read:    0.0000 secs, 0.0000 secs, 0.0483 secs

Status code distribution:
  [200] 100000 responses
```

```jsx
hey -n 100000 http://localhost:8080/netty

Summary:
  Total:        2.2352 secs
  Slowest:      0.0223 secs
  Fastest:      0.0001 secs
  Average:      0.0011 secs
  Requests/sec: 44739.2500

  Total data:   1100000 bytes
  Size/request: 11 bytes

Response time histogram:
  0.000 [1]     |
  0.002 [99538] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.005 [409]   |
  0.007 [2]     |
  0.009 [5]     |
  0.011 [11]    |
  0.013 [2]     |
  0.016 [11]    |
  0.018 [5]     |
  0.020 [6]     |
  0.022 [10]    |

Latency distribution:
  10% in 0.0009 secs
  25% in 0.0010 secs
  50% in 0.0011 secs
  75% in 0.0012 secs
  90% in 0.0013 secs
  95% in 0.0014 secs
  99% in 0.0020 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0001 secs, 0.0223 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0025 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0018 secs
  resp wait:    0.0010 secs, 0.0001 secs, 0.0149 secs
  resp read:    0.0000 secs, 0.0000 secs, 0.0023 secs

Status code distribution:
  [200] 100000 responses
```

# 参考

- [https://cacm.acm.org/magazines/2022/5/260357-the-go-programming-language-and-environment/fulltext](https://cacm.acm.org/magazines/2022/5/260357-the-go-programming-language-and-environment/fulltext)
- [https://github.com/netty/netty/blob/4.1/example/src/main/java/io/netty/example/http/helloworld/HttpHelloWorldServer.java](https://github.com/netty/netty/blob/4.1/example/src/main/java/io/netty/example/http/helloworld/HttpHelloWorldServer.java)
- [https://stackoverflow.com/questions/3732109/simple-http-server-in-java-using-only-java-se-api](https://stackoverflow.com/questions/3732109/simple-http-server-in-java-using-only-java-se-api)