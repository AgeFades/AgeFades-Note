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
worker_processes  2;

error_log  /usr/local/nginx/logs/error.log  notice;

pid        /usr/local/nginx/logs/nginx.pid;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    server_names_hash_bucket_size   128;
    client_header_buffer_size       128k;
    large_client_header_buffers 8   128k;
    client_max_body_size    200m;
    client_body_buffer_size 128k;
    proxy_connect_timeout   600;
    proxy_read_timeout  600;
    proxy_send_timeout  600;
    proxy_buffer_size   16k;
    proxy_buffers   4   32k;
    proxy_busy_buffers_size 64k;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"' '"$upstream_addr"' '"$upstream_response_time"';

    access_log  /usr/local/nginx/logs/access.log  main;

    sendfile        on;
    tcp_nopush     on;
    tcp_nodelay    on;

    keepalive_timeout  300s 300s;
    keepalive_requests 8192;
    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_buffer_size 64k;
    fastcgi_buffers 4 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;

    gzip  on;
    gzip_min_length     1k;
    gzip_buffers        4 16k;
    gzip_comp_level     8;
    gzip_http_version   1.1;
    gzip_types      text/plain  application/xml;
    gzip_vary       on;
    #隐藏版本号
    server_tokens off;

   # To allow special characters in headers
   ignore_invalid_headers off;
   # Allow any size file to be uploaded.
   # Set to a value such as 1000m; to restrict file size to a specific value
#   client_max_body_size 1000m;
   # To disable buffering
   proxy_buffering off;

   include conf.d/*.conf;

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
    restart: always
    ports:
      - 80:80
      - 8080:8080
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
    hostname: nginx
    networks:
      jdy:
        ipv4_address: 173.18.0.13
networks:
  jdy:
    external: true
```

### 简单部署脚本

```shell
vim deploy.sh
```

```shell
# deploy.sh 内容
sudo docker-compose -f docker-compose-nginx.yaml down
sudo docker-compose -f docker-compose-nginx.yaml up -d
```

### 执行安装

```shell
sh deploy.sh
```

