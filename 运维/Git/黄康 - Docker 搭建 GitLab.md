# 黄康 - Docker 搭建 GitLab

## 创建挂载目录

```shell
mkdir -p /docker/gitlab/{config,logs,data}
```

## 启动容器

```shell
docker run --restart=always -d \
--name gitlab \
-h [自己的ip] \
-p 10443:443 \
-p 10080:80 \
-p 10022:22 \
-v /docker/gitlab/config:/etc/gitlab \
-v /docker/gitlab/logs:/var/log/gitlab \
-v /docker/gitlab/data:/var/opt/gitlab \
docker.io/twang2218/gitlab-ce-zh

# 访问端口为 10080，设置默认 root 密码
# 以上命令不要使用 root，否则会出现权限问题
```

## 问题

```shell
# 修改之后，发现 url 和 ssh 上都没有端口号了
# 进入挂载目录并且编辑配置文件
vim /docker/gitlab/config/gitlab.rb

# 添加以下内容
external_url 'http://[自己的ip]:10080'
nginx['listen_port'] = 10080
gitlab_rails['gitlab_shell_ssh_port'] = 10022
nginx['listen_https'] = false

# 重启新的容器
docker stop gitlab
docker rm gitlab
# 执行上面的 docker run 命令
```

