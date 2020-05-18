Nginx

*Nginx* (engine x) 是一个高性能的[HTTP](https://baike.baidu.com/item/HTTP)和[反向代理](https://baike.baidu.com/item/反向代理/7793488)web服务器，同时也提供了IMAP/POP3/SMTP[服务](https://baike.baidu.com/item/服务/100571)。其特点是占有内存少，[并发](https://baike.baidu.com/item/并发/11024806)能力强，

# 1.基本概念

- 反向代理和反向代理

  **正向代理的过程，它隐藏了真实的请求客户端，服务端不知道真实的客户端是谁，客户端请求的服务都被代理服务器代替来请求**

  **反向代理服务器会帮我们把请求转发到真实的服务器那里去**

  **正向代理**代理的对象是客户端，**反向代理**代理的对象是服务端

- 负载均衡：

  由一个独立的统一入口来收敛流量，再做二次分发的过程就是「负载均衡」，它的本质和「分布式系统」一样，是「分治」。

- 动静分离：

  在Web开发中，通常来说，动态资源其实就是指那些后台资源，而静态资源就是指HTML，JavaScript，CSS，img等文件。

  一般来说，都需要将动态资源和静态资源分开，将静态资源部署在Nginx上，当一个请求来的时候，如果是静态资源的请求，就直接到nginx配置的静态资源目录下面获 资源，如果是动态资源的请求，nginx利用反向代理的原理，把请求转发给后台应用去处理，从而实现动静分离，加快解析速度。

# 2.常用命令

查看Nginx的版本号：**nginx -V**

启动Nginx：**start nginx**

快速停止或关闭Nginx：**nginx -s stop**

正常停止或关闭Nginx：**nginx -s quit**

配置文件修改重装载命令：**nginx -s reload**

# 3.配置文件

- 全局块

  从配置文件开始到events块之间的内容，设置一些影响nginx服务器整体运行的配置指令，包括配置用户组，允许生成的worker_processes数，日志存放路径等

  e.g:

  ```properties
  worker_processes  1
  ```

  worker_processes越大，可支持的并发处理量越多

- events块

  涉及的指令主要影响nginx与用户的网络连接

  e.g

  ```pro
  worker_connections  1024;  支持的最大连接数
  ```

- http块

  又可以进一步分为http全局块，server块

  - http全局块

    包括文件引入，日志自定义，连接超时时间等

  - server块

  ```
  server {
          listen       80 default_server;
          listen       [::]:80 default_server;
          server_name  182.254.161.54;
          root         /usr/share/nginx/html;
  
          # Load configuration files for the default server block.
          include /etc/nginx/default.d/*.conf;
  
          location / {
          proxy_pass http://pic; 
          }
  
          error_page 404 /404.html;
              location = /40x.html {
          }
  
          error_page 500 502 503 504 /50x.html;
              location = /50x.html {
          }
      }
  ```

  每个http块可以包含多个server块，每个server块即可视为一个虚拟主机

# 4.反向代理配置

1. windows的host文件中添加DNS记录，例如

   ```
   127.0.0.1  www.helloworld.com
   ```

2. nginx进行请求转发

   ```
   	#设定实际的服务器列表 
       upstream zp_server1{
           server 127.0.0.1:8089;
       }
   	server {
            ...
            #监听80端口，80端口是知名端口号，用于HTTP协议
           listen       80;
           
           #定义使用www.xx.com访问
           server_name  www.helloworld.com;
   		#反向代理的路径（和upstream绑定），location 后面设置映射的路径
           location / {
               proxy_pass http://zp_server1;
           } 
   }
   ```

3. 也可配置location使不同的请求转发至不同的地址

   ```
   location ~ /aa/ {
               proxy_pass http://127.0.0.1:8081;
           } 
    location ~ /bb/ {
               proxy_pass http://127.0.0.1:8083;
           }        
   ```

# 5.负载均衡配置

```
upstream test{ 
      server 11.22.333.11:6666 weight=1; 
      server 11.22.333.22:8888 down; 
      server 11.22.333.33:8888 backup;
      server 11.22.333.44:5555 weight=2; 
}

location / {
            proxy_pass http://test;
        } 
```

# 6.动静分离配置

```
server {
    server_name www.mylinuxops.com;
    listen 80;
    location / {
        root /data/www;
        index index.html;
    }
    location /app {
        proxy_pass http://app;
    }
    location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js)$ {
        root /data/static;
        index index.html;
    }
}
```

# 7.原理

在nginx启动后，会有一个master进程和多个worker进程。

- **Master 进程：**管理 Worker 进程。对外接口：接收外部的操作（信号）；对内转发：根据外部的操作的不同，通过信号管理 Worker；**监控：**监控 Worker 进程的运行状态，Worker 进程异常终止后，自动重启 Worker 进程。
- **Worker 进程：**所有 Worker 进程都是平等的。实际处理：网络请求，由 Worker 进程处理。Worker 进程数量**：**在 nginx.conf 中配置，一般设置为核心数，充分利用 CPU 资源，同时，避免进程数量过多，避免进程竞争 CPU 资源，增加上下文切换的损耗。

### **HTTP 连接建立和请求处理过程如下：**

- Nginx 启动时，Master 进程，加载配置文件。
- Master 进程，初始化监听的 Socket。
- Master 进程，Fork 出多个 Worker 进程。
- Worker 进程，竞争新的连接，获胜方通过三次握手，建立 Socket 连接，并处理请求。

### 当master接收到重新加载的信号会怎么处理(./nginx -s reload)?

master会重新加载配置文件，然后启动新的进程，使用的新的worker进程来接受请求，并告诉老的worker进程他们可以退休了，老的worker进程将不会接受新的，老的worker进程处理完手中正在处理的请求就会退出。



每个worker里面只有一个主线程，但一个worker可以同时处理多个请求。nginx采取异步非阻塞的方式处理请求，每个请求进来，worker线程将其注册处理转发给下游服务（如php-fpm）后，并不是挂起等待，而是切换处理别的请求。

Nginx的IO通常使用epoll，epoll函数使用了I/O复用模型。

### 惊群现象

惊群现象：惊群效应就是当一个fd的事件被触发时，所有等待这个fd的线程或进程都被唤醒。一般都是socket的accept()会导致惊群，很多个进程都block在server socket的accept()，一但有客户端进来，所有进程的accept()都会返回，但是只有一个进程会读到数据，就是惊群。
 Nginx 采用accept-mutex来解决惊群问题：当一个请求到达的时候，只有竞争到锁的worker进程才会惊醒处理请求，其他进程会继续等待，结合 timer_solution 配置的最大的超时时间继续尝试获取accept-mutex