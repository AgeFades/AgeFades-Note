#### CPU负载和CPU利用率的区别是什么？

##### 命令查看 CPU 的平均负载

1. ```shell
   uptime
   w
   ```

   ![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhvwRIlJd8pSfSb6AAxw1kUhFPJ8KibRia4KxkHWHwBQvTOkhtib39Kxfvg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. ```shell
   top
   ```

   ![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhDY6yINBYhEycwlZWr8XxGqhiaS5mozVMVZ3yq42H2dAyZWgQXxkFUqg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### Load Average

- 三个数字，代表系统在过去的 `1分钟、5分钟、15分钟内` 的系统平均负载
  - 代表 当前系统 `正在运行的 和 处于等待运行的进程数之和`
  - 也指 `处于可运行状态 和 不可中断状态 的平均进程数`

##### CPU 负载分析

- 单核CPU时，`Load Average 达到1`、就代表 `CPU满负荷`
  - Load Average 超过 1时，后面的进程就需要排队等待处理了
- 多核多CPU时，
  - 如: 2个CPU、每个CPU2个核，那总负载不超过 4 就没什么问题

##### 查看CPU有多少核

```shell
# 查看 CPU 情况
cat /proc/cpuinfo | grep "model name"

# 查看 CPU 核数
cat /proc/cpuinfo | grep "cpu cores"
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhSuPhGGBMah9FW9Z6bI1lzsHwDCJiaepVIicic3JsD6GXl28ZBcJXR22Ag/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhOemAFZSIiaase0bGTt4Mt8AXT4hJec9m26eQbvxrM6W1lI0QgPAglwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### CPU 利用率

- 当前正在运行的进程、实时占用CPU的百分比
  - 是对一段时间内，CPU使用状况的统计
- 比如: 
  - 单核 CPU，A进程占用、B进程排队，负载就是 2
  - 60min内，A进程使用10min、B进程使用20min，这个小时利用率就是 50%

#### CPU负载很高，利用率很低怎么办？

- 说明处于 `等待状态` 的任务很多，负载越高，代表可能很多僵死进程
  - 通常这种情况是 `IO密集型` 任务，大量请求在请求相同的IO，导致任务队列堆积

##### 解决思路

- 先通过 `top` 命令观察
  - 在通过命令 ps -axjf 查看是否存在状态为 `D+` 状态的进程
- `D+状态`：
  - 指 `不可中断的睡眠状态` 的进程，无法终止、无法自行退出
  - 只能通过 `恢复其依赖的资源` 或 `重启系统` 解决

#### CPU负载很低，利用率很高怎么办？

- 通常是 `计算密集型任务`，产生了 `大量耗时短的计算任务`
  - 可能代码写的点有问题，比如死循环之类
- 通过 `top` 找到使用率最高的任务，再进行定位即可

#### CPU使用率达到100%怎么办？

- 通过 `top` 找到使用率高的进程

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhuRsibVXxW9SRfyqpQ3BibwBQPtgsCBfDAaeXdYcVeQTUjuYYIocdqMeA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 通过 top -Hp pid 找到占用 CPU高 的 `线程id`

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhRs9ZKX1UAuuABVHeqWiaMJsDOUjGIjUyHnJAbLEicWgD4iaWQYaibklbag/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 把线程id转化为16进制 

```shell
printf "0x%x\n" 线程id
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhKOCk3psjNH4lOMxJwKmP5033B8icpshzkQvdzDottaVh28kcZ6zJlxw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 查找问题代码

```shell
# 0x3be 是一步转16进制后的线程id
jstack 163 | grep '0x3be' -C5 --color
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUmvPFvQ3yVL1tQYGibHicYClhgPK9rmZ5npLAtgmIia4TLjFdGVrSXAAZeMn3m5RE8tKMLn9WBnU9CZw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 常用Linux命令

```shell
# 随便说几个
ls
touch
more
less	
tail
chmod
chown
zip
unzip
gzip
tar
top
ps
```

