[TOC]



## NIO是啥？ 

NIO是Java从JDK1.4开始引入的一系列改进版输入输出处理手段，也就是New IO，简称NIO，也有说法叫NonBlocking IO，是**同步非阻塞式的IO模型**，准确地说它支持阻塞非阻塞两种模式。

笔者在[NIO、BIO、AIO、同步异步、阻塞非阻塞傻傻分不清楚？](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/112105183)一文中详细总结了同步、非阻塞等相关概念，分析了NIO与传统BIO的主要区别。

本篇主要介绍NIO提供的三大组件的概念及使用：Buffer，Channel，Selector。

## Buffer

Buffer可以理解为是一个容器，用于**存储数据**，**本质是个数组**，存储的元素类型是基本类型。

无论是发送还是读取Channel中的数据，都必须先置入Buffer。

`java.nio.Buffer`是一个抽象类，子类包括有**除boolean外其他所有基本类型的XxBuffer**，最常用的是ByteBuffer。

### Buffer中的重要概念

capacity：缓冲区的容量，表示该Buffer的最大数据容量，即最多可以存储多少数据。

limit：限制位，标记position能够到达的最大位置，默认为缓冲区最后一位。

position：操作位，指向即将操作位置，默认指向0。

mark：可选标记位。默认不启用，Buffer允许直接将position定位到mark处。

> 他们满足的关系：mark <= position <= limit <= capacity

![image-20201219011845686](img/NIO/image-20201219011845686.png)

以`ByteBuffer.allocate(capacity)`为例，说明几个重要的过程：

- 初始化创建HeapByteBuffer，mark = -1，position = 0，limit = cap。
- 通过put方法向Buffer中加入数据，position++。
- 装入数据结束后，**调用flip方法，将limit设置为position所在位置，position置为0**，表示[position，limit]这段需要开始进行输出了【可以使用get方法读取数据】。
- 输出结束后，调用clear方法，将position置为0，limit置为cap，为下一次读取数据做好准备。

Buffer的所有子类都提供了put和get方法，对应向Buffer存入数据和从Buffe中取出数据，方式分为以下两种：

- 相对：从Buffer的当前pos处开始读取或写入数据，然后将pos的值按处理元素的个数增加。
- 绝对：直接根据索引向Buffer中读取或写入数据，不会影响pos位置本身的值。

### Buffer使用Demo

```java
    public static void main(String[] args) {

        // 创建 bytebuffer  allocate指定底层数组长度
        ByteBuffer byteBuffer = ByteBuffer.allocate(10);

        // 添加数据
        byteBuffer.put("summerday".getBytes());

        // 获取操作位的位置pos = 9
        System.out.println(byteBuffer.position());
        
        // 如果我想遍历summerday呢?哪里结束? limit = 10
        System.out.println(byteBuffer.limit());

        //反转缓冲区 可以利用flip()代替下面两步操作
        //将limit移到position的位置
        byteBuffer.limit(byteBuffer.position());
        
        //将pos移到0
        byteBuffer.position(0);

        // position < limit 
        // 可以利用 hasRemaining代替判断pos和limit之间是否还有可处理元素。
        while(byteBuffer.position() < byteBuffer.limit()){
            // 获取数据
            byte b = byteBuffer.get();
            System.out.println(b);
        }
    }


```

### 常用方法介绍

其实Buffer操作的逻辑比较简单，每个方法操作的字段也不外乎上面介绍的几个，下面是一些常用的方法：

**设置方法**

- Buffer position(newPosition)： 将pos设置为newPosition。
- Buffer limit(newLimit)：将limit设置为newLimit。

**数据操作**

- Buffer reset：将pos置为mark的位置。
- Buffer rewind：将pos置为0，取消设置的mark。
- Buffer flip: 将limit置pos位置，pos置0。
- Buffer clear：将position置为0，limit置为cap。

其他操作

```java
    public static void main(String[] args) {

        // 如果数据已知,可以使用wrap方法创建ByteBuffer
        ByteBuffer byteBuffer = ByteBuffer.wrap("summerday".getBytes());
        // 获取底层字节数组
        byte[] array = byteBuffer.array();
        System.out.println(new String(array));

    }
```

## Channel

### Channel概述

Channel 类似于传统的流对象，但有些不同：

- Channel 直接将指定文件的部分或全部直接映射 Buffer。
- **程序不能直接访 Channel 中的数据，包括读取、写入都不行**，Channel只能与 Buffer 进行交互。意思是，程序要读数据时需要先通过Buffer从Channel中获取数据，然后从Buffer中读取数据。
- Channel通常可以异步读写，但默认是阻塞的，需要手动设置为非阻塞。

Channel不应该通过构造器来直接创建，而是通过传统的节点InputStream、OutputStream的getChannel()方法来返回对应的Channel，或者通过RandomAccessFile对象的getChannel方法。

Channel中最常用的三种方法：

- map()：将Channel对应的部分或全部数据映射成ByteBuffer。
- read()：从Buffer中读取数据。
- write()：向Buffer中写入数据。

### RandomAccessFile#getChannel

下面是个简单的示例，通过RandomAccessFile的getChannel方法：

```java
    public static void main(String[] args) throws FileNotFoundException {

        RandomAccessFile file = new RandomAccessFile("D://b.txt", "rw");
        // 获取RandomAccessFile对应的channel
        try (FileChannel fileChannel = file.getChannel()) {
            ByteBuffer buf = ByteBuffer.allocate(48);
            int byteRead = fileChannel.read(buf);
            while(byteRead != -1){
                System.out.println("read " + byteRead);
                buf.flip();
                while (buf.hasRemaining()){
                    System.out.println((char) buf.get());
                }
                buf.clear();
                byteRead = fileChannel.read(buf);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
```

### SocketChannel与ServerSocketChannel

Java为Channel接口根据不同功能，提供了不同的实现类，比如我们下面的示例：支持TCP网络通信的SocketChannel和ServerSocketChannel。

```java
public class Client {

    public static void main(String[] args) throws IOException {
        // 开启客户端的channel
        SocketChannel sc = SocketChannel.open();
        // 手动设置为非阻塞模式
        sc.configureBlocking(false);
        // 发起连接
        sc.connect(new InetSocketAddress(8081));
        // 手动判断保证连接的建立
        while(!sc.isConnected()){
            // 如果多次连接都没有脸上,会认为此次连接无法建立
            sc.finishConnect();
        }
        // 发送数据
        sc.write(ByteBuffer.wrap("hello , i am client".getBytes()));
        // 关闭通道
        sc.close();
    }
}


public class Server {

    public static void main(String[] args) throws IOException {
        // 开启服务器端的通道
        ServerSocketChannel ssc = ServerSocketChannel.open();
        // 监听端口号
        ssc.bind(new InetSocketAddress(8081));
        // 手动设置为非阻塞模式
        ssc.configureBlocking(false);
        // 接收连接
        SocketChannel sc = ssc.accept();
        // 保证连接
        while (sc == null) {
            sc = ssc.accept();
        }
        // 读取数据
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        sc.read(buffer);
        byte[] bs = buffer.array();
        System.out.println(new String(bs, 0, buffer.position()));
        // 关流
        ssc.close();

    }
}
```

## Selector

NIO实现非阻塞IO的其中关键组件之一就是**Selector多路复用选择器**，可以注册多个Channel到一个Selector中。Selector可以不断执行select操作，判断这些注册的Channel是否有已就绪的IO事件，如可读，可写，网络连接已完成等。

一个线程通过使用一个Selector管理多个Channel。

```java
public class Server {

    public static void main(String[] args) throws IOException {

        // 开启服务端的通道
        ServerSocketChannel ssc = ServerSocketChannel.open();
        // 设置非阻塞
        ssc.configureBlocking(false);

        // 绑定端口
        ssc.bind(new InetSocketAddress(8081));

        // 开启选择器
        Selector selector = Selector.open();
        // 将通道注册到选择器上
        ssc.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            // 选择已注册的通道
            selector.select();
            // 获取选择通道的事件
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = keys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                // 接收
                if (key.isAcceptable()) {
                    // 从事件中获取通道
                    ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                    // 接收连接
                    SocketChannel sc = channel.accept();
                    // 设置非阻塞
                    sc.configureBlocking(false);
                    // 注册读 + 写事件
                    sc.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                }
                // 读
                if (key.isReadable()) {
                    // 获取通道
                    SocketChannel sc = (SocketChannel) key.channel();
                    // 读取数据到buffer
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    sc.read(buffer);
                    // 反转缓冲区
                    buffer.flip();
                    System.out.println(new String(buffer.array(), 0, buffer.limit()));
                    // 在同一通道上注册,将会将之前注册的事件给注册
                    // 注销read事件
                    sc.register(selector, key.interestOps() ^ SelectionKey.OP_READ);
                }
                // 写
                if (key.isWritable()) {
                    // 获取通道
                    SocketChannel sc = (SocketChannel) key.channel();
                    sc.write(ByteBuffer.wrap("hello client, i am server!".getBytes()));
                    // 注销write事件
                    sc.register(selector, key.interestOps() ^ SelectionKey.OP_WRITE);
                }
            }
            iterator.remove();
        }
    }

}


public class Client {

    public static void main(String[] args) throws IOException {

        SocketChannel sc = SocketChannel.open();
        sc.connect(new InetSocketAddress(8081));
        sc.write(ByteBuffer.wrap("hello ! i am client !".getBytes()));
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        sc.read(buffer);
        System.out.println(new String(buffer.array(), 0, buffer.position()));

    }
}

```

## 参考阅读

- [http://ifeve.com/overview/](http://ifeve.com/overview/)
- 《疯狂Java讲义》
- [NIO、BIO、AIO、同步异步、阻塞非阻塞傻傻分不清楚？](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/112105183)