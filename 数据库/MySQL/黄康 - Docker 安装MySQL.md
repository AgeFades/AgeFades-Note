# 黄康 - Docker 安装MySQL

```shell
# 创建挂载目录
mkdir -p /docker/mysql/conf
mkdir -p /docker/mysql/data
```

```shell
# 修改字符编码
vim /docker/mysql/conf/my.cnf

[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

```shell
# 启动容器
docker run -p 13307:3306 \
--name mysql \
-e MYSQL_ROOT_PASSWORD=agefades \
--privileged=true \
-v /docker/mysql/data:/var/lib/mysql \
-v /docker/mysql/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7
```

安装 mysql8 和上述一致，切换 mysql版本就好 docker.io/mysql:8.0.16