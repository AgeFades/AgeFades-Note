## 安装

- **前提**：创建Docker网络
- 版本：最新

```shell
# 自己找个目录存放这些文件地址

# 创建数据挂载目录、日志挂载目录、配置挂载目录
mkdir data logs config

# 切换目录
cd config

# 创建nginx子配置目录、证书目录
mkdir conf.d ssl

# 切换目录
cd ../

# 创建配置文件挂载目录
touch nginx.conf

# 创建 docker-compose 脚本
touch docker-compose-nginx.yaml
```

### nginx.conf

```shell
# vim nginx.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

### default.conf

```shell
vim config/conf.d/deafult.conf
```

```apl
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    
}
```

### docker-compose脚本

```yml
# vim docker-compose-nginx.yaml
version: '3.4'
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    hostname: nginx
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      # 日志
      - ./logs:/usr/local/nginx/logs
      # https 证书
      - ./config/ssl:/usr/local/nginx/ssl
      # 数据
      - ./data:/usr/local/nginx/data
      # 配置文件
      - ./nginx.conf:/etc/nginx/nginx.conf
      # 子配置文件
      - ./config/conf.d/:/etc/nginx/conf.d
```

### 简单部署脚本

```shell
vim deploy.sh
```

```shell
# deploy.sh 内容
docker-compose -f docker-compose-nginx.yaml down
docker-compose -f docker-compose-nginx.yaml up -d
```

### 执行安装

```shell
sh deploy.sh
```

