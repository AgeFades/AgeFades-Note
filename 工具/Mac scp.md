# Mac OS下使用scp

Mac OS的shell自带命令scp，可以通过scp来上传和下载文件。

### 上传

```ruby
scp -r local_folder remote_username@remote_ip:remote_folder
```

其中，`local_folder`是源地址，`remote_username@remote_ip:remote_folder`是目标地址。命令会将源地址的内容上传到目标地址。

### 下载

```ruby
scp -r remote_username@remote_ip:remote_folder local_folder 
```

下载和上传对应，将2个参数互换顺序就可以。

### 命令参数

有一些好用的scp命令参数。

```css
1.-v 显示进度
2.-r 递归处理
3.-C 压缩选项
4.-P 选择端口
```

> `-p`已经被rcp使用。

更多参数可以通过命令`man scp`来查看。

## 18.4. 使用示例

### 实例1：从远处复制文件到本地目录

```
$scp root@10.6.159.147:/opt/soft/demo.tar /opt/soft/
```

说明： 从10.6.159.147机器上的/opt/soft/的目录中下载demo.tar 文件到本地/opt/soft/目录中

### 实例2：从远处复制到本地

```
$scp -r root@10.6.159.147:/opt/soft/test /opt/soft/
```

说明： 从10.6.159.147机器上的/opt/soft/中下载test目录到本地的/opt/soft/目录来。

### 实例3：上传本地文件到远程机器指定目录

```
$scp /opt/soft/demo.tar root@10.6.159.147:/opt/soft/scptest
```

说明： 复制本地opt/soft/目录下的文件demo.tar 到远程机器10.6.159.147的opt/soft/scptest目录

### 实例4：上传本地目录到远程机器指定目录

```
$scp -r /opt/soft/test root@10.6.159.147:/opt/soft/scptest
```

说明： 上传本地目录 /opt/soft/test到远程机器10.6.159.147上/opt/soft/scptest的目录中