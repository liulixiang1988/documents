---
typora-copy-images-to: img
---

# Nginx 笔记

[TOC]



## 1. 为什么是Nginx?



Nginx可以：

- 反向代理（Reverse Proxy )
- 负载均衡(Load Balancing)
- 内容缓存(Content Caching)
- 监控与管理(Monitoring & Management)

## 2. 开始

### 2.1 安装Nginx

包安装：

```bash
rpm -qa | grep grep epel-release
yum install epel-release
yum install nginx
```

配置文件目录：

```
/etc/nginx
```

最重要的是`nginx.conf`文件

日志目录`/var/log/nginx`

启动nginx: `service nginx start`

查看nginx: `netstat -ntlp | grep nginx`

重新加载：`service nginx reload`

### 2.2 Nginx架构

![1522502296763](D:\notes\img/1522502296763.png)

查看nginx的进程树：`ps -ef --forest | grep nginx`

nginx.conf 文件中下面四个是最基本的：

- `user nginix`: worker进程的运行用户。注意，master进程的运行用户是启动时所用的用户
- `worker_processes auto`:worker进程的数量，当为auto时表示等于CPU的核数
- `error_log /var/log/nginx/error.log`: 错误日志的为止
- `pid logs/nginx.pid`: master进程的PID

### 2.3 Nginx配置——Event与HTTP

#### 2.3.1 Event

```
events {
    worker_connections  1024;
}
```

- `worker_connections`表示每个worker进程能够处理的连接数量

#### 2.3.2 HTTP

```
http {
    # ...
}
```

我们详细分解一下：

1. 日志格式与访问日志的路径

```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
      '$status $body_bytes_sent "$http_referer" '
      '"$http_user_agent" 
```

2. 包含其它配置

```
http {
  include mime.types;
  include /etc/nginx/conf.d/*.conf;
}
```

验证nginx配置是正确：`nginx -t`

### 2.4 配置第一个网站

```
http {
  server {
    listen 80;
    server_name  localhost;
    root html;
    location / {
            root   html;
            index  index.html index.htm;
    }
  }
}
```

指定网站根目录与首页：

```
root /usr/share/nginx/html;
index index.html index.htm;
```

上面这部分也可以直接放到根目录下面。

### 2.5 MIME

mime.types

```
types{
  text/html	html htm;
  video/mp4	mp4;
}
```

包含mime.types，遇到匹配的文件类型时添加对应的content-type头

nginx.conf

```
http {
  include mime.types;
  # 默认mime
  default_type application/octet-stream;
}
```

## 3. 反向代理

### 3.1 可以用反响代理做什么

- 隐藏了原始的后台服务
- 保护服务器免于网络攻击
- 提供缓存功能
- 优化内容与压缩
- SSL代理
- 请求路由

![1522578748029](D:\notes\img/1522578748029.png)

### 3.2 配置Nginx为反向代理

`proxy_pass`声明了要代理的后端服务

```
http {
  server {
    server_name _; #原始Host
    location / {
      proxy_pass http://192.168.10.50;
    }

    # 或者
    location /admin {
      proxy_pass http://192.168.10.50;
    }
    location /app {
      #包含了URI
      proxy_pass http://192.168.10.100/application;
    }
  }
}

```

### 3.3 X-Real-IP

```
server {
  server_name liulx.io;
  
  location / {
    proxy_pass http://192.168.189.139;
    # 设置真实IP
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

配置好后，我们可以在日志中打印`http_x_real_ip`，这个就是真实的IP

```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
      '$status $body_bytes_sent "$http_referer" '
      '"$http_user_agent" "$http_x_real_ip"';
```

### 3.4 Proxy Host Header

![1522580700106](D:\notes\img/1522580700106.png)

当反响代理的后端服务绑定多个域名时，为了保证能正确的访问到正确的域名，我们需要把原始访问的域名带给后端，因此需要设置`Host`

```
server {
  server_name _;
  
  location / {
    proxy_pass http://192.168.189.139;
    # 设置真实HOST
    proxy_set_header Host $host;
  }
```

## 4. 负载均衡(Load Balancer)

![1522729713301](D:\notes\img\1522729713301.png)

### 4.1 配置基本的Load Balancer

定义upstream，并且在server中引用upstream

```
upstream backend {
    server 52.4.12.3;
    server 52.3.20.1;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
    }
}
```

可以人为把一个节点下线：

```
upstream backend {
    server 52.4.12.3 down;
    server 52.3.20.1;
}
```

### 4.2 Nginx对后端主动进行健康监控(Active Health Monitoring)

```
upstream backend {
    zone backend 64k;
    server 52.4.12.3;
    server 52.3.20.1;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
        health_check;
    }
}
```

`health_check`默认每5s向后端访问`/`，如果能正常响应，则说明后端服务正常。

可以给health_check设置如下属性：

- interval: 检查间隔
- fails: 失败次数，当检查失败次数为指定次数时，认为backend已经down了
- passes: 成功次数，当一个backend已经down了，如果再连续指定次数成功，则表明又恢复了。

```
location / {
    proxy_pass http://backend;
    health_check interval=10 fails=3 passes=2;
}
```

### 4.3 条件匹配(Match Condition)

我们可以指定健康检查时的匹配条件：

```
upstream backend {
    server 52.4.12.3;
    server 52.3.20.1;
}

match server_test {
    status 200-300;
    body !~ "maintenance mode";
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
        health_check url=/test.txt match=server_test;
    }
}
```

我们也可以指定健康检查响应body其它条件，比如包含某项内容时是ok的，不包含某项是ok的：

```
match welcome {
    status 200;
    header Content-Type = text/html;
    body ~ "Welcome to nginx!";
}

match not_redirect {
    status ! 301-303 307;
    header ! Refresh;
}
```

健康检查除了可以用与HTTP协议之外，还能用于其它协议，比如FastCGI, memcached, SCGI, uwsgi, TCP和UDP等。

### 4.4 共享内存与健康检查

![1522731049216](D:\notes\img\1522731049216.png)

默认情况下，每个worker之间是不共享内存的，每个健康检查都会自己保存检查结果。我们向让一个worker对后端的检查结果，其它worker也能使用，可以使用共享内存 `zone backend 64k`。其中backend是我们对共享内存的命名。

```
upstream backend {
    zone backend 64k;
    server 52.4.12.3;
    server 52.3.20.1;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
        health_check;
    }
}
```

### 4.5 主动与被动健康检查(Active & Passive)

前面讲的`health_check`是主动健康检查，还有被动健康检查。被动健康检查是直接将前端请求发送给后端，如果后端失败时，则重新发送到别的后端，并记录失败次数，当失败次数达到限值时，则标记对应的后端为不可用，并在指定时间内不再访问该后端。下面表示连续3次失败后，30秒内不再访问第二个后端。

```
upstream backend {
    server 52.4.12.3;
    server 52.3.20.1 max_fails=3 fail_timeout=30s;
}
```

### 4.6 Server Weights

Load Balancer默认采用Round Robin算法，也就是轮询，但是我们可以进行设置权重：

```
upstream backend {
    server 52.4.12.3;
    server 52.3.20.1 weight=3;
}
```

上面表示，每4个请求，有3个会走第二个服务。没有设置weight的，weight默认为1.

### 4.7 负载均衡方法：最小连接数(Least Connect)

有这种场景，有两个请求，A与B，执行A请求需要20s，执行B请求只需要1s，如果LB使用Round Robin算法，那么，有可能所有的耗时连接都会发送到同一个后端上（比如请求顺序时A-B-A-B）。为了改进，我们可以使用最小连接数的方法，也就是看后端的连接数，那个最小使用哪一个后端。

```
upstream backend {
    least_conn;
    server 52.4.12.3;
    server 52.3.20.1;
}
```

使用apache benchmark可以进行测试：

```
ab -n 20 -c 10 loadbalancer_host:port
```





