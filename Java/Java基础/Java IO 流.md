#### IO 流分类

| 接口   | 字节输入流          | 字节输出流           | 字符输入流        | 字符输出流        |
| ------ | ------------------- | -------------------- | ----------------- | ----------------- |
| 接口   | InputStream         | OutputStream         | Reader            | Writer            |
| 文件   | FileInputStream     | FileOutputStream     | FileReader        | FileWriter        |
| 缓冲   | BufferedInputStream | BufferedOutputStream | BufferedReader    | BufferedWriter    |
| 转换   |                     |                      | InputStreamReader | OutputSreamWriter |
| 序列化 | ObjectInputStream   | ObjectOutputStream   |                   |                   |

字节输入流：以内存为基准，把磁盘文件中的数据或者网络中的数据以字节为单位读入到内存中去。

字节输出流：以内存为基准，把内存中的数据以字节为单位写出到磁盘文件或者网络中去。

字符输入流：以内存为基准，把磁盘文件中的数据或者网络中的数据以字符为单位读入到内存中去。

字符输出流：以内存为基准，把内存中的数据以字符为单位写出到磁盘文件或者网络中去。

> 字符流更适合读文本文件。

缓冲流：提高字节流和字符流读写数据的性能。

> BufferedReader多了一个readLine()方法，可以一次读一行数据。
>
> BufferedWriter多了一个newLine()方法，可以换行。

字符转换流：可以按照指定编码将字节流转换成字符流。

对象序列化流/对象反序列化流：可以将Java对象序列化到文件或者将文件反序列化成 Java 对象。

FileCopyDemo1

```java
InputStream fis = new FileInputStream("D:\\data\\person.txt");
OutputStream fos = new FileOutputStream("D:\\person.txt");
int len;
byte[] buffer = new byte[1024];
while((len = fis.read(buffer)) != -1){
    fos.write(buffer,0,len);
}
```

FileCopyDemo2

```java
BufferedReader br = new BufferedReader(new FileReader("D:\\data\\person.txt"));
BufferedWriter bw = new BufferedWriter(new FileWriter("D:\\data\\person.txt"));
String line;
while((line = br.readLine()) != null){
    bw.write(line);
    bw.newLine();
}
```

#### 字节流与字符流

问题本质：不管是文件读写还是网络发送接收，信息的最小存储单元都是字节，那为什么 I/O 流操作要分为字节流操作和字符流操作呢？

如果我们不知道编码类型就很容易出现乱码问题，所以字符流会方便我们平时对文本文件进行流操作。如果音频文件、图片等媒体文件用字节流比较好，如果涉及到字符的话使用字符流比较好。

Reader：字符流读取
InputStream：字节流读取
InputStreamReader：将字节流转换为字符流，可以指定编码格式

FileReader：InputStreamReader 的子类，适合读取文件
FileInputStream：InputStream 的子类

读文件：FileReader + Buffered
其他：new InputStreamReader(new FileInputStream())

springboot 线程池限制线程数量：主线程 acquire，子线程 release

#### BIO、NIO、AIO

* BIO (Blocking I/O)：同步阻塞 I/O 模式。 线程发起 IO 请求后，一直阻塞 IO，直到缓冲区数据就绪后，再进入下一步操作，即数据的读取写入必须阻塞在一个线程内等待其完成。

* NIO (Non-blocking I/O)：NIO 是一种同步非阻塞的 I/O 模型，Java NIO 中引入了 IO 多路复用技术。

  <img src="/Users/licheng/Documents/Typora/Picture/image-20200629104140536.png" alt="image-20200629104140536" style="zoom: 67%;" />

  * Buffer：本质上是一个数组，向 Channel 读写的数据都必须先置于缓冲区中。
  * Channel：通道只负责传输数据，不会操作和存储数据。一共有四种类型：
    * FileChannel：作用于文件 IO 流。
    * DatagramChannel：作用于 UDP 协议。
    * SocketChannel：作用于 TCP 协议。
    * ServerSocketChannel：作用于 TCP 协议。
  * Selector：用于监听多个通道的事件。

  客户端/服务器初始时会获取一个 Channel，每个 Channel 都会和 Selector 绑定一个事件，然后生成一个 SelectionKey 对象。Selector 会不断执行 `select()` 方法监听是否有 Channel 准备就绪。一旦 Channel 准备好，就可以通过 `selector.selectedKeys()` 获取就绪 Channel 的集合，进行后续的 IO 操作。

  在 NIO 中一共有四种事件：

  * `SelectionKey.OP_CONNECT` ：连接事件
  * `SelectionKey.OP_ACCEPT` ：接收事件
  * `SelectionKey.OP_READ` ：读事件
  * `SelectionKey.OP_WRITE` ：写事件

* AIO (Asynchronous I/O)：异步非阻塞的 IO 模型。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，当相关处理完成，操作系统会通知相应的线程进行后续的操作。

举例说明：

BIO：A 顾客去吃海底捞，就这样干坐着等了一小时，然后才开始吃火锅。

NIO：B 顾客去吃海底捞，他一看要等挺久，于是去逛商场，每次逛一会就跑回来看有没有排到他。于是他最后既购了物，又吃上海底捞了。

AIO：C 顾客去吃海底捞，由于他是高级会员，所以店长说，你去商场随便玩吧，等下有位置，我立马打电话给你。于是C顾客不用干坐着等，也不用每过一会儿就跑回来看有没有等到，最后也吃上了海底捞。

#### Socket

socket 屏蔽了各个协议的通信细节，使得程序员无需关注协议本身，直接使用 socket 提供的接口来进行互联的不同主机间的进程的通信。

<img src="/Users/licheng/Documents/Typora/Picture/image-20200813094959780.png" alt="image-20200813094959780" style="zoom: 67%;" />