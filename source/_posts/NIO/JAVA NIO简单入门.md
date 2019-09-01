---
title: JAVA NIO简单入门
tags: nio
categories: java
---

##  简述
在前面的一篇文章中我们有提到过JAVA的I/O模型，详见《[深入理解Tomcat架构（二）]("http://blog.duanshaojie.cn/2019/07/29/tomcat/深入理解Tomcat架构（二）")》，
所以这里拉出来再简单记录下，毕竟不经常用老是忘啊喂~

<!--more-->

##  IO和NIO的有什么区别？

*   I/O是针对流处理的；NIO是针对缓冲区处理

*   I/O是线程阻塞的；NIO是线程非阻塞的

*   NIO拥有Selector

##  NIO简介
这里主要记录的是NIO相关的知识点，所有可能关于I/O的就一笔带过
*   什么是NIO？
    *   NIO貌似有两种解释，一种是New IO，但我觉得Non-blocking IO可能解释的比较明了，NIO就是一个非阻塞的IO操作。
*   你应该要知道的知识，以下只是概述，详情看下一节
    *   Channel and Buffer：上面有说NIO与IO的区别之一，就是所有的数据都是从channel读到buffer，或者反之
    *   Non-blocking：非阻塞（同步），这里就是说如果线程读取数据，发现没有的时候，会立即返回（这个时候你可以继续做其他事情），
    只有说在获取到数据以后，这个线程才会挂起
    *   Selector：用于检查channel是否可读可写（也叫多路复用器）

##  一些使用
*   channel通道其实和我们的流操作类似，都是把本地文件，或者网络文件数据读取到这个通道内，如下：
```
//读取本地文件
RandomAccessFile file = new RandomAccessFile("c:/edison/workspace/spring-demo/test.txt", "rw");
FileChannel channel = file.getChannel();
```
*   而上面的channel和流操作的区别在于，input流和output流都是单向的操作，你要根据你的需求流程选择，
而channel却可以将数据读入或写出，如下
```
//读取本地文件
FileInputStream file = null;
        FileChannel channel = null;
        try {
            file = new FileInputStream("c:/edison/workspace/spring-demo/test.txt");
            channel = file.getChannel();

            ByteBuffer byteBuffer = ByteBuffer.allocate(15);
            while (file.read()!=-1) {
                channel.read(byteBuffer);
            }
            byteBuffer.position(0);
            byteBuffer.limit(byteBuffer.capacity());
            while (byteBuffer.remaining()>0) {
                System.out.print((char)byteBuffer.get());
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            file.close();
            channel.close();
        }

        //写操作
        FileInputStream in = new FileInputStream("c:/edison/workspace/spring-demo/test.txt");
                FileOutputStream out = new FileOutputStream("c:/edison/workspace/spring-demo/test2.txt");
                int b = 0;
                while ((b =in.read()) != -1) {
                    out.write(b);
                }
                out.close();
                in.close();
```
*   如果有人对上面的buffer操作不太了解的话，就看下这条
    *   首先ByteBuffer中主要有几个关键字，也许你应该要了解
    *   mark用于对当前position的标记

        position表示当前可读写的指针，如果是向ByteBuffer对象中写入一个字节，那么就会向position所指向的地址写入这个字节，如果是从ByteBuffer读出一个字节，那么就会读出position所指向的地址读出这个字节，读写完成后，position加1

        limit是可以读写的边界，当position到达limit时，就表示将ByteBuffer中的内容读完，或者将ByteBuffer写满了。

        capacity是这个ByteBuffer的容量，上面的程序中调用ByteBuffer.allocate(128)就表示创建了一个容量为capacity字节的ByteBuffer对象。

*   selector使用，先打开一个Selector，然后注册一个channel（首先监听OP_ACCEPT），当有链接进入时，再将注册状态更改为OP_READ。如下为服务端代码
```
        public static void main(String[] args) throws IOException {

            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.bind(new InetSocketAddress(8099));

            Selector selector = Selector.open();
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

            ByteBuffer buffer = ByteBuffer.allocate(200);
            while(true) {
                int select = selector.select();
                if (0 == select) {
                    continue;
                }

                Set<SelectionKey> selectionKeySet = selector.selectedKeys();
                Iterator<SelectionKey> selectionKeyIterator = selectionKeySet.iterator();
                while (selectionKeyIterator.hasNext()) {
                    SelectionKey key = selectionKeyIterator.next();
                    if (key.isAcceptable()) {
                        ServerSocketChannel serverSocketChannel1 = (ServerSocketChannel) key.channel();
                        SocketChannel channel = serverSocketChannel1.accept();
                        channel.configureBlocking(false);

                        channel.register(selector, SelectionKey.OP_READ);
                    }

                    if (key.isReadable()) {
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        socketChannel.read(buffer);
                        String result = new String(buffer.array());
                        System.out.println(socketChannel.getRemoteAddress());
                        System.out.println("remote say =" + result);
                    }
                }
                selectionKeyIterator.remove();
            }
        }
```