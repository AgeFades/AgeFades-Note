## 安装

- **前提**：创建Docker网络
- 版本：8.0.20

```shell
# 自己找个目录存放这些文件地址

# 创建数据挂载目录、配置文件挂载目录
mkdir data config

# 创建 docker-compose 脚本
touch docker-compose-mysql.yaml
```

### MySQL配置文件

```shell
# 创建 my.cnf 文件
vim config/my.cnf
```

```shell
# my.cnf 文件内容
[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

### docker-compose脚本

```yaml
# vim docker-compose-mysql.yaml
# 注意复制过去可能产生的 格式缩进 问题
version: '3.4'
services:
  mysql:
    container_name: mysql
    hostname: mysql
    image: mysql:8.0.20
    restart: always
    ports:
      - "13309:3306"
    environment:
      MYSQL_ROOT_PASSWORD: -tt!EYuyqfs~
      MYSQL_ROOT_HOST: "%"
    volumes:
      - ./data:/var/lib/mysql
      - ./config:/etc/mysql/conf.d
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M
      --default-time-zone='+8:00'
```

### 简单部署脚本

```shell
vim deploy.sh
```

```shell
# deploy.sh 内容
docker-compose -f docker-compose-mysql.yaml down
docker-compose -f docker-compose-mysql.yaml up -d
```

### 执行安装

```shell
sh deploy.sh
```

