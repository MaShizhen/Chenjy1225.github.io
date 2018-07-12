---
layout: post
title:  "Android 客户端和 web服务器通信"
date:   2018-05-15 11:00:00 +0800
categories: Android
tags: Android
author: chenjy
---



* content
{:toc}

本篇简单介绍`Android`客户端和`web`服务器使用`socket`进行通讯，向客户端发送文件的`demo`。




## socket

> 套接字使用`TCP`提供了两台计算机之间的通信机制。客户端创建一个套接字，并尝试连接服务端的嵌套字。当连接建立时，服务器会创建一个 Socket 对象。客户端和服务器现在可以通过对 Socket 对象的写入和读取来进行通信。

`java.net.Socket`类代表一个套接字，并且` java.net.ServerSocket`类为服务器程序提供了一种来监听客户端，并与他们建立连接的机制。

TCP 是一个双向的通信协议，因此数据可以通过两个数据流在同一时间发送


## 服务端

为了实现向客户端发送文件，我们基于前面的[jFinal 文件上传](https://chenjy1225.github.io/2016/12/18/JFinal-project-upload/) 来完成。

```java

public class ServerUtils {

	private volatile static ServerUtils serverInstance;

	private static ServerSocket serverSocket;

	private ServerUtils() {
	}

	public static ServerUtils getServerInstance() {

		if (serverInstance == null) {
			synchronized (ServerUtils.class) {
				if (serverInstance == null) {
					serverInstance = new ServerUtils();
				}
			}
		}
		return serverInstance;
	}

	public void init(int port) {
		try {
			if (serverSocket == null) {
				serverSocket = new ServerSocket(port);
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	public static ServerSocket getServerSocket() {
		return serverSocket;
	}

     /**
       *  发送文件线程
       **/
	public static class SendThread implements Runnable {
		static Socket socket;
		File file;
		static FileInputStream fis;
		static DataOutputStream dos;

		public SendThread(File file) {
			this.file = file;
		}

		@Override
		public void run() {

			try {
                 // 监听并接受到此嵌套字的连接 (该方法会阻塞等待，直到客户端连到服务端的指定端口)
				socket = getServerSocket().accept();
              
				// 上传的模型文件
				File getFile = file;
				fis = new FileInputStream(getFile);

				// 获取嵌套字的输出流
				dos = new DataOutputStream(socket.getOutputStream());
                  // 嵌套字的输入流
				//dis = new DataInputStream(socket.getInputStream());
              
				// 模型名称和大小
				dos.writeUTF(getFile.getName());
				dos.flush();
				dos.writeLong(getFile.length());
				dos.flush();

				byte[] bytes = new byte[1024];
				int length = 0;

				while ((length = fis.read(bytes, 0, bytes.length)) != -1) {
					dos.write(bytes, 0, length);
					dos.flush();
				}

			} catch (IOException e) {
				e.printStackTrace();

			} finally {
				if (fis != null)
					fis.close();
				if (dos != null)
					dos.close();
				if (socket != null) {
					socket.close();
				}
			}

		}
	}

}

```


`UploadController`:

```java


private static ThreadPoolExecutor threadPool;
private static SendThread sendThread;

// jfinal 获取 Web Uploader 上传的文件 
final File getFile = getFile().getFile();

threadPool = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());

// 初始化 ServerSocket
ServerUtils.getServerInstance().init(port);

sendThread = new SendThread(getFile);

// 创建一个线程 向客户端发送文件
threadPool.submit(sendThread);


```

## 客户端

```java


   final Socket socket = new Socket();
        final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(4, 4,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>());

        threadPool.submit(new Runnable() {
            @Override
            public void run() {

                try {
                    socket.connect(new InetSocketAddress("192.168.0.100", 6001));
                    // 获取嵌套字输入流
                    final DataInputStream dis = new DataInputStream(socket.getInputStream());

                    // 获取文件名
                    String fileName = dis.readUTF();
                    // 获取文件大小
                    final long fileLength = dis.readLong();
                    File file = new File(App.RECEIVE_PATH + File.separatorChar + fileName);
                    final FileOutputStream fos = new FileOutputStream(file);

                    threadPool.execute(new Runnable() {
                        @Override
                        public void run() {
                            try {
                                int length = 0;
                                byte[] bytes = new byte[1024];

                                while ((length = dis.read(bytes, 0, bytes.length)) != -1) {
                                    fos.write(bytes, 0, length);
                                    fos.flush();

                                }
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }

                    });

                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });

```

## 应用场景

这里只是一个最基本的上传demo，每次用户web页面上传文件，服务端都会开启一个线程来发送文件。客户端连接并接收上传的文件，文件发送结束 关闭客户端。

关于socket 优化以及深入了解

可参见[Java Socket编程基础及深入讲解](https://www.cnblogs.com/yiwangzhibujian/p/7107785.html#q2)


## NIO

上述的demo是基于`阻塞式api`的，当程序输入或输出操作后，在操作返回前会一直阻塞线程。服务器需要为每一个客户端提供一个线程进行处理。当有大量的客户端的时候，性能比较底下。

>JDK1.4 开始提供了 NIO(new io)开发高性能的服务器，可以使服务器使用一个或有限几个线程同时处理所有客户端

### IO和NIO的区别

面向流和面向缓冲区

* `io`面向流，从上面的`demo`可以看出所有输入输出都是通过流来完成。每次从流中读出数据它们没有被缓存在任何地方，直到被读完。需要需要前后移动数据需要先将数据缓存到一个缓冲区。

* `nio`面向缓冲区，面向块。使用`channel`(通道)模拟传统输入输出流，所有从`channel`读取的数据或是发送的数据都需要先放到`buffer`(缓冲)中。程序不能直接对channel直接操作。使数据操作更加灵活

阻塞式和非阻塞式

* 阻塞式如上面所述，需要阻塞直到数据完全读取写入。

* 非阻塞式，一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。 线程通常将非阻塞io的空闲时间用于在其它通道上执行io操作，所以一个单独的线程现在可以管理多个输入和输出通道。

### Selector

`nio`通过`Selector`(选择器)来实现一个线程对多个`channel`的管理。

服务器上的所有`channel`都需要向`Selector`注册，而`Selector`负责监视这些`channel`的状态。当有任意个`channel`有可用的io操作时,`Selector`的`select()`方法会返回一个大于0的值，表示当前有多少`channel`有可用的io操作。可以通过`selectedKeys()`获取`channel`集合。所以只需要一个线程不断调用`select()`方法即可。

Tips：无可用的`channel`时，调用`select()`方法的线程会被阻塞。


## Netty

> Netty 是业界流行的 NIO 框架之一，它的健壮性、功能、性能、可定制性和可扩展性在同类框架中都说首屈一指的，也已经得到了成百上千商用项目的验证。

优点有很多..... 反正就是很好用 很耐用，而且开发门槛低。.








[Netty Api](http://netty.io/4.1/api/index.html)













