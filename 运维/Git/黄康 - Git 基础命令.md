# 黄康 - Git 命令

## 基础命令

```shell
git status # 查看当前 Git Repository 状态，红色文件为新修改文件，绿色为已添加到本地库

# 下述笔记中 [文件名] 均用 xxx 代理
git rm [文件名] # 删除本地库中的 Git 管理文件，Push 后同步到远端

git add xxx # 添加文件到本地库

git add . # 添加所有修改过的文件到本地库

git commit -m "提交注释" xxx # 提交本地库指定修改文件

git commit -m "提交注释" # 提交本地库所有修改文件

git rm --cached xxx # 撤销提交
```

## 分支管理

```shell
git branch -a # 查看本地库和远端的所有分支

git branch -r # 查看本地库所有分支

git remote add origin git@xxx # 添加远端连接

git push origin master # 推送本地库文件到远端 master 分支

git remote show origin # 显示远端资源

git checkout --track origin/master # 切换到远端 master 分支

git branch -D master rm dev # 删除本地库 dev 分支

git checkout -b xxx # 有则切换，无则建立[分支]

git merge origin/dev # 将远端 dev 分支合并到当前分支
```

## 远端命令

```shell
git clone xxx[远端地址] # 克隆远端仓库

git remote -v # 查看远程仓库

git remote add [name] [url] # 添加远程仓库

git remote rm [name] # 删除远程仓库

git remote set-url --push [name] [newUrl] # 修改远程仓库

git pull # 拉取

git push # 推送
```

