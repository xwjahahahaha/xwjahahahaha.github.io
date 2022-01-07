---
title: java实现一个简单的c/s文件上传
tags:
  - java
categories:
  - technical
  - java
toc: true
date: 2020-05-30 15:43:21
---
**一个简单的图片文件c/s模拟上传**

>主要使用的内容：

>Socket、ServerSocket、多线程

<!-- more -->

## 代码

* 客户端； ClientSocket.java
  ```java

  import java.io.*;
  import java.net.Socket;

  public class ClientSocket {
      public static void main(String[] args) throws IOException {
          //客户端:
          //创建Socket对象
          Socket socket = new Socket("127.0.0.1", 8888);
          //创建网络输出流
          OutputStream os = socket.getOutputStream();
          //写数据
          os.write("你好服务器".getBytes());
          //接收服务器传回数据
          InputStream is = socket.getInputStream();
          //处理
          byte[] bytes = new byte[102400];
          int len = is.read(bytes);
          String resStr = new String(bytes, 0, len);
          System.out.println(resStr);
          //判断是否建立了连接
          if("收到！谢谢".equals(resStr)){
              //开始上传文件
              System.out.println("客户端开始上传文件");
              //首先读取本地文件，使用本地流
              FileInputStream fis = new FileInputStream(new File("mydemo/pic/xwj.jpg"));
              while ((len = fis.read(bytes)) != -1){
                  //要网络流去传，而不是本地流
                  os.write(bytes, 0, len);
              }
              fis.close();
              // !!!重点：客户端到-1结束，但是-1不会传送给服务器，故要使用shutdownOutput
              socket.shutdownOutput();
              //接收服务器的消息
              len = is.read(bytes);
              System.out.println(new String(bytes, 0 ,len));
          }else{
              System.out.println("服务器未响应，无法传送文件");
          }
          //关闭客户端Socket
          socket.close();
      }
  }
  ```

* 服务器端：  Server_Socket.java
  ```java
  import java.io.*;
  import java.net.ServerSocket;
  import java.net.Socket;
  import java.util.Random;

  public class Server_Socket {
      public static void main(String[] args) throws IOException {
          //服务器端
          //创建ServerSocket对象 指定端口
          ServerSocket serverSocket = new ServerSocket(8888);
          //让服务器一直接受连接，传送图片数据
          while(true) {
              //获取该端口的客户端
              Socket socket = serverSocket.accept();

              //为了增加效率，使用多线程处理
              new Thread(new Runnable() {
                  @Override
                  public void run() {
                      //线程任务：处理文件的上传
                      //注意！！！因为父类不抛出异常，所以这里要把与异常处理掉
                      try {
                          //创建接收流
                          InputStream is = socket.getInputStream();
                          //接收客户端数据,处理
                          byte[] bytes = new byte[102400];
                          int len = is.read(bytes);
                          System.out.println(new String(bytes, 0, len));
                          //向客户端返回数据
                          OutputStream os = socket.getOutputStream();
                          os.write("收到！谢谢".getBytes());
                          //开始接收文件
                          //创建一个本地输出流，将客户端传来的数据保存到本地的硬盘中
                          System.out.println("服务器开始接收数据");
                          //服务器接收文件的文件夹
                          File file = new File("/upload");
                          if (!file.exists()) {
                              //文件夹不存在就创建
                              file.mkdir();
                          }
                          //接收图片名称的拼接
                          String imgName = "gumptlu" + System.currentTimeMillis() + new Random().nextInt(999999) + ".jpg";
                          FileOutputStream fos = new FileOutputStream(file + "/" + imgName);
                          //使用网络流读取，而不是本地流
                          while ((len = is.read(bytes)) != -1) {
                              //写到硬盘中
                              fos.write(bytes, 0, len);
                          }
                          fos.close();
                          //写入完毕，发送客户端响应
                          os.write("服务器：文件已经保存！".getBytes());
                          //关闭
                      }catch (Exception e){
                          System.out.println(e);
                      }finally {
                          try {
                              socket.close();
                          }catch (Exception e){
                              System.out.println(e);
                          }

                      }
                  }
              }).start();
          }
          //服务器一直接受，不关闭
          //serverSocket.close();
      }
  }

  ```

## 注意的点

1. 客户端读本地文件，服务器向硬盘写文件都是用**本地流**,客户端与服务器之间传送数使用**网络流**
2. 阻塞：客户端读本地文件到-1后停止，但是不会把-1发送给服务器，故要使用Socket的**shutdownOutput**方法使得服务器结束
3. 服务器端死循环实现反复接受客户端的数据，使用多线程来加速传输

> 作者：gumptlu

> 转载请注明出处！