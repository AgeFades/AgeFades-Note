## 最简单的反向代理配置

```apl
server {
        listen       80;
        server_name  www.agefades.com;

        location / {
            proxy_pass http://127.0.0.1:10080;
            index  index.html index.htm index.jsp;
        }
    }
```

## 最简单的Https反向代理配置

```apl
server {
    listen 80;
    server_name hub.agefades.com;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
   listen       443;
   server_name  hub.agefades.com;
   ssl on;
   ssl_certificate /usr/local/nginx/ssl/hub.agefades.com/fullchain.cer;
   ssl_certificate_key /usr/local/nginx/ssl/hub.agefades.com/hub.agefades.com.key;
   index index.html index.htm index.php index.jsp;
   charset utf-8;


   location / {
      proxy_pass          http://172.17.0.1:8880;
      proxy_redirect      default;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    Range $http_range;
      proxy_set_header X-Forwarded-Proto  $scheme;
      client_max_body_size 100m;    
      client_body_buffer_size 128k;  
      proxy_connect_timeout 600; 
      proxy_send_timeout 600;        
      proxy_read_timeout 600;         
      proxy_buffer_size 16k;             
      proxy_buffers 4 32k;               
      proxy_busy_buffers_size 64k;    
      proxy_temp_file_write_size 64k;  
   }
 
}
```



## 操作案例

```shell
# 在 nginx 的 conf.d 目录下
vim gateway.conf
```

```shell
server{
    listen 80;
    server_name dev.admin-api.jdy.iyobee.com;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
   listen       443;
   server_name  dev.admin-api.jdy.iyobee.com;
   ssl on;
   ssl_certificate /usr/local/nginx/ssl/dev.admin-api.jdy.iyobee.com/fullchain.cer;
   ssl_certificate_key /usr/local/nginx/ssl/dev.admin-api.jdy.iyobee.com/dev.admin-api.jdy.iyobee.com.key;
   index index.html index.htm index.php index.jsp;
   charset utf-8;


   location /admin/ {
      proxy_pass          http://admin;
      proxy_redirect      default;
      #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    Range $http_range;
      proxy_set_header X-Forwarded-Proto  $scheme;
      client_max_body_size 100m;    #允许客户端请求的最大单文件字节数
      client_body_buffer_size 128k;  #缓冲区代理缓冲用户端请求的最大字节数，
      proxy_connect_timeout 600;  #nginx跟后端服务器连接超时时间(代理连接超时)
      proxy_send_timeout 600;        #后端服务器数据回传时间(代理发送超时)
      proxy_read_timeout 600;         #连接成功后，后端服务器响应时间(代理接收超时)
      proxy_buffer_size 16k;             #设置代理服务器（nginx）保存用户头信息的缓冲区大小
      proxy_buffers 4 32k;               #proxy_buffers缓冲区，网页平均在32k以下的，这样设置
      proxy_busy_buffers_size 64k;    #高负荷下缓冲大小（proxy_buffers*2）
      proxy_temp_file_write_size 64k;  #设定缓存文件夹大小，大于这个值，nginx会先将文件写入“proxy_temp_path ”缓存目录
   }

   include /etc/nginx/conf.d/*.location;
 
}

upstream admin {
    server 172.17.0.1:8001 weight=1 max_fails=2 fail_timeout=30s;
    keepalive 768;
}

include /etc/nginx/conf.d/*.upstream;
```

### 重新加载 nginx

```shell
# 在 sudo su 权限下，否则加 sudo
docker exec -it nginx nginx -s reload
```

## 同域名不同后缀操作案例

```shell
# 仍然在 nginx 的 conf.d 目录下
vim nacos.location
```

```shell
location /nacos/ {
      proxy_pass          http://nacos;
      proxy_redirect      default;
      #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    Range $http_range;
      proxy_set_header X-Forwarded-Proto  $scheme;
      client_max_body_size 100m;    #允许客户端请求的最大单文件字节数
      client_body_buffer_size 128k;  #缓冲区代理缓冲用户端请求的最大字节数，
      proxy_connect_timeout 600;  #nginx跟后端服务器连接超时时间(代理连接超时)
      proxy_send_timeout 600;        #后端服务器数据回传时间(代理发送超时)
      proxy_read_timeout 600;         #连接成功后，后端服务器响应时间(代理接收超时)
      proxy_buffer_size 16k;             #设置代理服务器（nginx）保存用户头信息的缓冲区大小
      proxy_buffers 4 32k;               #proxy_buffers缓冲区，网页平均在32k以下的内存大小，这样设置
      proxy_busy_buffers_size 64k;    #高负荷下缓冲大小（proxy_buffers*2）
      proxy_temp_file_write_size 64k;  #设定缓存文件夹大小，大于这个值，nginx会先将文件写入“proxy_temp_path ”缓存目录
   }
```

```shell
vim nacos.upstream
```

```shell
upstream nacos {
    server nacos:8848 weight=1 max_fails=2 fail_timeout=30s;
    keepalive 768;
}
```

## 前端案例

```shell
"mechanism-admin.conf" 22L, 622C                                                                                                                                                              17,50         All
server{
    listen 80;
    server_name dev.admin.jdy.iyobee.com;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
   listen       443;
   server_name  dev.admin.jdy.iyobee.com;
   ssl on;
   ssl_certificate /usr/local/nginx/ssl/dev.admin.jdy.iyobee.com/fullchain.cer;
   ssl_certificate_key /usr/local/nginx/ssl/dev.admin.jdy.iyobee.com/dev.admin.jdy.iyobee.com.key;
   index index.html index.htm index.php index.jsp;
   charset utf-8;

   location / {
      root   /usr/local/nginx/data/mechanism-admin;
      index  index.html index.htm;   #访问index
      try_files $uri $uri/ /index.html;
   }

}
```

