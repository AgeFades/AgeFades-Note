### 创建账号token

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623223912763.png)

### 保存账号密码

```shell
# 第一次以后不用每次和远端操作都输入账号密码
git config --global credential.helper store
```

### clone代码

```shell
# 账号 -> 你的账号，密码 -> 刚生成的token
git clone https://gitlab.botpy.com/Botpy/VOSP/JDY-Backend.git
```

