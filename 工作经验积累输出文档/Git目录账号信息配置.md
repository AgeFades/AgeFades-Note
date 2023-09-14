## 需求场景

- 自己全局的 git 账号为 xxx(花名或乱七八糟的名字)、全局 git 邮箱为 xxx(qq邮箱或乱七八糟的邮箱)
- 入职之后，往公司 git 提交代码，如果公司需要改成本人真实名字、公司分配邮箱

### 举例

- github 个人笔记项目

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1634110098536.png)



- 公司项目

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1634110142172.png)

## Mac

```shell
# 新增 git 局部配置文件
# 举例 ~/.gitconfig-ali
vim ~/.gitconfig-公司名

----- 配置文件开始 -----
[user]
        name = 你的名字
        email = 你的公司邮箱
----- 配置文件结束 -----
        
# 编辑 git 全局配置文件
vim ~/.gitconfig

----- 配置文件开始 -----
[user]
        name = AgeFades # 这是我的git全局配置，无需关心
        email = 18433216@qq.com # 这是我的git全局配置，无需关心
        
# 配置局部目录账号邮箱信息        
[includeIf "gitdir:/Users/apple/Documents/Work/xxx/xxx/"]	# 新增配置行
        path = .gitconfig-公司名	# 新增配置行
        
[credential]
        helper = store
[core]
        autocrlf = input
[filter "lfs"]
        clean = git-lfs clean -- %f
        smudge = git-lfs smudge -- %f
        process = git-lfs filter-process
        required = true
----- 配置结束开始 -----

# 注意，上述示例在本地修改 user.name(真实姓名)、user.email(公司邮箱)、gitdir(修改为自己本地存储公司代码目录)
```

## Windows

- windows git 全局配置文件一般都在自己的 User 目录下

- 举例：

  - ```shell
    C:\Users\Administrator\.gitconfig
    ```

- 找到全局配置文件后，用 git bash 按上面的方法操作就行

