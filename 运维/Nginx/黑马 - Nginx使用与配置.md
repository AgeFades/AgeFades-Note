[TOC]

# 黑马 - Nginx使用与配置

## 什么是Nginx

> Nginx是一个高性能的HTTP和反向代理服务，也是一个IMAP/POP3/SMTP服务。

1. 处理响应请求很快
2. 高并发连接
3. 低的内存消耗
4. 具有很高的可靠性
5. 高扩展性
6. 热部署

> master管理进程与worker工作进程的分离设计，使得Nginx具有热部署的功能。
>
> 可以不停机 升级Nginx的可执行文件、修改配置文件、更换日志文件 等

### 可大量并行处理

> 官方测试中，Nginx能支持5w个并行连接，而在实际的运作中，可以支持2w至4w个并行连接。
>
> 作为对比，Tomcat的并行连接数只有几百个。

#### 常见Web服务器对比

| **对比项\服务器** | **Apache** | **Nginx** | **Lighttpd** |
| ----------------- | ---------- | --------- | ------------ |
| Proxy代理         | 非常好     | 非常好    | 一般         |
| Rewriter          | 好         | 非常好    | 一般         |
| Fcgi              | 不好       | 好        | 非常好       |
| 热部署            | 不支持     | 支持      | 不支持       |
| 系统压力          | 很大       | 很小      | 比较小       |
| 稳定性            | 好         | 非常好    | 不好         |
| 安全性            | 好         | 一般      | 一般         |
| 静态文件处理      | 一般       | 非常好    | 好           |
| 反向代理          | 一般       | 非常好    | 一般         |

### Nginx模块

> `整体采用模块化设计`是Nginx的一个重大特点，甚至http服务器核心功能也是一个模块。

旧版本的Nginx模块是静态的，添加和删除模块都要对Nginx进行重新编译。

1.9.11x 版本以后，支持动态模块加载。

高度模块化的设计是Nginx的架构基础，Nginx服务器被分解为多个模块，每个模块就是一个功能模块，只负责自身功能。

模块之间严格遵循 `高内聚、低耦合` 的原则。

![1699954513271.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1699954513271.png)

#### 核心模块

> 核心模块是Nginx服务器正常运行必不可少的模块，提供错误日志记录、配置文件解析、事件驱动机制、进程管理等核心功能。

#### 标准HTTP模块

> 提供HTTP协议解析相关的功能，如：端口配置、网页编码设置、HTTP响应头设置等。

#### 可选HTTP模块

> 主要用于扩展标准的HTTP功能，让Nginx能处理一些特殊的服务，如：Flash多媒体传输、解析GeoIP请求、SSL支持等。

#### 邮件服务模块

> 主要用于支持Nginx的邮件服务，包括对 POP3/IMAP/SMTP 协议的支持。

#### 第三方模块

> 扩展Nginx服务器的应用，完成开发者自定义功能，如：JSON支持、Lua支持等。

### Nginx应用场景

> Nginx是HTTP服务器和反向代理服务器；同时也是IMAP/POP3/SMTP代理服务器；
>
> 可以作为一个HTTP服务器进行网站的发布处理、可以作为反向代理进行负载均衡的实现。

#### 关于代理

##### 正向代理

> 代理客户端，代客户端发出请求。
>
> 举例：本机 -> VPN(正向代理) -> 外网。
>
> 用途：
>
> 1. 访问原来无法访问的资源，如Google
> 2. 可以做缓存，加速访问资源
> 3. 对客户端访问授权，上网进行认证
> 4. 代理可以记录用户访问记录（上网行为管理），对外隐藏用户信息

##### 反向代理

> 代理服务端，代服务端接收请求。
>
> 举例：本机 -> 百度域名(nginx) -> 后面的千千万万台服务器。
>
> 主要用于服务器集群分布式部署的情况下，反向代理隐藏了服务器的信息。
>
> 作用：
>
> 1. 保证内网的安全，通常将反向代理地址作为公网访问地址，Web服务器是内网。
> 2. 负载均衡，通过反向代理服务器来优化网站的负载。

## Nginx的安装

简单记录下yum安装方式

```shell
yum install -y nginx
systemctl start nginx.service
systemctl enable nginx.service
```

## Nginx常用命令

```bash
# 启动
nginx
nginx -c nginx.conf # 指定执行的配置文件

# 停止
nginx -s stop

# 退出
nginx -s quit

# 关闭
ps -aux | grep nginx
kill -9 nginx

# 重新加载配置文件
nginx -s reload

# 检查配置文件是否正确
nginx -t

# 查看nginx版本信息
nginx -v
```

## 配置文件结构

> Nginx安装目录下的conf目录下。
>
> 整个文件以block形式组合而成，每一个block都是用 `{}` 来表示。
>
> block中可以嵌套其他block层级，其中`main`层是最高层次。

Nginx配置文件主要有4部分：

- Main：全局设置，影响其他所有部分设置
- Server：主机设置，主要用于指定虚拟机主机域名，ip、port
- Upstream：上游服务器设置，主要为反向代理，负载均衡相关配置
  - 其指令用于设置一系列的后端服务器，设置反向代理及后端服务器的负载均衡
- Location：url匹配特定位置的设置，用于匹配网页位置（如: 根目录 "/"、图片目录"/images" 等）

它们之间的关系：Server 继承 Main，Location 继承 Server，Upstream 既不会继承指令也不会被继承。

在这4部分中，每个部分都包含若干指令，主要包含Nginx的主模块指令、事件模块指令、HTTP核心模块指令，同时每个部分还可以使用其他HTTP指令，例如 Http SSL 模块，HttpGzip Static 模块、Http Addition 模块...

通俗理解：**HTTP模块负责配置请求的全局参数配置信息。Server模块配置好服务监听信息，等待请求进入，根据主机信息，确认是哪一个来处理该请求。Location模块找哪个URL信息，去哪里找等。**

### 真实Nginx配置文件例子

```apl
########### 每个指令必须有分号结束。#################
#user administrator administrators;  #配置用户或者组，默认为nobody nobody。
#worker_processes 2;  #允许生成的进程数，默认为1
#pid /nginx/pid/nginx.pid;   #指定nginx进程运行文件存放地址
error_log log/error.log debug;  #指定日志路径，级别。这个设置可以放入全局块，http块，server块，级别依此为：debug|info|notice|warn|error|crit|alert|emerg
events {
    accept_mutex on;   #设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;  #设置一个进程是否同时接受多个网络连接，默认为off
    #use epoll;      #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
    worker_connections  1024;    #最大连接数，默认为512
}
http {
    include       mime.types;   #文件扩展名与文件类型映射表
    default_type  application/octet-stream; #默认文件类型，默认为text/plain
    #access_log off; #取消服务日志    
    log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
    access_log log/access.log myFormat;  #combined为日志格式的默认值
    sendfile on;   #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。
    sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    keepalive_timeout 65;  #连接超时时间，默认为75s，可以在http，server，location块。

    upstream mysvr {   
      server 127.0.0.1:7878;
      server 192.168.10.121:3333 backup;  #热备
    }
    error_page 404 https://www.baidu.com; #错误页
    server {
        keepalive_requests 120; #单连接请求上限次数。
        listen       4545;   #监听端口
        server_name  127.0.0.1;   #监听地址       
        location  ~*^.+$ {       #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
           #root path;  #根目录
           #index vv.txt;  #设置默认页
           proxy_pass  http://mysvr;  #请求转向mysvr 定义的服务器列表
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip           
        } 
    }
}
```

### 配置文件地址

> Nginx配置文件为简化日常维护而设计，并且提供了简单的手段用于Web服务器将来的扩展。
>
> 配置文件是一些文本文件，通常位于 `nginx安装路径/etc/nginx或/etc/nginx`，主配置文件通常命名为 `nginx.conf`。
>
> 为保持整洁，部分配置可以放到单独的文件中，再自动地被包含在主配置文件，所以和Nginx行为相关的配置都应该在一个集中的配置文件目录中。

### Nginx的全局配置

```apl
user nobody nobody;
worker_processes 2;
error_log logs/error.log notice;
pid logs/nginx.pid;
 
events{
    use epoll;
    worker_connections 65536;
}
```

#### user

> 一个主模块指令，指定Nginx Worker进程运行用户以及用户组，默认由nobody账号运行。
>
> 这个地方如果写错，就会出现获取不到用户的错误。

![image-20210224172748967.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/image-20210224172748967.png)

#### worker_processes

> 一个主模块指令，指定Nginx要开启的进程数，每个Nginx进程平均耗费10M-20M的内存，建议指定和CPU数量一致即可。

![image-20210224173230858.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/image-20210224173230858.png)

#### error_log

> 主模块指令，用来定义全局错误日志文件，日志输出级别从低到高有`debug、info、notice、warn、error、crit`
>
> 日志文件路径一般在nginx安装目录的logs目录中

![image-20210224173452549.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/image-20210224173452549.png)

#### pid

> 一个主模块指令，用来指定进程pid的存储文件位置。

![image-20210224173701306.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/image-20210224173701306.png)

### events事件指令

> events事件指令是设定Nginx的工作模式以及连接数上限。

#### use

> 一个事件模块指令，用来指定Nginx的工作模式。
>
> Nginx支持的工作模式有`select、poll、kqueue、epoll、rtsig和/dev/poll`。
>
> 其中 select、poll 都是标准的工作模式，kqueue、poll 是高效的工作模式，不同的是epoll用在Linux，kqueue用在BSD系统
>
> 对于Linux系统，epoll工作模式是首选。

#### worker_connections

> 也是个事件模块指令，用于定义Nginx每个进程的最大连接数，默认是1024。
>
> 最大客户端连接数由 `worker_processes 和 worker_connections` 决定，即：
>
> `Max_client = worker_processes * worker_connections`
>
> 在作为反向代理时，max_clients变为 `Max_client = worker_processes * worker_connections / 4`
>
> 进程的最大连接数受Linux系统进程的最大打开文件数限制，在执行操作系统命令 `ulimit -n 65536`后，worker_connections的设置才能生效。

### HTTP服务器配置

> Nginx对HTTP服务器相关属性的配置如下：

```apl
http {
    # 引入文件类型映射文件
    include       mime.types;
    # 如果没有找到指定的文件类型映射 使用默认配置
    default_type  application/octet-stream;
    # 日志打印格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # 启动零拷贝提高性能
    sendfile        on;
    # 设置keepalive长连接超时时间
    keepalive_timeout  65;
    # 引入子配置文件
    include /usr/local/openresty/nginx/conf/conf.d/*.conf;
}
```

#### include

> 一个主模块指令，实现对配置文件所包含的文件的设定，可以减少主配置文件的复杂度。
>
> 可以将其他各个模块的具体配置分散在不同的文件夹中。

#### default_type

> HTTP核心模块指令，这里设定默认类型为二进制流，也就是当文件类型未定义时，使用这种方式。
>
> 例如在没有配置PHP环境时，Nginx是不予解析的，此时，用浏览器访问PHP文件就会出现下载窗口。

#### log_format

> HttpLog模块指令，用于指定Nginx日志的输出格式。main为此日志输出格式的名称，可以在下面的access_log指令中引用。

```apl
log_format main '$remote_addr - $remote_user [$time_local] '
'"$request" $status $bytes_sent '
'"$http_referer" "$http_user_agent" '
'"$gzip_ratio"';

log_format download '$remote_addr - $remote_user [$time_local] '
'"$request" $status $bytes_sent '
'"$http_referer" "$http_user_agent" '
'"$http_range" "$sent_http_content_range"';
```

## Nginx路由匹配

### 准备工作

> 整俩目录、俩index

### 编译安装echo

> 涉及到`echo`操作的地方需要安装echo模块，使用`--add-module`重新编译安装

```apl
./configure  --prefix=/usr/local/nginx --add-module=/usr/local/nginx/modules/echo-nginx-module-master

make && make install
```

### 虚拟主机

> 即Web服务器里一个独立的网站站点，该站点对应独立的域名（也可能是ip或port），具有独立程序及资源，可以独立对外提供服务。
>
> 在Nginx中，使用一个 server{ } 标签来标识一个虚拟主机，一个Web服务里可以有多个虚拟主机标签对。
>
> 虚拟主机有两种类型：
>
> 1. 基于域名的虚拟主机
> 2. 基于 ip + port 的虚拟主机

#### 完全匹配虚拟主机

> 在conf.d文件夹下创建 `vhostserver.conf`，配置文件内容如下:

```apl
server {
        listen       80;
        charset utf-8;
        server_name  www.abc.com;

        location /{
            alias '/root/www/nginx/abc/';
            index  index.html index.htm;
            expires  7d;
        }
}

server {
        listen       80 ;
        charset utf-8;
        server_name  www.bbs.com;


        location /{
            alias '/root/www/nginx/bbs/';
            index  index.html index.htm;
            expires  7d;
        }
}
```

#### 通配符配置虚拟主机

```apl
server {
        listen       80;
        charset utf-8;
        server_name  *.com;


        location /{
        default_type text/html;
         echo "通配符在前";
        }
}

server {
        listen       80;
        charset utf-8;
        server_name  www.abc.*;


        location /{
        default_type text/html;
         echo "通配符在后";
        }
}
```

#### 虚拟机主机匹配顺序

1. 完全匹配
2. 通配符在前
3. 通配符在后

#### 默认主机匹配

> 如果有多个访问的域名指向该Web服务器，但某个域名未添加到Nginx虚拟主机中，就会访问默认虚拟主机（泛解析）
>
> 如下：如果访问一个没有配置的域名或者直接通过ip地址访问，就会跳到bbs的页面。

```apl
server {
    # 将域名bbs.com 设置为默认虚拟主机
        listen       80 default;
        charset utf-8;
        server_name  www.bbs.com;


        location /{
            alias '/root/www/nginx/bbs/';
            index  index.html index.htm;
            expires  7d; 
        }
}
```

### location的作用

> 定位的意思，根据请求不同的URL来进行不同的处理，在虚拟主机中(server)，location配置是必不可少的。
>
> 可以把网站不同的部分定位到不同的处理方式上。

#### location规则

> location区段，通过指定模式来与客户端请求的URI相匹配。
>
> 允许根据用户请求的URL来匹配定义的各location，匹配到时，此请求将被响应的location配置块中的配置所处理。

##### 语法

```apl
location [修饰符] pattern {...}
```

##### 常见的修饰符说明

| 修饰符 | 功能                                                         |
| ------ | ------------------------------------------------------------ |
| 空     | 前缀匹配 能够匹配以需要匹配的路径为前缀的uri                 |
| =      | 精确匹配                                                     |
| ~      | 正则表达式模式匹配，区分大小写                               |
| ~*     | 正则表达式模式匹配，不区分大小写                             |
| ^~     | 精确前缀匹配，类似于无修饰符的行为，也是以指定模块开始，不同的是，如果模式匹配，那么就停止搜索其他模式了，不支持正则表达式 |
| /      | 通用匹配，任何请求都会匹配到。                               |

#### 前缀匹配

> 没有修饰符表示必须以指定模式开始，指定模式前面没有任何修饰符。
>
> 直接在location后面写需要匹配的uri，它的优先级次于正则匹配。

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location /abc {
        default_type text/html;
        echo "前缀匹配-abc...";
    }
}
```

如下内容会被匹配：

- www.itcast.com/abc
- www.itcast.com/abc/
- www.itcast.com/abc?

#### 通用匹配

> 通用匹配使用一个/表示，可以匹配所有请求，一般nginx配置文件最后都会有一个通用匹配规则。
>
> 当其他匹配规则均失效时，请求会被路由给通用匹配规则处理，如果也没有配置通用匹配，Nginx返回404错误。

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location /{
        default_type text/html;
        echo "通用匹配-default";
    }
}
```

#### 精准匹配

> 用=表示，具有最高优先级。

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location = /abc {
        default_type text/html;
        echo "精确匹配-abc-accurate";
    }
}
```

如下内容会被匹配：

- www.itcast.com/abc
- www.itcast.com/abc?

如下内容不会匹配：

- www.itcast.com/abc/
- www.itcast.com/abc/...

#### 精准前缀匹配

> 优先级仅次于精准匹配

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location ^~ /abc {
        default_type text/html;
        echo "精确前缀匹配-abc-prefix";
    }
}
```

如下内容会被匹配：

- www.itcast.com/abc
- www.itcast.com/abc/
- www.itcast.com/abc?

#### 正则表达式

> 正则匹配分为 区分大小写(~) 和 不区分大小写(~*) 两种。
>
> 优先级 < 精准前缀匹配 < 精准匹配。
>
> 正则匹配之间没有优先级之说，而是按照在配置文件中出现的顺序。

##### 区分大小写

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location ~ ^/abc$ {
          default_type text/html;
          echo "正则区分大小写-abc-regular-x";
    }
}
```

如下内容会被匹配：

- www.itcast.com/abc
- www.itcast.com/abc?

如下内容不会匹配：

- www.itcast.com/abc/
- www.itcast.com/ABC
- www.itcast.com/abcde

##### 不区分大小写

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location ~* ^/abc$ {
         default_type text/html;
         echo "正则不区分大小写-abc-regular-Y";
    }
}
```

如下内容会被匹配：

- www.itcast.com/abc
- www.itcast.com/abc?
- www.itcast.com/ABC

如下内容不会匹配：

- www.itcast.com/abc/
- www.itcast.com/abcde

#### 完整例子

```apl
server {
        server_name www.itcast.com;
        default_type text/html;
        charset   utf-8;
        location = / {
          echo "规则A";
        }
        location = /login {
            echo "规则B";
        }
        location ^~ /static/ {
            echo "规则C";
        }
        location ^~ /static/files {
            echo "规则X";
        }
        location ~ \.(gif|jpg|png|js|css)$ {
            echo "规则D";
        }
        location ~* \.js$ {
            echo "规则E";
        }
        location /img {
            echo "规则Y";
        }
        location / {
           echo "规则F";
        }
}
```

| 请求uri                                  | 匹配路由规则 |
| ---------------------------------------- | ------------ |
| http://www.itcast.com/                   | 规则A        |
| http://www.itcast.com/login              | 规则B        |
| http://www.itcast.com/register           | 规则F        |
| http://www.itcast.com/static/a.html      | 规则C        |
| http://www.itcast.com/static/files/a.txt | 规则X        |
| http://www.itcast.com/a.png              | 规则D        |
| http://www.itcast.com/a.PNG              | 规则F        |
| http://www.itcast.com/img/a.gif          | 规则D        |
| http://www.itcast.com/img/a.tiff         | 规则Y        |

#### 匹配顺序

> 匹配顺序和优先级，由高到低依次为：

1. 带有=的精确匹配
2. 正则表达式
3. 没有修饰符的精确匹配

> 注意：多个正则时，按在配置文件出现的顺序
>
> 具体匹配规则如下：

1. =精准匹配命中时，go
2. 一般匹配(含精准前缀匹配)命中时，先收集所有的普通匹配，对比出最长的那一条
   1. 如果最长的那条为精准前缀匹配，go
   2. 否则继续往下走正则location
3. 按代码顺序执行正则匹配，第一条正则命中，go

#### path匹配过程

![nginx26.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/nginx26.png)

> 假设http请求路径为 http://192.168.0.132:8088/mvc/index?id=2，匹配过程如下：

1. 将url拆解为 域名/端口/path/params
2. 先由域名/端口，对应到目标server虚拟主机
3. path部分参与location匹配，path = path1匹配部分 + path2剩余部分
4. 进入location方法内部流程
5. 若是静态文件处理，则进入目标目录查找文件
   1. root指令时查找 path1 + path2 对应的文件
   2. alias指定时查找 path2 对应的文件
6. 若是proxy代理，则：
   1. 形如 proxy_pass = ip:port 时，转发path1 + path2到tomcat
   2. 形如 proxy_pass = ip:port/xxx 时，转发path2到tomcat
   3. params始终跟随转发。

#### 实际使用建议

```apl
#直接匹配网站根，通过域名访问网站首页比较频繁，使用这个会加速处理，官网如是说。
#这里是直接转发给后端应用服务器了，也可以是一个静态首页
# 第一个必选规则
location = / {
    proxy_pass http://tomcat:8080/index
}
# 第二个必选规则是处理静态文件请求，这是nginx作为http服务器的强项
# 有两种配置模式，目录匹配或后缀匹配,任选其一或搭配使用
location ^~ /static/ {
    alias /webroot/static/;
}
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}
# 第三个规则就是通用规则，用来转发动态请求到后端应用服务器
# 非静态文件请求就默认是动态请求，自己根据实际把握
# 毕竟目前的一些框架的流行，带.php,.jsp后缀的情况很少了
location / {
    proxy_pass http://tomcat:8080/
}
```

## Nginx负载均衡

> 负载均衡用于从 `upstream` 模块定义的后端服务器列表中选取一台服务器接受用户的请求。
>
> 一个最基本的upstream模块是这样的，模块内的server是服务器列表：

```apl
#动态服务器组
upstream dynamicserver {
  server 172.16.44.47:9001; #tomcat 1
  server 172.16.44.47:9002; #tomcat 2
  server 172.16.44.47:9003; #tomcat 3
  server 172.16.44.47:9004; #tomcat 4
}
```

> 在upstream模块配置完成后，要让指定的访问反向代理到服务器列表：

```apl
#其他页面反向代理到tomcat容器
location ~.*$ {
  index index.jsp index.html;
  proxy_pass http://dynamicserver;
}
```

这就是最简单的负载均衡示例，但不足以满足实际需求。

目前Nginx服务器的upstream模块支持6种方式的分配。

> 完整配置如下：

```apl
upstream dynamicserver {
  server 192.168.64.1:9001; #tomcat 1
  server 192.168.64.1:9002; #tomcat 2
  server 192.168.64.1:9003; #tomcat 3
  server 192.168.64.1:9004; #tomcat 4
}
server {
        server_name www.itcast.com;
        default_type text/html;
        charset   utf-8;

        location ~ .*$ {
            index index.jsp index.html;
            proxy_pass http://dynamicserver;
       }
}
```

### 常用参数

| 参数         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| server       | 反向服务地址加端口                                           |
| weight       | 权重                                                         |
| fail_timeout | 与max_fails结合使用。                                        |
| max_fails    | 设置在fail_timeout参数设置的时间内最大失败次数，如果在这个时间内，所有针对该服务器的请求都失败了，那么认为该服务器会被认为是停机了 |
| max_conns    | 允许最大连接数                                               |
| fail_time    | 服务器会被认为停机的时间长度,默认为10s                       |
| backup       | 标记该服务器为备用服务器，当主服务器停止时，请求会被发送到它这里。 |
| down         | 标记服务器永久停机了                                         |
| slow_start   | 当节点恢复，不立即加入                                       |

### 负载均衡策略

> 这里只写Nginx自带的负载均衡策略，第三方不做描述。

| 负载策略           | 描述            |
| ------------------ | --------------- |
| 轮询               | 默认方式        |
| weight             | 权重方式        |
| ip_hash            | 依据ip分配方式  |
| least_conn         | 最少连接方式    |
| fair（第三方）     | 响应时间方式    |
| url_hash（第三方） | 依据URL分配方式 |

#### 轮询

> 轮询时，如果服务器宕机，会自动剔除该服务器。
>
> 适合服务器配置相当，无状态且短平快的服务使用。

#### 权重

> weight参数用于指定轮询几率，默认值为1.
>
> 权重越高分配到需要处理的请求越多
>
> 此策略可以与 least_conn 和 ip_hash 结合使用
>
> 此策略比较适合服务器的硬件配置差别比较大的情况

```apl
#动态服务器组
upstream dynamicserver {
  server 192.168.64.1:9001  weight=2;                   #tomcat 1
  server 192.168.64.1:9002;                             #tomcat 2
  server 192.168.64.1:9003;                         #tomcat 3
  server 192.168.64.1:9004; #tomcat 4
}
```

#### ip_hash

> 按照基于客户端ip的分配方式。确保相同客户端请求一直发送到相同的服务器，以保证session会话。
>
> 每个访客都固定访问一个后端服务器，可以用来解决session不能跨服务器的问题。
>
> 注意：
>
> - Nginx1.3.1之前，不能在ip_hash中使用权重
> - ip_hash不能与backup同时使用
> - 此策略适合有状态服务，比如session
> - 当有服务器需要剔除，必须手动down掉

```apl
upstream dynamicserver {
  ip_hash;  #保证每个访客固定访问一个后端服务器
  server 192.168.64.1:9001  weight=2;                   #tomcat 1
  server 192.168.64.1:9002;                             #tomcat 2
  server 192.168.64.1:9003;                         #tomcat 3
  server 192.168.64.1:9004; #tomcat 4
}
```

#### least_conn

> 把请求发给连接数较少的后端服务器。简而言之，谁有空谁干活。
>
> 适合请求处理时间长短不一致，造成服务器过载的情况。

```apl
upstream dynamicserver {
  least_conn;  #把请求转发给连接数较少的后端服务器
  server 192.168.64.1:9001  weight=2;                   #tomcat 1
  server 192.168.64.1:9002;                             #tomcat 2
  server 192.168.64.1:9003;                         #tomcat 3
  server 192.168.64.1:9004; #tomcat 4
}
```

### 重试策略

> 现在对外服务的网站，很少只使用一个服务节点，而是部署多台服务器，上层通过一定机制保证容错和负载均衡。

#### 基础失败重试

> 为了方便理解，使用了以下配置进行分析（proxy_next_upstream 没有特殊配置）

##### 重试配置

```apl
upstream dynamicserver {
      server 192.168.64.1:9001 fail_timeout=60s max_fails=3; #Server A
      server 192.168.64.1:9002 fail_timeout=60s max_fails=3; #Server B
}
```

##### 配置说明

> `max_fials=3 fail_timeout=60s` 代表在60s内请求某一应用失败3次，就认为该应用宕机
>
> 后等待60s，这期间内不会再把新请求发送到宕机应用，而是直接发送到正常的服务器
>
> 时间到后，再有请求进来仅需尝试连接宕机应用，且仅尝试1次，如果还是失败，则继续等待60s
>
> 依次循环，直到恢复

##### 模拟异常

> 模拟后端异常可以直接将服务关闭，造成 connect refused 的情况，对应error错误

开始AB都正常，轮询处理请求。假设此时A挂了，则出现如下情况：

1. request1 -> A 处理异常 -> B 正常处理，A fails + 1
2. request2 -> B 正常处理
3. request3 -> A 处理异常 -> B 正常处理，A fails + 1，达到max_fails，将被屏蔽60s
4. 屏蔽A期间，所有请求 -> B，直至屏蔽到期后，A恢复服务重新加入存活列表。

> 如果A屏蔽期还没结束，B也挂了，则会出现：

1. request1 -> B 异常，此时所有线上节点都异常，就
   - AB节点一次性恢复，都重新加入存活列表
   - request -> A 异常 -> B 异常，触发 no live upstreams 报错，返回 502 错误
   - 所有节点再次一次性恢复，加入存活列表

#### 错误重试

> 有时系统出现500等error的情况下，希望nginx能到其他服务进行重试，可以通过配置那些错误码进行重试。

##### 配置说明

> 在nginx的配置文件中，`proxy_next_upstream`项定义了什么情况下进行重试，官网文档中给出的说明如下：

```apl
Syntax: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | off ...;
Default:    proxy_next_upstream error timeout;
Context:    http, server, location
```

> 默认情况下，当请求服务器发生错误或超时时，会尝试到下一台服务器，还有一些其他的配置项如下：

| 错误状态       | 描述                                                         |
| -------------- | :----------------------------------------------------------- |
| error          | 与服务器建立连接，向其传递请求或读取响应头时发生错误;        |
| timeout        | 在与服务器建立连接，向其传递请求或读取响应头时发生超时;      |
| invalid_header | 服务器返回空的或无效的响应;                                  |
| http_500       | 服务器返回代码为500的响应;                                   |
| http_502       | 服务器返回代码为502的响应;                                   |
| http_503       | 服务器返回代码为503的响应;                                   |
| http_504       | 服务器返回代码504的响应;                                     |
| http_403       | 服务器返回代码为403的响应;                                   |
| http_404       | 服务器返回代码为404的响应;                                   |
| http_429       | 服务器返回代码为429的响应（1.11.13）;                        |
| non_idempotent | 通常，请求与 非幂等 方法（POST，LOCK，PATCH）不传递到请求是否已被发送到上游服务器（1.9.13）的下一个服务器; 启用此选项显式允许重试此类请求; |
| off            | 禁用将请求传递给下一个服务器。                               |

##### 配置说明

> 配置500等错误的时候进行重试

```apl
upstream dynamicserver {
  server 192.168.64.1:9001 fail_timeout=60s max_fails=3; #tomcat 1
  server 192.168.64.1:9002 fail_timeout=60s max_fails=3; #tomcat 2
}

server {
        server_name www.itcast.com;
        default_type text/html;
        charset   utf-8;

        location ~ .*$ {
            index index.jsp index.html;
            proxy_pass http://dynamicserver;
            #下一节点重试的错误状态
            proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
       }
}
```

##### 模拟异常

> 正常情况下，500会直接出现异常页面，现在加入了500等状态码的重试策略，重试错误的流程和上面的流程就是一样的了。

#### backup服务器

> Nginx支持设置备用节点，当所有线上节点都异常时启用备用节点，同时备用节点也会影响到失败重试的逻辑。

##### backup处理逻辑

> upstream的配置中，可以通过 `backup` 来定义备用服务器，其含义如下：

1. 正常情况下，请求不会转发到backup服务器，包括失败重试的场景
2. 当所有正常节点全部不可用时，backup服务器生效，开始处理请求
3. 一旦有正常节点恢复，就使用已经恢复的正常节点
4. backup服务器生效期间，**不会存在所有正常节点一次性恢复的逻辑**
5. 如果全部backup服务器也异常，则会将所有节点一次性恢复，加入存活列表
6. 如果全部节点，包括backup服务器，都异常了，则Nginx返回502错误

##### 配置说明

```apl
upstream dynamicserver {
  server 192.168.64.1:9001 fail_timeout=60s max_fails=3; #Service A
  server 192.168.64.1:9002 fail_timeout=60s max_fails=3; #Server B
  server 192.168.64.1:9003 backup; #backup
}

server {
        server_name www.itcast.com;
        default_type text/html;
        charset   utf-8;

        location ~ .*$ {
            index index.jsp index.html;
            proxy_pass http://dynamicserver;
            #下一节点重试的错误状态
            proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
       }
}
```

#### 限制重试方法

> 默认配置是没有做重试机制限制的，即尽可能去重试直至失败。
>
> Nginx提供了以下两个参数来控制重试次数以及重试超时时间。

- `proxy_next_upstream_tries`：设置重试次数，默认0表示无限制，该参数包含所有请求upstream server次数，包括第一次之后所有重试之和。
- `proxy_next_upstream_timeout`：设置重试最大超时时间，默认0表示不限制，指第一次连接时间加上后续重试连接时间，不包含连接上节点之后的处理时间。

##### 配置说明

```apl
upstream dynamicserver {
      server 192.168.64.1:9001 fail_timeout=60s max_fails=3; #Server A
      server 192.168.64.1:9002 fail_timeout=60s max_fails=3; #Server B
}

server {
        server_name www.itcast.com;
        default_type text/html;
        charset   utf-8;

        location ~ .*$ {
            index index.jsp index.html;
            proxy_pass http://dynamicserver;
            # 表示重试超时时间是3s
            proxy_connect_timeout 3s;
            #表示在 6 秒内允许重试 3 次，只要超过其中任意一个设置，Nginx 会结束重试并返回客户端响应
            proxy_next_upstream_timeout 6s;
            proxy_next_upstream_tries 3;
       }
}
```

## Nginx常用案例

### 代理静态文件

> 访问 http://localhost:10086/data/ 下面的资源就是访问 /usr/local/data 文件夹的资源

```apl
server {
        listen       10086;
        server_name  www.itcast.com;
    
        location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_redirect off;
        }
        
        location /data/ {
            alias '/usr/local/data/'; 
            #这里是重点，就是代理这个文件夹 
            expires    7d;
        }
}
```

### 反向代理

```apl
server {
    listen       80;
    server_name  www.itcast.com;;

    location / {
        proxy_pass http://127.0.0.1:8080;
        index  index.html index.htm .jsp;
    }
}
```

### 跨域配置

```apl
server {
        listen       80;
        server_name  www.itcast.com;

    if ( $host ~ (.*).itcast.com){
        set $domain $1;##记录二级域名值
    }
    #是否允许请求带有验证信息
    add_header Access-Control-Allow-Credentials true;
    #允许跨域访问的域名,可以是一个域的列表，也可以是通配符*
    add_header Access-Control-Allow-Origin  *;
    #允许脚本访问的返回头
    add_header Access-Control-Allow-Headers 'x-requested-with,content-type,Cache-Control,Pragma,Date,x-timestamp';
    #允许使用的请求方法，以逗号隔开
    add_header Access-Control-Allow-Methods 'POST,GET,OPTIONS,PUT,DELETE';
    #允许自定义的头部，以逗号隔开,大小写不敏感
    add_header Access-Control-Expose-Headers 'WWW-Authenticate,Server-Authorization';
    #P3P支持跨域cookie操作
    add_header P3P 'policyref="/w3c/p3p.xml", CP="NOI DSP PSAa OUR BUS IND ONL UNI COM NAV INT LOC"';
    if ($request_method = 'OPTIONS') {##OPTIONS类的请求，是跨域先验请求
            return 204;##204代表ok
     }
}
```

### 防盗链

> 通过Referer实现防盗链比较基础，尽可以简单实现方式资源被盗用，构造Referer的请求很容易实现
>
> 场景：由于图片链接可以跨域访问，所以图片链接往往被其他网站盗用，从而增加服务器负担
>
> 解决方案：Nginx可以通过valid_referers配置进行防盗链配置

### valid_referers指令

> 指定合法的来源 `referer`，决定了内置变量 `$invalid_referer` 的值
>
> 如果referer头部包含在这个合法网址里面，这个变量被设置为0，否则设置为1
>
> 注意：这里并不区分大小写。

- 语法：valid_referers none | blocked | server_names | string...;
- 配置端：server, locaiton

#### 配置说明

- none：允许没有http_refer的请求访问资源
- blocked：允许不是http://开头的，不带协议的请求访问资源
- 192.168.0.1：只允许指定ip来的请求访问资源
- *.google.com：允许通配符域名请求访问资源

#### 配置代码

```apl
# 需要防盗的后缀
location ~* \.(jpg|jpeg|png|gif|bmp|swf|rar|zip|doc|xls|pdf|gz|bz2|mp3|mp4|flv)$
    #设置过期时间
    expires     30d;
    # valid_referers 就是白名单的意思
    # 支持域名或ip
    # 允许ip 192.168.0.1 的请求
    # 允许域名 *.google.com 所有子域名
    valid_referers none blocked 192.168.0.1 *.google.com;
    if ($invalid_referer) {
        # return 403;
        # 盗链返回的图片，替换盗链网站所有盗链的图片
        rewrite ^/ https://site.com/403.jpg;
    }
    root  /usr/share/nginx/img;
}
```