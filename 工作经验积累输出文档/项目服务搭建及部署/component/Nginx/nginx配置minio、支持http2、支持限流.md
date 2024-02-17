[TOC]

# nginx配置minio、支持http2、支持限流

[http2参考链接](https://zhuanlan.zhihu.com/p/223304805#:~:text=%E5%8F%AA%E9%9C%80%E5%B0%86http2%E5%8F%82%E6%95%B0%E6%B7%BB%E5%8A%A0%E5%88%B0%E6%89%80%E6%9C%89LISTEN%E6%8C%87%E4%BB%A4%EF%BC%8C%E5%8D%B3%E5%8F%AF%E5%90%AF%E7%94%A8HTTP%2F2%E6%94%AF%E6%8C%81%EF%BC%8C%E5%A6%82%E4%B8%8B%E9%9D%A2%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%E6%89%80%E7%A4%BA%E3%80%82%20listen%20443,ssl%20http2%3B%20%E7%A4%BA%E4%BE%8B%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%9D%97%E9%85%8D%E7%BD%AE%E5%A6%82%E4%B8%8B%E6%89%80%E7%A4%BA%E3%80%82)

[限流配置参考链接](https://zhuanlan.zhihu.com/p/130237695)

[图像裁剪服务thumbor参考链接](https://blog.xiaoz.org/archives/17804)

## 实际配置案例

```apl
server {
    listen 80;
    server_name minio.abc.com;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
   listen [::]:443 ssl ipv6only=on http2;
   listen 443 ssl http2; # managed by Certbot
   server_name  minio.abc.com;
   ssl_certificate xxx/fullchain.cer;
   ssl_certificate_key xxx/minio.abc.com.key;
   index index.html index.htm index.php index.jsp;
   charset utf-8;

   location / {
      proxy_pass          http://127.0.0.1:9001;
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
			proxy_http_version      1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
			#限制每ip每秒不超过20个请求，漏桶数burst为5
      #brust的意思就是，如果第1秒、2,3,4秒请求为19个，
      #第5秒的请求为25个是被允许的。
      #但是如果你第1秒就25个请求，第2秒超过20的请求返回503错误。
      #nodelay，如果不设置该选项，严格使用平均速率限制请求数，
      #第1秒25个请求时，5个请求放到第2秒执行，
      #设置nodelay，25个请求将在第1秒执行。
      limit_req zone=allips burst=100 nodelay;
   }
location /view/ {
      proxy_pass          http://127.0.0.1:19000/;
      proxy_redirect      default;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For 		      	   $proxy_add_x_forwarded_for;
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
      proxy_http_version      1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
 	    #限制每ip每秒不超过20个请求，漏桶数burst为5
      #brust的意思就是，如果第1秒、2,3,4秒请求为19个，
      #第5秒的请求为25个是被允许的。
      #但是如果你第1秒就25个请求，第2秒超过20的请求返回503错误。
      #nodelay，如果不设置该选项，严格使用平均速率限制请求数，
      #第1秒25个请求时，5个请求放到第2秒执行，
      #设置nodelay，25个请求将在第1秒执行。
      limit_req zone=allips burst=100 nodelay;
}
location /thumbor/ {
      proxy_pass          http://127.0.0.1:18888/;
      proxy_redirect      default;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
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
      proxy_http_version      1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
			#限制每ip每秒不超过20个请求，漏桶数burst为5
      #brust的意思就是，如果第1秒、2,3,4秒请求为19个，
      #第5秒的请求为25个是被允许的。
      #但是如果你第1秒就25个请求，第2秒超过20的请求返回503错误。
      #nodelay，如果不设置该选项，严格使用平均速率限制请求数，
      #第1秒25个请求时，5个请求放到第2秒执行，
      #设置nodelay，25个请求将在第1秒执行。
      limit_req zone=allips burst=100 nodelay;
  }
}
```

