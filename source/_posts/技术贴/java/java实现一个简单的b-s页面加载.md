---
title: java实现一个简单的b/s页面加载
tags:
  - java
categories:
  - technical
  - java
toc: true
date: 2020-05-31 15:53:25
---

## 效果图

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200531155507.png)
<!-- more -->
## 模拟B/S服务器

1. 准备页面数据，放到web文件夹中
2. 模拟服务器端：
   ```java
    import java.io.*;
    import java.net.ServerSocket;
    import java.net.Socket;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;

    public class BS_Server {
        public static void main(String[] args) throws IOException {
            ServerSocket serverSocket = new ServerSocket(8080);
            //始终开启服务器
            while(true){
                //获取当前接入的Socket
                Socket socket = serverSocket.accept();
                //创建线程池   处理多个资源（html、图片等）加载
                ExecutorService es = Executors.newFixedThreadPool(10);
                es.submit(()-> {
                        try{
                            //获取网络字节输入流
                            InputStream is = socket.getInputStream();
                            //只读取请求第一行即可：GET /day13/web/index.html HTTP/1.1
                            //先转换成字符缓冲流
                            BufferedReader br = new BufferedReader(new InputStreamReader(is));

                            //字符串的处理：
                            //GET /day13/web/index.html HTTP/1.1
                            String getMessage = br.readLine();
                            //根据空格分割开
                            String[] messages = getMessage.split(" ");
                            //第1个为：/day13/web/index.html ， 去掉开头的/
                            String sourcesPath = messages[1].substring(1);

                            //读取本地资源返回给浏览器：
                            //获取网络输出流
                            OutputStream os = socket.getOutputStream();
                            //先写响应头部，固定的格式
                            os.write("HTTP/1.1 200 OK\r\n".getBytes());
                            os.write("Content-Type: text/html\r\n".getBytes());
                            //必须要写入空行，不然浏览器不解析
                            os.write("\r\n".getBytes());
                            //创建本地文件读取流
                            FileInputStream fis = new FileInputStream("mydemo\\src\\work\\gumptlu\\javaAdvanced\\" + sourcesPath);
                            byte[] bytes = new byte[102400];
                            int len = 0;
                            while((len = fis.read(bytes)) != -1){
                                //发送数据
                                os.write(bytes);
                            }
                            //关闭
                            fis.close();
                            br.close();
                            socket.close();
                        }catch (IOException e){
                            System.out.println(e);
                        }
                    }
                );

            }
            //始终开启服务器，不用关闭
            //serverSocket.close();
        }
    }
   ```

3. 打开浏览器访问：`http://127.0.0.1:8080/day13/web/index.html`


> 作者：gumptlu

> 转载请注明出处！