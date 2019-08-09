# 黄康 - Mongo 数据迁移

## 挂载目录

```shell
docker ps | grep mongo # 查询 mongo 容器
docker inspect id[name] # 通过容器 id or name 查询容器具体信息，找到挂载目录

# 如果未挂载目录
docker cp id[name]:/data/db /docker/mongo/data # 将数据备份出来
```

## 数据迁移

```shell
# 进入 mongo 的 bin 目录下 
# 导出数据
mongodump -h 127.0.0.1 -port 27017 -u root -p root -d my-db -o /tmp/mongo

# 导入数据
mongorestore -h 127.0.0.1 -port 27017 -u root -p root -d my-db --dir /tmp/mongo

# 带条件导出数据
mongoexport -h 192.168.86.108 --port 27017 -d my-db -c my-collection -f id -q '{"id":{"$exists":true}}'  --csv -o 123

-h ： 主机
--port ：端口号
-d ：数据库名
-c ：表名
-o ：输出的文件名
--type ： 输出的格式，默认为json
-f ：输出的字段，如果-type为csv，则需要加上-f "字段名"
-q ：输出查询条件

# 导入
mongoimport -h 192.168.86.126 --port 27017 -d my-db -c my-collection --type csv --headerline -f _id --file 123
```

