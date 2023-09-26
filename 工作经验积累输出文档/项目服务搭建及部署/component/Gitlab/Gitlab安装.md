## 安装

```bash
# 创建挂载目录
mkdir config logs data
```

## 修改配置

```bash
vim config/gitlab.rb
```

```apl
external_url 'http://117.50.175.156:10080'
nginx['listen_port'] = 10080
gitlab_rails['gitlab_shell_ssh_port'] = 10022
nginx['listen_https'] = false
postgresql['shared_buffers'] = "128MB"
postgresql['max_worker_processes'] = 4
sidekiq['concurrency'] = 15
```

## docker-compose

```bash
vim docker-compose-gitlab.yaml
```

```yaml
version: '3'
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    restart: always
    ports:
      - '10080:10080'
      - '10022:22'
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'
```

### 简单部署脚本

```shell
vim deploy.sh
```

```shell
# deploy.sh 内容
docker-compose -f docker-compose-gitlab.yaml down
docker-compose -f docker-compose-gitlab.yaml up -d
```

### 执行安装

```shell
sh deploy.sh
```

# GitLab默认密码

[CSDN原文链接](https://blog.csdn.net/timonium/article/details/119451755)

- 默认用户名：root

- gitlab-ce-14初装以后，把密码放在了一个临时文件中了
  `/etc/gitlab/initial_root_password`
- 这个文件将在首次执行reconfigure后24小时自动删除，记得改密码

## 注意

- 按上面的配置，需要开放服务器端口10022，否则ssh免密连接不上

