# 黄康 - Docker 安装 ES

## 单机版

```shell
# 拉取镜像
docker pull docker.io/elasticsearch:6.7.0

# 运行容器 设置 JVM 参数
docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 --name es docker.io/elasticsearch:6.7.0
```

## 生产版

```shell
# 创建挂载文件
mkdir -p /docker/elasticsearch/data
chmod -R 777 /docker/elasticsearch/data

# 启动容器
docker run -d \
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-p 10092:9200 \
-p 10093:9300 \
-v /docker/elasticsearch/data:/usr/share/elasticsearch/data \
--name es6.7 docker.io/elasticsearch:6.7.0

# 如果内存足够的话
docker run -d \
-e ES_JAVA_OPTS="-Xms1g -Xmx1g" \
-p 9200:9200 \
-p 9300:9300 \
-v /docker/elasticsearch/data:/usr/share/elasticsearch/data \
--name es6.7 docker.io/elasticsearch:6.7.0
```

## 集群版

```shell
# 创建目录、文件
mkdir /root/elasticsearch-cluster-yml
touch /root/elasticsearch-cluster-yml/es-cluster{1,2,3}.yml

# 编写配置文件，三个文件分别修改 node.name[集群节点的名字]，不能相同
# network.publish_host[集群节点的 ip]
# discovery.zen.ping.unicast.hosts[集群ip，集群节点互相通信，在一台机器上虚拟出三个端口]
vim es-cluster1.yml

echo "cluster.name: es-cluster
node.name: es-master1
network.bind_host: 0.0.0.0
network.publish_host: 118.187.4.89
http.port: 9201
transport.tcp.port: 9301
http.cors.enabled: true
http.cors.allow-origin: \"*\"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts: [\"118.187.4.89:9302\",\"118.187.4.89:9303\"]
discovery.zen.minimum_master_nodes: 1" > /root/elasticsearch-cluster-yml/es-cluster1.yml

vim es-cluster2.yml

echo "cluster.name: es-cluster
node.name: es-data1
network.bind_host: 0.0.0.0
network.publish_host: 118.187.4.89
http.port: 9202
transport.tcp.port: 9302
http.cors.enabled: true
http.cors.allow-origin: \"*\"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts: [\"118.187.4.89:9301\",\"118.187.4.89:9303\"]
discovery.zen.minimum_master_nodes: 1" > /root/elasticsearch-cluster-yml/es-cluster2.yml

vim es-cluster3.yml

echo "cluster.name: es-cluster
node.name: es-data2
network.bind_host: 0.0.0.0
network.publish_host: 118.187.4.89
http.port: 9203
transport.tcp.port: 9303
http.cors.enabled: true
http.cors.allow-origin: \"*\"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts: [\"118.187.4.89:9301\",\"118.187.4.89:9302\"]
discovery.zen.minimum_master_nodes: 1" > /root/elasticsearch-cluster-yml/es-cluster3.yml
```

```shell
# 挂载数据文件夹
mkdir /root/elasticsearch-cluster-data
mkdir /root/elasticsearch-cluster-data/data{1,2,3}
chmod -R 777 /root/elasticsearch-cluster-data

# 启动容器
docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d \
-p 9201:9201 \
-p 9301:9301 \
-v /root/elasticsearch-cluster-yml/es-cluster1.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /root/elasticsearch-cluster-data/data1:/usr/share/elasticsearch/data --name es-master1 \
docker.io/elasticsearch:6.7.0

docker run -d \
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-p 9202:9202 \
-p 9302:9302 \
-v /root/elasticsearch-cluster-yml/es-cluster2.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /root/elasticsearch-cluster-data/data2:/usr/share/elasticsearch/data --name es-data1 \
docker.io/elasticsearch:6.7.0

docker run -d \
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-p 9203:9203 \
-p 9303:9303 \
-v /root/elasticsearch-cluster-yml/es-cluster3.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /root/elasticsearch-cluster-data/data3:/usr/share/elasticsearch/data --name es-data2 \
docker.io/elasticsearch:6.7.0
```

## Kibana 安装

```shell
# 下载 es 对应 kibana 版本镜像
docker pull docker.io/kibana:6.7.0

# 编写配置文件
mkdir -p /docker/kibana/conf
vim /docker/kibana/conf/kibana.yml

# 添加以下内容 host
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://111.67.196.127:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true

# 启动容器
docker run -d \
--name kibana6.7 \
-p 5601:5601 \
-v /docker/kibana/conf/kibana.yml:/usr/share/kibana/config/kibana.yml \
-e ELASTICSEARCH_URL=http://118.187.4.89:10092/ docker.io/kibana:6.7.0 
```

## IK 分词器安装

```shell
# 进入容器内部编辑
docker exec -it es6.7 bash

# 安装 IK 分词器插件
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.7.0/elasticsearch-analysis-ik-6.7.0.zip

# 下载完之后检测
cd plugins/
ls | grep ik

# 如果失败
docker logs -f es6.7 # 查看文件
# 如果是权限的原因
vim /etc/sysctl.conf
# 在最后一行添加
vm.max_map_count=655360
# 保存退出后执行
sysctl -p
```

