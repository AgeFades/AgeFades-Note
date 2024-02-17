[TOC]

# 黑马 - OpenResty

## OpenResty介绍

> 一个基于Nginx与Lua的高性能Web平台，内部集成了大量精良的Lua库、第三方模块以及大多数的依赖项

用于方便地搭建能够处理超高并发、扩展性极高的动态Web应用、Web服务和动态网关。

OpenResty的目标是：让Web服务直接跑在Nginx内部，利用其非阻塞I/O模型，不仅是对HTTP客户端请求，甚至于对远程后端诸如 MySQL、PostgreSQL、Memcached、Redis等都进行一致的高性能响应。

### Nginx的流程定义

> Nginx将请求处理流程划分为11个阶段，将请求执行逻辑细分，各阶段按照处理时机定义了清晰的执行语义。
>
> 开发者可以很容易分辨自己需要开发的模块应该定义在什么阶段。

![nginx28.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/nginx28.png)

1. 当请求进入Nginx后，先 read request headers 读取头部，然后再分配由哪个指令操作
2. Identity 寻找匹配哪个 Location*
3. Apply Rate Limits 是否要对该请求限制
4. Preform Authertication 权限验证
5. Generate Content 生成给用户的响应内容
6. 如果配置了反向代理，那么将要和上游服务器通信 Upstream Services
7. 当返回给用户请求的时候要经过过滤模块 Response Filter
8. 发送给用户的同时，记录一个Log日志

#### 流程详解

| 阶段           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| post-read      | 接收到完整的http头部后处理的阶段，在uri重写之前，一般跳过    |
| server-rewrite | location匹配前，修改uri的阶段，用于重定向，location块外的重写指令（多次执行） |
| find-config    | uri寻找匹配的location块配置项（多次执行）                    |
| rewrite        | 找到location块后再修改uri，location级别的uri重写阶段（多次执行） |
| post-rewrite   | 防死循环，跳转到对应阶段                                     |
| preaccess      | 权限预处理                                                   |
| access         | 判断是否允许这个请求进入                                     |
| post-access    | 向用户发送拒绝服务的错误码，用来响应上一阶段的拒绝           |
| try-files      | 访问静态文件资源                                             |
| content        | 内容生成阶段，该阶段产生响应，并发送到客户端                 |
| log            | 记录访问日志                                                 |

#### OpenResty处理流程

![nginx31.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/nginx31.png)

| 指令                          | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| init_by_lua,init_by_lua_block | 运行在Nginx loading-config 阶段，注册Nginx Lua全局变量，和一些预加载模块。是Nginx master进程在加载Nginx配置时执行 |
| init_worker_by_lua            | 在Nginx starting-worker阶段，即每个nginx worker启动时会调用，通常用来hook worker进程，并创建worker进行的计时器，用来健康检查，或者设置熔断记时窗口等等。 |
| access_by_lua                 | 在access tail阶段，用来对每次请求做访问控制，权限校验等等，能拿到很多相关变量。例如：请求体中的值，header中的值，可以将值添加到ngx.ctx, 在其他模块进行相应的控制 |
| balancer_by_lua               | 通过Lua设置不同的负载均衡策略, 具体可以参考[lua-resty-balancer](https://link.zhihu.com/?target=https://github.com/openresty/lua-resty-balancer) |
| content_by_lua                | 在content阶段，即content handler的角色，即对于每个api请求进行处理，注意不能与proxy_pass放在同一个location下 |
| proxy_pass                    | 真正发送请求的一部分, 通常介于access_by_lua和log_by_lua之间  |
| header_filter_by_lua          | 在output-header-filter阶段，通常用来重新响应头部，设置cookie等，也可以用来作熔断触发标记 |
| body_filter_by_lua            | 对于响应体的content进行过滤处理                              |
| log_by_lua                    | 记录日志即，记录一下整个请求的耗时，状态码等                 |

## OpenResty安装

### yum安装

> 简单记录下yum安装方式，其余安装网上搜搜资料即可

```bash
yum install openresty
```

### 验证

```shell
nginx -v
```

![nginx66.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/nginx66.png)

### 环境配置

#### 配置文件修改

```apl
user  root;
worker_processes  2;

error_log  logs/error.log  info;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    
    include conf.d/*.conf;
}
```

#### 创建配置目录

```shell
mkdir /usr/local/openresty/nginx/conf/conf.d
```

#### 创建Nginx配置文件

> 在conf.d目录中创建一个测试的配置文件
>
> 这里将html代码直接写在了配置文件中。

```apl
server {
    server_name www.itcast.com;
    charset   utf-8;
    location /{
        default_type text/html;
        content_by_lua '
            ngx.say("<p>Hello, World!</p>")
            ';
    }
}
```

#### 启动OpenResty

```apl
nginx -c /usr/local/openresty/nginx/conf/nginx.conf
```

#### 测试

```apl
curl http://127.0.0.1/
<p>Hello, World!</p>
```

## OpenResty常用命令

> 主要帮助对http请求取参、取header头、输出等

| 命令                       | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| ngx.arg                    | 指令参数，如跟在content_by_lua_file后面的参数                |
| ngx.var                    | request变量，ngx.var.VARIABLE引用某个变量,lua使用nginx内置的绑定变量. `ngx.var.remote_addr`为获取远程的地址， |
| ngx.ctx                    | 请求的lua上下文,每次请求的上下文，可以在ctx里记录，每次请求上下文的一些信息，例如：`request_id`, `access_key`等等 |
| ngx.header                 | 响应头，ngx.header.HEADER引用某个头                          |
| ngx.status                 | 响应码                                                       |
| ngx.log                    | 输出到error.log                                              |
| ngx.send_headers           | 发送响应头                                                   |
| ngx.headers_sent           | 响应头是否已发送                                             |
| ngx.resp.get_headers       | 获取响应头                                                   |
| ngx.is_subrequest          | 当前请求是否是子请求                                         |
| ngx.location.capture       | 发布一个子请求                                               |
| ngx.location.capture_multi | 发布多个子请求                                               |
| ngx.print                  | 输出响应                                                     |
| ngx.say                    | 输出响应，自动添加‘\n‘                                       |
| ngx.flush                  | 刷新响应                                                     |
| ngx.exit                   | 结束请求                                                     |

### 商品详情页系统架构介绍

> 商品详情页架构的要求，高可用，高性能，高并发 ；一般来说 业界分为两种主流的方案

#### 中小公司的详情页方案

> 很多中小型 电商的商品详情页 可能一分钟都没有一个访问，这种的话，就谈不上并发设计，一个tomcat 就能搞定

​		还有一种中小型公司呢？虽然说公司不大，但是也是有几十万日活，然后几百万用户，他们的商品详情用，采取的方案可能是全局的一个静态页面这样子

![1700127836693.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700127836693.png)

​		就是我们有把商品详情页直接做成一个静态页面，然后这样子每次全量的更新，把数据全部静态放到`redis`里面，每次数据变化的时候，我们就通过一个Java服务去渲染这个数据，然后把这个静态页面推送到到文件服务器

##### 缺点

- 这种方案的缺点，如果商品很多，那么渲染的时间会很长，达不到实时的效果
- 文件服务器性能高，tomcat性能差，压力都在Tomcat服务器了
- 只能处理一些静态的东西，如果动态数据很多，比如有库存的，你不可能说每次去渲染，然后推送到文件服务器，那不是更加慢？

#### 大型公司的商品详情页的核心思想

> 下面这张图就是整体的思路

![1700127857259.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700127857259.png)

> 上图展示了核心思想主要有以下五步来完成

##### 生成静态页

​		添加修改页面的时候生成静态页，这个地方生成的是一个通用的静态页，敏感数据比如 价格，商品名称等，通过占位符来替换，然后将生成的静态页的链接，以及敏感数据同步到redis中，如果只修改价格不需要重新生成静态页，只需要修改redis敏感数据即可。

##### 推送到文件服务器

​		这个的文件服务器泛指能够提供静态文件处理的文件服务器，nginx代理静态文件，tomcat，以及OSS等都算静态文件服务器，生成完静态文件后将文件推送到文件服务器，并将请求连接存放进redis中

##### 布隆过滤器过滤请求

​		Redis和nginx的速度很快，但是如果有人恶意请求不存在的请求会造成redis很大的开销，那么可以采用布隆过滤器将不存在的请求过滤出去。

##### lua直连Redis读取数据

​		因为java连接Reids进行操作并发性能很弱，相对于OpenResty来说性能差距很大，这里使用OpenResty，读取Redis中存放的URL以及敏感数据。

##### OpenResty 渲染数据

​		从Redis获取到URL后lua脚本抓取模板页面内容，然后通过redis里面的敏感数据进行渲染然后返回前端，因为都是lua脚本操作性能会很高

#### 环境准备

> 下面我们就基于这个架构来安装和搭建所需要的环境

##### 配置文件服务器

> 我们的的文件服务器页面在`nginx-server`的代码中可以通过`http://IP/template.html`访问

##### 配置资源反向代理

> 通过nginx来配置资源服务器

```apl
upstream dynamicserver {
      server 192.168.64.1:9001 fail_timeout=60s max_fails=3;
      server 192.168.64.1:9002 fail_timeout=60s max_fails=3;
      keepalive 256;
}

server {
        server_name www.resources.com 127.0.0.1;
        default_type text/html;
        charset   utf-8;
        location ~ .*$ {
            index index.jsp index.html;
            proxy_pass http://dynamicserver;
            # 表示重试超时时间是3s
            proxy_connect_timeout      30;   
            proxy_send_timeout         10;    
            proxy_read_timeout         10; 
            #表示在 6 秒内允许重试 3 次，只要超过其中任意一个设置，Nginx 会结束重试并返回客户端响应
            proxy_next_upstream_timeout 60s;
            proxy_next_upstream_tries 3;

       }
}
```

##### 访问测试

> 可以通过访问`www.resources.com/template.html`访问测试

![1700127891799.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700127891799.png)

#### Redis安装布隆过滤器

##### 什么是布隆过滤器

> 使用布隆过滤器来解决缓存穿透问题

###### 布隆过滤器的原理

​		布隆过滤器其本质就是一个只包含0和1的数组，具体操作当一个元素被加入到集合里面后，该元素通过K个Hash函数运算得到K个hash后的值，然后将K个值映射到这个位数组对应的位置，把对应位置的值设置为1。查询是否存在时，我们就看对应的映射点位置如果全是1，他就很可能存在（跟hash函数的个数和hash函数的设计有关），如果有一个位置是0，那这个元素就一定不存在。

![1700127905793.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700127905793.png)

###### 布隆过滤器缺点

> bloom filter之所以能做到在时间和空间上的效率比较高，是因为牺牲了判断的准确率、删除的便利性

- 存在误判，可能要查到的元素并没有在容器中，但是hash之后得到的k个位置上值都是1，如果bloom filter中存储的是黑名单，那么可以通过建立一个白名单来存储可能会误判的元素
- 删除困难，一个放入容器的元素映射到bit数组的k个位置上是1，删除的时候不能简单的直接置为0，可能会影响其他元素的判断，可以采用`Counting Bloom Filter`

##### 插件形式安装

###### 下载布隆过滤器插件

> 在[redis布隆过滤器插件地址](https://codeload.github.com/RedisBloom/RedisBloom/tar.gz/v2.2.5)下载最新的release源码，在编译服务器进行解压编译

```shell
 wget https://github.com/RedisBloom/RedisBloom/archive/v2.2.4.tar.gz
```

> 解压插件进行插件的编译

```shell
tar RedisBloom-2.2.4.tar.gz
cd RedisBloom-2.2.4
make
```

编译得到动态库`rebloom.so` 启动redis时，如下启动即可加载bloom filter插件

> 配置文件形式配置

```shell
#在redis配置文件(redis.conf)中加入该模块即可
vim redis.conf
#添加
loadmodule /root/bloom/redisbloom-2.2.4/rebloom.so （前面为你自己的路径）
```

> 启动命令挂载

```shell
redis-server redis.conf --loadmodule /usr/rebloom/rebloom.so INITIAL_SIZE 1000000   ERROR_RATE 0.0001   
#容量100万, 容错率万分之一, 占用空间是4m
```

##### docker方式单机安装

```shell
docker run -d -p 6379:6379 --name redis-redisbloom redislabs/rebloom:latest
```

##### Redis集群部署安装

###### 创建目录

```shell
mkdir -p /tmp/etc/redis/
mkdir -p /tmp/data/redis/node{1..6}
```

###### redis配置文件

> 将配置文件写入到redis文件中

```apl
cat > /tmp/etc/redis/redis.conf<< EOF
protected-mode no
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""
databases 1
always-show-logo yes
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
rdb-del-sync-files no
dir ./
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-diskless-load disabled
repl-disable-tcp-nodelay no
replica-priority 100
acllog-max-len 128
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
lazyfree-lazy-user-del no
oom-score-adj no
oom-score-adj-values 0 200 800
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
jemalloc-bg-thread yes
loadmodule /etc/redis/redisbloom.so
EOF
```

 

> 这里将本地编译好的redisbloom.so挂载到/etc/redis/redisbloom.so目录

###### 配置docker-compose.yml文件

```yaml
version: '2'
services:
    redis01:
        image: redis
        hostname: redis01
        container_name: redis01
        networks:
            docker-network:
                ipv4_address: 172.18.0.2
        ports:
            - "7001:6379"
        volumes:
            - "/tmp/etc/redis/redis.conf:/etc/redis/redis.conf"
            - "/tmp/data/redis/node1:/data"
            - "/tmp/etc/redis/redisbloom.so:/etc/redis/redisbloom.so"
        command:
           redis-server /etc/redis/redis.conf
    redis02:
        image: redis
        hostname: redis02
        container_name: redis02
        networks:
            docker-network:
                ipv4_address: 172.18.0.3
        ports:
            - "7002:6379"
        volumes:
            - "/tmp/etc/redis/redis.conf:/etc/redis/redis.conf"
            - "/tmp/data/redis/node2:/data"
            - "/tmp/etc/redis/redisbloom.so:/etc/redis/redisbloom.so"
        command:
           redis-server /etc/redis/redis.conf
    redis03:
        image: redis
        hostname: redis03
        container_name: redis03
        networks:
            docker-network:
                ipv4_address: 172.18.0.4
        ports:
            - "7003:6379"
        volumes:
            - "/tmp/etc/redis/redis.conf:/etc/redis/redis.conf"
            - "/tmp/data/redis/node3:/data"
            - "/tmp/etc/redis/redisbloom.so:/etc/redis/redisbloom.so"
        command:
            redis-server /etc/redis/redis.conf
    redis04:
        image: redis
        hostname: redis04
        container_name: redis04
        networks:
            docker-network:
                ipv4_address: 172.18.0.5
        ports:
            - "7004:6379"
        volumes:
            - "/tmp/etc/redis/redis.conf:/etc/redis/redis.conf"
            - "/tmp/data/redis/node4:/data"
            - "/tmp/etc/redis/redisbloom.so:/etc/redis/redisbloom.so"
        command:
            redis-server /etc/redis/redis.conf
    redis05:
        image: redis
        hostname: redis05
        container_name: redis05
        networks:
            docker-network:
                ipv4_address: 172.18.0.6
        ports:
            - "7005:6379"
        volumes:
            - "/tmp/etc/redis/redis.conf:/etc/redis/redis.conf"
            - "/tmp/data/redis/node5:/data"
            - "/tmp/etc/redis/redisbloom.so:/etc/redis/redisbloom.so"
        command:
            redis-server /etc/redis/redis.conf
    redis06:
        image: redis
        hostname: redis06
        container_name: redis06
        networks:
            docker-network:
                ipv4_address: 172.18.0.7
        ports:
            - "7006:6379"
        volumes:
            - "/tmp/etc/redis/redis.conf:/etc/redis/redis.conf"
            - "/tmp/data/redis/node6:/data"
            - "/tmp/etc/redis/redisbloom.so:/etc/redis/redisbloom.so"
        command:
            redis-server /etc/redis/redis.conf

networks:
    docker-network:
        ipam:
            config:
                - subnet: 172.18.0.0/16
                  gateway: 172.18.0.1
```

###### 启动布隆过滤器集群

```apl
docker-compose up -d
```

###### 配置集群

> 进入一个容器

```apl
docker exec -ti redis01 /bin/bash
```

> 执行创建集群命令

```apl
redis-cli --cluster create 172.18.0.2:6379 172.18.0.3:6379 172.18.0.4:6379 172.18.0.5:6379 172.18.0.6:6379 172.18.0.7:6379 --cluster-replicas 1
```

###### 布隆过滤器常用命令

| 命令         | 功能                                                         | 参数                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BF.RESERVE   | 创建一个大小为capacity，错误率为error_rate的空的Bloom        | BF.RESERVE {key} {error_rate} {capacity} [EXPANSION expansion] [NONSCALING] |
| BF.ADD       | 向key指定的Bloom中添加一个元素item                           | BF.ADD {key} {item}                                          |
| BF.MADD      | 向key指定的Bloom中添加多个元素                               | BF.MADD {key} {item} [item...]                               |
| BF.INSERT    | 向key指定的Bloom中添加多个元素，添加时可以指定大小和错误率，且可以控制在Bloom不存在的时候是否自动创建 | BF.INSERT {key} [CAPACITY {cap}] [ERROR {error}] [EXPANSION expansion] [NOCREATE] [NONSCALING] ITEMS {item...} |
| BF.EXISTS    | 检查一个元素是否可能存在于key指定的Bloom中                   | BF.EXISTS {key} {item}                                       |
| BF.MEXISTS   | 同时检查多个元素是否可能存在于key指定的Bloom中               | BF.MEXISTS {key} {item} [item...]                            |
| BF.SCANDUMP  | 对Bloom进行增量持久化操作                                    | BF.SCANDUMP {key} {iter}                                     |
| BF.LOADCHUNK | 加载SCANDUMP持久化的Bloom数据                                | BF.LOADCHUNK {key} {iter} {data}                             |
| BF.INFO      | 查询key指定的Bloom的信息                                     | BF.INFO {key}                                                |
| BF.DEBUG     | 查看BloomFilter的内部详细信息（如每层的元素个数、错误率等）  | BF.DEBUG {key}                                               |

###### 测试

> 可以登录集群执行命令

```apl
# 进入一个redis集群的节点内部
docker exec -ti redis01 /bin/bash
# 以集群方式登录172.18.0.2:3306节点
redis-cli -h 172.18.0.2 -c
# 在redis中添加一个布隆过滤器 错误率是0.01 数量是1万个
BF.RESERVE bf_test 0.01 10000 NONSCALING
# 在bf_test 的布隆过滤器添加一个key
BF.ADD bf_test key
# 验证布隆过滤器key是否存在
BF.EXISTS bf_test key
# 验证布隆过滤器key1是否存在
BF.EXISTS bf_test key1
```

#### OpenResty支持Reids集群配置

##### 下载安装lua_resty_redis

> lua_resty_redis 它是一个基于**rockspec** API的为ngx_lua模块提供Lua redis客户端的驱动。
>
> resty-redis-cluster模块地址：https://github.com/steve0511/resty-redis-cluster

- 将`resty-redis-cluster/lib/resty/`下面的文件 拷贝到 `openresty/lualib/resty`

总共两个文件`rediscluster.lua`,`xmodem.lua`

##### 连接Redis集群封装

> 创建`redisUtils.lua`的文件

```lua
--操作Redis集群，封装成一个模块
--引入依赖库
local redis_cluster = require "resty.rediscluster"


--配置Redis集群链接信息
local config = {
    name = "testCluster",                   --rediscluster name
    serv_list = {                           --redis cluster node list(host and port),
                   {ip="192.168.245.164", port = 7001},
                   {ip="192.168.245.164", port = 7002},
                   {ip="192.168.245.164", port = 7003},
                   {ip="192.168.245.164", port = 7004},
                   {ip="192.168.245.164", port = 7005},
                   {ip="192.168.245.164", port = 7006},
    },
    keepalive_timeout = 60000,              --redis connection pool idle timeout
    keepalive_cons = 1000,                  --redis connection pool size
    connection_timout = 1000,               --timeout while connecting
    max_redirection = 5
}


--定义一个对象
local lredis = {}

--创建查询数据get()
function lredis.get(key)
        local red = redis_cluster:new(config)
        local res, err = red:get(key)
        if err then
            ngx.log(ngx.ERR,"执行get错误:",err)
            return false
        end
        return res
end

-- 执行hgetall方法并封装成table
function lredis.hgetall(hash_key)
    local red = redis_cluster:new(config)
    local flat_map, err = red:hgetall(hash_key)
    if err then
        ngx.log(ngx.ERR,"执行hgetall错误:",err)
        return false
    end
    local result = {}
    for i = 1, #flat_map, 2 do
        result[flat_map[i]] = flat_map[i + 1]
    end
    return result
end

-- 判断key中的item是否在布隆过滤器中
function lredis.bfexists(key,item)
             local red = redis_cluster:new(config)
             -- 通过eval执行脚本
             local res,err = red:eval([[
             local  key=KEYS[1]
             local  val= ARGV[1]
             local  res,err=redis.call('bf.exists',key,val)
             return res
                ]],1,key,item)
     if  err then
        ngx.log(ngx.ERR,"过滤错误:",err)
        return false
     end
     return res
end

return lredis
```

##### 配置lua脚本路径

> 我们编写了lua脚本需要交给nginx服务器去执行，我们需要将创建一个lua目录，并在全局的nginx‘配置文件中配置lua目录，配置参数使用`lua_package_path`

```apl
# 配置redis 本地缓存
lua_shared_dict redis_cluster_slot_locks 100k;
# 配置lua脚本路径
lua_package_path "/usr/local/openresty/script/?.lua;;"; #注意后面是两个分号
```

> 完整的配置如下

```apl

user  root;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
# 注意！！ error日志输出info级别日志
error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;
    # 注意！！配置redis 本地缓存
    lua_shared_dict redis_cluster_slot_locks 100k;
    # 注意！！lua脚本路径
    lua_package_path "/usr/local/openresty/script/?.lua;;";
    #gzip  on;
    include     conf.d/*.conf;
}
```

##### 测试脚本

> 在 nginx配置文件嵌入lua脚本进行测试

```apl
server {
        listen 9999;
        charset utf-8;
    
        location /test {
            default_type text/html;
            content_by_lua '
               local lrredis = require("redisUtils")
               -- 尝试读取redis中key的值
               local value = lrredis.get("key")
               --判断key是否在bf_taxi的布隆过滤器中
               local bfexist = lrredis.bfexists("bf_taxi","key")
               local htest = lrredis.hgetall("h_taxi")
               ngx.log(ngx.INFO, "key的值是",value)
               ngx.log(ngx.INFO, "bf_taxi布隆过滤器key的状态",bfexist)
               ngx.log(ngx.INFO, "h_taxi[url]的值是",htest["url"])
            ';
        }
    }
```

###### 设置Redis数据

```apl
# 登录Redis
docker exec -ti redis01 /bin/bash
redis-cli -h 172.18.0.2 -c
# 设置key
set key value11
# 设置hash
hset h_taxi url http://www.baidu.com
# 创建布隆过滤器
BF.RESERVE bf_taxi 0.01 10000 NONSCALING
# 布隆过滤器添加key
BF.ADD bf_taxi key
```

###### 访问测试

> 访问查看nginx日志打印

```apl
tail -f /usr/local/openresty/nginx/logs/error.log
```

![1700128061202.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128061202.png)

#### 请求参数封装

> nginx为了能够处理请求需要获取请求数据，需要将获取请求参数的lua脚本封装以下，创建`requestUtils.lua`的文件

```lua
--定义一个对象
local lreqparm={}
-- 获取请求参数的方法
function lreqparm.getRequestParam()
    -- 获取请求方法 get或post
   local request_method = ngx.var.request_method
    -- 定义参数变量
    local args = nil
    if "GET" == request_method then
        args = ngx.req.get_uri_args()
    elseif "POST" == request_method then
        ngx.req.read_body()
        args = ngx.req.get_post_args()
    end
    return args
end
return lreqparm
```

##### 测试脚本

```apl
server {
        listen 9999;
        charset utf-8;

        location /testreq {
            default_type text/html;
            content_by_lua '
              local lreqparm = require("requestUtils")
              local params = lreqparm.getRequestParam()
              local title = params["title"]
              if title ~= nil then
                  ngx.say("<p>请求参数的Title是：</p>"..title)
                return 
              end
              ngx.say("<P>没有输入title请求参数<P>")
            ';
        }
    }
```

> 访问http://192.168.64.150:9999/testreq?title=ceshi 测试

![1700128097139.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128097139.png)

#### 抓取模板内容封装

##### 下载安装lua-resty-http

> 下载地址 https://github.com/ledgetech/lua-resty-http

将`lua-resty-http-master\lib\resty`下的所有文件复制到`openresty/lualib/resty` httpclient

总共两个文件`http.lua`,`http_headers.lua`

> 因为需要从远程服务器抓取远程页面的内容，需要用到http模块，将其封装起来，创建`requestHtml.lua`

```lua
-- 引入Http库
local http = require "resty.http"
--定义一个对象
local lgethtml={}

function lgethtml.gethtml(requesturl)

    --创建客户端
    local httpc = http:new()
    local resp,err = httpc:request_uri(requesturl,
        {
                method = "GET",
                headers = {["User-Agent"]="Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/40.0.2214.111 Safari/537.36"}
        })
    --关闭连接
    httpc:close()
    if not resp then
        ngx.say("request error:",err)
        return
    end
    local result = {}
    --获取状态码
    result.status = resp.status
    result.body  = resp.body
    return result

end

return lgethtml

```

##### 测试脚本

> 模板文件我们用

```apl
server {
        listen 9999;
        charset utf-8;

        # 配置路径重写
        location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
            rewrite ^/(.*) http://www.resources.com/$1 permanent;
        }
    
        location /testgetHtml {
            default_type text/html;
            content_by_lua '
                local lgethtml = require("requestHtml")
                local url = "http://127.0.0.1/template.html"
                local result = lgethtml.gethtml(url);
                ngx.log(ngx.INFO, "状态是",result.status)
                ngx.log(ngx.INFO, "body是",result.body)
                ngx.say(result.body)
            ';
        }
    }
```

> 访问http://192.168.64.181:9999/testgetHtml 测试

![1700128158382.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128158382.png) 

#### 模版渲染配置

##### 下载安装lua-resty-template

```apl
wget https://github.com/bungle/lua-resty-template/archive/v1.9.tar.gz
tar -xvzf v1.9.tar.gz
```

​		解压后可以看到lib/resty下面有一个template.lua，这个就是我们所需要的，在template目录中还有两个lua文件，将这两个文件复制到/usr/openResty/lualib/resty中即可。

##### 使用方式

```lua
local template = require "resty.template"
-- Using template.new
local html=[[<html>
<body>
  <h1>{{message}}</h1>
</body>
</html>
]]
template.render(html, { message = params["title"] })
```

##### 测试

> 在nginx中配置文件中嵌入lua脚本执行测试

```apl
server {
        listen 9999;
        charset utf-8;

         location /testtemplate {
            default_type text/html;
            content_by_lua '
              local lreqparm = require("requestUtils")
              local template = require "resty.template"
              local params = lreqparm.getRequestParam()
               -- Using template.new
               local html=[[<html>
                             <body>
                                <h1>{{message}}</h1>
                             </body>
                         </html>
                                ]]
                template.render(html, { message = params["title"] })
            ';
        }

    }
```

> 访问`http://192.168.64.150:9999/testtemplate?title=123456`

![1700128191559.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128191559.png)

### 整体业务分析

> 整个调用流程如下，需要使用lua脚本编写，整体流程分为7 步

![1700128208241.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128208241.png)

> 上面我们将各个组件都给封装了，下面我们需要将各个组件组合起来

#### 编写lua脚本

> 创建一个`requestTemplateRendering.lua`的lua脚本

```lua
---
--- Generated by EmmyLua(https://github.com/EmmyLua)
--- Created by baiyp.
--- DateTime: 2020/11/24 13:24
---


local template = require("resty.template")
local lrredis = require("redisUtils")
local lgethtml = require("requestHtml")
local lreqparm = require("requestUtils")
--获取请求参数
local reqParams = lreqparm.getRequestParam()
-- 定义本地缓存
local html_template_cache = ngx.shared.html_template_cache

-- 获取请求ID的参数
local reqId = reqParams["id"];

ngx.log(ngx.INFO, "requestID:", reqId);
-- 校验参数
if reqId==nil then
    ngx.say("缺少ID参数");
    return
end

-- 布隆过滤器检查id是否存在
local bfexist = lrredis.bfexists("bf_taxi",reqId)
ngx.log(ngx.INFO, "布隆过滤器检验:", bfexist)

-- 校验key不存在直接返回
if bfexist==0 then
    ngx.say("布隆过滤器校验key不存在...")
    return
end

-- 拼接hget的key
local hgetkey = "hkey_".. reqId

-- 通过hget获取map的所有数据
local templateData = lrredis.hgetall(hgetkey);
if next(templateData) == nil then
    ngx.say("redis没有存储数据...")
    return
end


--获取模板价格数据
local amount = templateData["amount"]
ngx.log(ngx.INFO, "amount:", amount)
if amount == nil then
    ngx.say("价格数据未配置");
    return
end


-- 获取本地缓存对象
ngx.log(ngx.INFO, "开始从缓存中获取模板数据----")
local html_template = html_template_cache:get(reqId)
-- 判断本地缓存是否存在
if html_template == nil then
    -- 获取模板url中的数据
        ngx.log(ngx.INFO, "缓存中不存在数据开始远程获取模板")
    local url = templateData["url"]
        ngx.log(ngx.INFO, "从缓存中获取的url地址:", url)
    if url == nil then
        ngx.say("URL路径未配置");
        return
    end
        -- 抓取远程url的html
        ngx.log(ngx.INFO, "开始抓取模板数据:", url)
    local returnResult = lgethtml.gethtml(url);
    -- 判断抓取模板是否正常
    if returnResult==nil then
        ngx.say("抓取URL失败...");
        return
    end
        -- 判断html状态
    if returnResult.status==200 then
        html_template = returnResult.body
                --设置模板缓存为一小时
                ngx.log(ngx.INFO, "将模板数据加入到本地缓存")
        html_template_cache:set(reqId,html_template,60 * 60)
    end
end
ngx.log(ngx.INFO, "缓存中获取模板数据结束----")

-- 模板渲染
--编译得到一个lua函数

local func = template.compile(html_template)
local context = {amount=amount}
ngx.log(ngx.INFO, "开始渲染模板数据")
--执行函数，得到渲染之后的内容
local content = func(context)

--通过ngx API输出
ngx.say(content)

```

#### 编写nginx配置

##### 添加本地缓存

> 在nginx主配置文件中添加模板缓存

```apl
lua_shared_dict html_template_cache 10m;
```

##### 编写nginx配置文件

> 我们这里使用`content_by_lua_file`的方式执行脚本

```apl
vi template.conf
server {
        listen 8888;
        charset utf-8;
        # 配置路径重写
        location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
            rewrite ^/(.*) http://www.resources.com/$1 permanent;
        }
    
        #删除本地缓存
        location /delete {
             default_type text/html;
             content_by_lua '
              local lreqparm = require("requestUtils")
              --获取请求参数
              local reqParams = lreqparm.getRequestParam()
               -- 定义本地缓存
              local html_template_cache = ngx.shared.html_template_cache

                -- 获取请求ID的参数
              local reqId = reqParams["id"];

              ngx.log(ngx.INFO, "requestID:", reqId);
              -- 校验参数
              if reqId==nil then
                  ngx.say("缺少ID参数");
                  return
               end
               -- 获取本地缓存对象
               html_template_cache:delete(reqId);
               ngx.say("清除缓存成功");
            ';
        }

        location /template {
            default_type text/html;
            content_by_lua_file /usr/local/openresty/script/requestTemplateRendering.lua;
        }
}
```

#### 初始化数据

```apl
# 进入一个redis集群的节点内部
docker exec -ti redis01 /bin/bash
# 以集群方式登录172.18.0.2:3306节点
redis-cli -h 172.18.0.2 -c
# 在redis中添加一个布隆过滤器 错误率是0.01 数量是1万个
BF.RESERVE bf_taxi 0.01 10000 NONSCALING
# 在bf_test 的布隆过滤器添加一个key
BF.ADD bf_taxi 1
#检查数据是否存在
BF.EXISTS bf_taxi 1
# 添加URL以及价格
hset hkey_1 url http://127.0.0.1/template.html amount 100.00
```

#### 访问测试

> 访问`http://192.168.64.181:8888/template?id=1`进行测试

### kong网关(自学)

#### kong网关简介

> Kong是一款基于OpenResty（Nginx + Lua模块）编写的高可用、易扩展的，由Mashape公司开源的API Gateway项目

​		Kong是基于NGINX和Apache Cassandra或PostgreSQL构建的，能提供易于使用的RESTful API来操作和配置API管理系统，所以它可以水平扩展多个Kong服务器，通过前置的负载均衡配置把请求均匀地分发到各个Server，来应对大批量的网络请求。

#### 为什么需要 API 网关

> API网关是一个服务器，是系统的唯一入口。从面向对象设计的角度看，它与外观模式类似。API网关封装了系统内部架构，为每个客户端提供一个定制的API。它可能还具有其它职责，如身份验证、监控、负载均衡、缓存、请求分片与管理、静态响应处理。API网关方式的核心要点是，所有的客户端和消费端都通过统一的网关接入微服务，在网关层处理所有的非业务功能。通常，网关也是提供REST/HTTP的访问API。

![1700128263872.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128263872.png)

​		在微服务架构之下，服务被拆的非常零散，降低了耦合度的同时也给服务的统一管理增加了难度。如上图左所示，在旧的服务治理体系之下，鉴权，限流，日志，监控等通用功能需要在每个服务中单独实现，这使得系统维护者没有一个全局的视图来统一管理这些功能。API 网关致力于解决的问题便是为微服务纳管这些通用的功能，在此基础上提高系统的可扩展性。如右图所示，微服务搭配上 API 网关，可以使得服务本身更专注于自己的领域，很好地对服务调用者和服务提供者做了隔离。

​		目前，比较流行的网关有：Nginx 、 Kong 、Orange等等，还有微服务网关Zuul 、Spring Cloud Gateway等等

​		对于 API Gateway，常见的选型有基于 Openresty 的 Kong、基于 Go 的 Tyk 和基于 Java 的 gateway，这三个选型本身没有什么明显的区别，主要还是看技术栈是否能满足快速应用和二次开发。

##### 和Spring Cloud Gateway区别

- 像Nginx这类网关，性能肯定是没得说，它适合做那种门户网关，是作为整个全局的网关，是对外的，处于最外层的；而Gateway这种，更像是业务网关，主要用来对应不同的客户端提供服务的，用于聚合业务的。各个微服务独立部署，职责单一，对外提供服务的时候需要有一个东西把业务聚合起来。
- 像Nginx这类网关，都是用不同的语言编写的，不易于扩展；而Gateway就不同，它是用Java写的，易于扩展和维护
- Gateway这类网关可以实现熔断、重试等功能，这是Nginx不具备的

> 所以，你看到的网关可能是这样的

![1700128283732.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128283732.png)

#### 为什么要使用kong

- 插件市场丰富，很多插件可以降低开发成本；
- 可扩展性，可以编写lua脚本来定制自己的参数验证权限验证等操作；
- 基于openResty，openResty基于Nginx保障了强劲的性能；
- 便捷性能扩容，只需要水平增加服务器资源性能就能提升 ；
- 负载均衡健康检查

##### kong的组成部分

> Kong主要有三个组件

- Kong Server ：基于nginx的服务器，用来接收API请求。
- Apache Cassandra/PostgreSQL ：用来存储操作数据。
- Kong dashboard：官方推荐UI管理工具，当然，也可以使用 restfull 方式 管理admin api

​	Kong采用插件机制进行功能定制，插件集（可以是0或N个）在API请求响应循环的生命周期中被执行。插件使用Lua编写，目前已有几个基础功能：HTTP基本认证、密钥认证、CORS（Cross-Origin Resource Sharing，跨域资源共享）、TCP、UDP、文件日志、API请求限流、请求转发以及Nginx监控。

##### Kong网关的特性

> Kong网关具有以下的特性

- 可扩展性: 通过简单地添加更多的服务器，可以轻松地进行横向扩展，这意味着您的平台可以在一个较低负载的情况下处理任何请求；
- 模块化: 可以通过添加新的插件进行扩展，这些插件可以通过RESTful Admin API轻松配置；
- 在任何基础架构上运行: Kong网关可以在任何地方都能运行。您可以在云或内部网络环境中部署Kong，包括单个或多个数据中心设置，以及public，private 或invite-only APIs。

#### kong网关架构

1. Kong核心基于OpenResty构建，实现了请求/响应的Lua处理化；
2. Kong插件拦截请求/响应，如果接触过Java Servlet，等价于拦截器，实现请求/响应的AOP处理；
3. Kong Restful 管理API提供了API/API消费者/插件的管理；
4. 数据中心用于存储Kong集群节点信息、API、消费者、插件等信息，目前提供了PostgreSQL和Cassandra支持，如果需要高可用建议使用Cassandra；
5. Kong集群中的节点通过gossip协议自动发现其他节点，当通过一个Kong节点的管理API进行一些变更时也会通知其他节点。每个Kong节点的配置信息是会缓存的，如插件，那么当在某一个Kong节点修改了插件配置时，需要通知其他节点配置的变更。

##### Kong网关请求流程

> 为了更好地理解系统，这是使用Kong网关的API接口的典型请求工作流程：

![1700128299876.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128299876.png)

​		当Kong运行时，每个对API的请求将先被Kong命中，然后这个请求将会被代理转发到最终的API接口。在请求（Requests）和响应（Responses）之间，Kong将会执行已经事先安装和配置好的任何插件，授权您的API访问操作。Kong是每个API请求的入口点（Endpoint）。

#### kong 部署

##### 搭建网络

​		首先我们创建一个Docker自定义网络，以允许容器相互发现和通信。在下面的创建命令中`kong-net`是我们创建的Docker网络名称。

```apl
docker network create kong-net
```

##### 搭建数据库环境

​		Kong 目前使用Cassandra(Facebook开源的分布式的NoSQL数据库) 或者PostgreSql,你可以执行以下命令中的一个来选择你的Database。请注意定义网络 `--network=kong-net` 。

```apl
docker run -d --name kong-database --network=kong-net -p 5432:5432 -v /tmp/data/kong:/var/lib/postgresql/data -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" -e "POSTGRES_PASSWORD=kong" postgres:11.7
```

##### kong网关部署

###### 初始化kong数据

> 使用`docker run --rm`来初始化数据库，该命令执行后会退出容器而保留内部的数据卷（volume）。

```apl
docker run --rm --network=kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database" -e "KONG_PG_PASSWORD=kong" -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" kong:latest kong migrations bootstrap
```

这个命令我们还是要注意的，一定要跟你声明的网络，数据库类型、host名称一致。同时注意Kong的版本号，本文是在`Kong 1.4.x` 版本下完成的。

> 出现如下信息说明数据初始化成功

![1700128325171.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128325171.png)

###### 启动Kong容器

> 完成初始化数据库后，我们就可以启动一个连接到数据库容器的Kong容器，请务必保证你的数据库容器启动状态，同时检查所有的环境参数 -e 是否是你定义的环境

```apl
docker run -d --name kong \
--network=kong-net \
-e "KONG_DATABASE=postgres" \
-e "KONG_PG_HOST=kong-database" \
-e "KONG_PG_PASSWORD=kong" \
-e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
-e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
-e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
-e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
-e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
-e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
-p 8000:8000 \
-p 8443:8443 \
-p 8001:8001 \
-p 8444:8444 \
kong:latest
```

###### 验证

> 通过服务器执行curl访问尝试服务是否已经启动

```apl
curl http://localhost:8001/status
```

![1700128350572.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128350572.png)

> 出现如下信息说明服务已经启动

#### Kong配置

##### nginx模板

> 我们来看一个典型 Nginx 的配置对应在 Kong 上是怎么样的，下面是一个典型的 Nginx 配置

```apl
upstream passportUpstream {
    server 192.168.64.1:9001 weight=10;
    server 192.168.64.1:9002 weight=10;
}
server {
    listen 80;
    location /hello {
        proxy_pass http://passportUpstream;
    }
}
```

##### kong配置案例

> 下面我们开看看其对应 Kong 中的配置

```apl
# 配置 upstream 
curl -X POST http://localhost:8001/upstreams --data "name=passportUpstream" 

# 配置 target 
curl -X POST http://localhost:8001/upstreams/passportUpstream/targets  --data "target=172.16.44.47:9001" --data "weight=10" 

# 配置 service 
curl -X POST http://localhost:8001/services --data "name=hello" --data "host=passportUpstream" 

# 配置 route 
curl -X POST http://localhost:8001/routes \
--data "paths[]=/hello" \
--data "service.id=2fa18352-2aed-449d-9707-8cbfadba3972"
```

​		这一切配置都是通过其 Http Restful API 来动态实现的，无需我们再手动的 reload Nginx.conf

##### 测试验证

> 可以通过如下URL进行测试

```apl
http://192.168.245.145:8000/hello/index
```

![1700128389528.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128389528.png)

##### Kong关键术语

​		在上述的配置中涉及到了几个概念： Upstream、Target、Service、Route 等概念，它们是 Kong 的几个核心概念，也是我们在使用 Kong API 时经常打交道的，下面我们就其几个核心概念做一下简单的说明。

###### Upstream

​		Upstream 对象表示虚拟主机名，可用于通过多个服务（目标）对传入请求进行负载均衡。例如：service.v1.xyz 为 Service 对象命名的上游 Host 是 service.v1.xyz 对此服务的请求将代理到上游定义的目标。

###### Target

> 目标 IP地址/主机名，其端口表示后端服务的实例。每个上游都可以有多个 Target，并且可以动态添加 Target。

​		由于上游维护 Target 的更改历史记录，因此无法删除或者修改 Target。要禁用目标，请发布一个新的 Targer weight=0，或者使用 DELETE 来完成相同的操作

###### Service

> 顾名思义，服务实体是每个上游服务的抽象，服务的示例是数据转换微服务，计费API等。

​		服务的主要属性是它的 URL（其中，Kong 应该代理流量），其可以被设置为单个JSON串或通过指定其 protocol， host，port 和path。

​		服务与路由相关联（服务可以有许多与之关联的路由），路由是 Kong 的入口点，并定义匹配客户端请求的规则。一旦匹配路由，Kong 就会将请求代理到其关联的服务。

###### Route

> 路由实体定义规则以匹配客户端的请求。每个 Route 与一个 Service 相关联，一个服务可能有多个与之关联的路由，与给定路由匹配的每个请求都将代理到其关联的 Service 上。可以配置的字段有:

- hosts
- paths
- methods

Service 和 Route 的组合（以及它们之间的关注点分离）提供了一种强大的路由机制，通过它可以在 Kong 中定义细粒度的入口点，从而使基础架构路由到不同上游服务。

###### Consumer

​		Consumer 对象表示服务的使用者或者用户，你可以依靠 Kong 作为主数据库存储，也可以将使用者列表与数据库映射，以保持Kong 与现有的主数据存储之间的一致性。

###### Plugin

​		插件实体表示将在 HTTP请求/响应生命周期 期间执行的插件配置。它是为在 Kong 后面运行的服务添加功能的，例如身份验证或速率限制。

​		将插件配置添加到服务时，客户端向该服务发出的每个请求都将运行所述插件。如果某个特定消费者需要将插件调整为不同的值，你可以通过创建一个单独的插件实例，通过 service 和 consumer 字段指定服务和消费者。

#### kongAPI操作

##### 配置网关

###### 配置服务

```apl
# 添加服务
curl -i -X POST http://192.168.64.181:8001/services/ -d 'name=test-service' -d 'url=http://116.62.213.90/1.html'
```

> url 参数是一个简化参数，用于一次性添加 protocol，host，port 和 path。

###### 添加路由

```apl
# 添加service 的路由
curl -i -X POST http://192.168.64.181:8001/routes/ -d 'methods=GET' -d 'paths=/service' -d 'service.id=65c11356-e86d-431e-a76a-aa7c3acd9aeb'
```

###### 访问测试

```apl
# 通过如下URL访问
http://192.168.64.181:8000/service?a=123
```

##### 配置负载均衡

###### 配置 upstream 和 target

> 创建一个名称 load 的 upstream

```apl
curl -X POST http://192.168.64.181:8001/upstreams --data "name=load"
```

> 添加3000 端口的负载

```apl
curl -X POST http://192.168.64.181:8001/upstreams/load/targets --data "target=192.168.64.1:9001" --data "weight=10"

curl -X POST http://192.168.64.181:8001/upstreams/load/targets --data "target=192.168.64.1:9002" --data "weight=10"
```

> 如上的配置对应了 Nginx 的配置

```apl
upstream load {
    server 192.168.64.181:9001 weight=10;
    server 192.168.64.181:9002 weight=10;
}
```

###### 配置service

> host 的值便对应了 upstream 的名称，配置成功后会返回生成的 service 的 id

```apl
curl -X POST http://192.168.64.181:8001/services --data "name=load-service" --data "host=load"
```

###### 配置路由信息

> 这里的service.id 就是上面添加service

```apl
curl -X POST http://192.168.64.181:8001/routes --data "paths[]=/load" --data "service.id=3a6b839b-00c1-49b7-b271-8024a12d19be"
```

###### 访问测试

> http://192.168.64.181:8000/load/index

#### 安装Kong 管理UI

> Kong 企业版提供了管理UI，开源版本是没有的。但是有很多的开源的管理 UI ，其中比较好用的是Konga。项目地址：https://github.com/pantsel/konga

![1700128434315.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128434315.png)

##### 初始化konga数据

> 使用`docker run --rm`来初始化数据库，该命令执行后会退出容器而保留内部的数据卷（volume）。

```apl
docker run --name konga --rm \
--network=kong-net \
pantsel/konga -c prepare -a postgres -u postgresql://kong:kong@kong-database:5432/kong
```

> 执行后出现如下界面说明执行成功

![1700128450109.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128450109.png)

##### 启动konga

```apl
docker run -p 1337:1337 -d \
--network=kong-net \
-e "DB_ADAPTER=postgres" \
-e "DB_HOST=kong-database" \
-e "DB_USER=kong" \
-e "DB_PASSWORD=kong" \
-e "DB_DATABASE=kong" \
-e "KONGA_HOOK_TIMEOUT=120000" \
-e "DB_PG_SCHEMA=public" \
-e "NODE_ENV=production" \
--name konga \
pantsel/konga
```

##### 验证

> 通过浏览器请求 http://IP:1337 检查是否能够访问，如果能够访问说明已经启动成功

![1700128465851.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128465851.png)

##### 连接kong网关

> 在这个界面添加一个用户 登录就可以看到如下页面

![1700128479212.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128479212.png)

> 点击Connections 来填加kong的信息

![1700128514278.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128514278.png)

> 添加kong名称以及API地址就可以管理kong了

![1700128529043.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128529043.png)

> 添加后点击'active'激活按钮就可以激活kong网关

![1700128542578.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128542578.png)

> 然后就会出现如下界面管理可以通过菜单操作管理kong

![1700128634647.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128634647.png)

##### 限流插件

> 限流的场景：服务提供的能力是有限的。为了防止大量的请求将服务拖垮，可以通过网关对服务的请求做限流。例如：某个服务1s只能处理100个请求，超过限流阈值的请求丢弃。

###### 路由配置限流

> 为/load的路由添加限流插件

![1700128649144.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128649144.png)

> 设置限流为每分钟2个

![1700128670474.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128670474.png)

###### 测试验证

> 多次访问 http://192.168.64.150:8000/load 发现已经被限流了

![1700128681918.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1700128681918.png)

###### 配置属性如下

| 属性                | 说明                                                         | 是否必须 | 默认       | 示例     |
| ------------------- | ------------------------------------------------------------ | -------- | ---------- | -------- |
| consumer            | 设置消费之，当时用身份认证时能够识别出消费者                 | 否       | 所有消费者 |          |
| second              | 限制每秒最多有几个请求                                       | 是       | 无         | 2        |
| minute              | 限制每分钟最多有几个请求                                     | 是       | 无         | 10       |
| hour                | 限制每小时最多有几个请求                                     | 是       | 无         | 100      |
| day                 | 限制每天最多有几个请求                                       | 是       | 无         | 100      |
| year                | 限制每年最多有几个请求                                       | 是       | 无         | 100      |
| limit by            | 统计限额的标准，consumer, credential, ip, service，如果无法确定，将以IP为主 | 否       | consumer   | consumer |
| policy              | cluster:将计数器保存在数据库里，local:将计数器保存在本地，redsi:将计数器保存在redis里面 | 是       | cluster    | cluster  |
| fault tolerant      | 第三方数据存储遇到问题时是否会代理请求，如果为YES，在数据库恢复正常前，限流将会禁用，如果为 NO，将会报500错误 | 是       | YES        | YES      |
| redis host          | 当 policy 为 redis 时设置                                    | 否       | 无         |          |
| redis port          | 当 policy 为 redis 时设置                                    | 否       | 6379       |          |
| redis password      | 当 policy 为 redis 时设置                                    | 否       | 无         |          |
| redis timeout       | 当 policy 为 redis 时设置                                    | 否       | 2000       |          |
| redis database      | 当 policy 为 redis 时设置                                    | 否       | 0          |          |
| hide client headers | 隐藏客户端响应头                                             | 是       | NO         | NO       |