# 黄康 - Docker 安装MySQL

```shell
# 创建挂载目录
mkdir -p /docker/mysql8/conf
mkdir -p /docker/mysql8/data
```

```shell
# 修改字符编码
vim /docker/mysql8/conf/my.cnf

[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

```shell
# 启动容器
docker run -p 13308:3306 \
--name mysql8 \
-e MYSQL_ROOT_PASSWORD=dc980816 \
--privileged=true \
-v /docker/mysql8/data:/var/lib/mysql \
-v /docker/mysql8/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:8.0.16
```

安装 mysql8 和上述一致，切换 mysql版本就好 docker.io/mysql:8.0.16