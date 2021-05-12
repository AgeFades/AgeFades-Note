## Mac安装Go

[下载地址](https://golang.org/dl/)

1. 下载Mac系统对应Go语言安装包
2. 执行安装包
3. 终端执行命令 `go env`，成功输出环境信息则表示安装成功

## Hello World

```shell
# 新建 go 文件
vim hello.go
```

```go
// 在 hello.go 文件中编写代码
package main

import "fmt"

func main() {
  fmt.Print("Hello World")
}
```

```shell
# 执行 go 文件
go run hello.go
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620618318896.png)

