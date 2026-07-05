---
title: Java NIO — Channel/Buffer/Selector
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - NIO
  - Channel
  - Buffer
  - Selector
categories:
  - Java
  - 网络
---

![Java NIO: Channels and Buffers](/java/netty/overview-channels-buffers.png)

在 Java NIO 中，数据总是从 Channel 读入 Buffer，或者从 Buffer 写入 Channel。

| Channel 的实现      | 作用                                                     |
| ------------------- | -------------------------------------------------------- |
| FileChannel         | 支持文件读写                                             |
| DatagramChannel     | 支持 UDP 报文读写                                        |
| SocketChannel       | 支持 TCP 读写                                            |
| ServerSocketChannel | 监听新建立的 TCP 连接，对每个新的连接创建一个 SocketChannel |

| Buffer 的实现 | 作用 |
| ------------- | ---- |
| ByteBuffer    |      |
| CharBuffer    |      |
| DoubleBuffer  |      |
| FloatBuffer   |      |
| IntBuffer     |      |
| LongBuffer    |      |
| ShortBuffer   |      |
