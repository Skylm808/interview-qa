# Linux常考题目面经

> 这一份不追求百科全书式展开，目标是面试里能把“现象、定位、原因、结论”说清楚。

## 1. 面试里怎么答故障诊断

我一般按 4 步说：

1. **先看现象**：是变慢、超时、CPU 高、内存高，还是磁盘打满。
2. **再看资源**：CPU、内存、磁盘 I/O、网络、连接数、fd。
3. **再缩范围**：是机器问题、进程问题、依赖问题，还是代码本身问题。
4. **最后落结论**：根因是什么，临时止血怎么做，长期优化怎么做。

一句话模板：

“我排障不会一上来猜代码，而是先看系统资源和错误现象，再逐层缩小范围，最后区分短期止血和长期优化。”

## 2. 常见排查维度

### CPU 高

- 先看是不是系统整体 CPU 高，还是某个进程高。
- 再看是用户态高、系统态高，还是上下文切换高。
- 如果是单进程高，再结合线程栈、火焰图、热点函数定位。

常见原因：

- 死循环
- 热点请求暴增
- 锁竞争
- GC 频繁
- 大量 JSON 编解码 / 正则 / 排序

### 内存高

- 先区分是缓存占用、正常堆增长，还是内存泄漏。
- 再看 RSS、堆大小、GC 频率、OOM 记录。

常见原因：

- 对象堆积
- 大量连接未释放
- 缓存没有上限
- goroutine 泄漏
- 消息堆积导致队列膨胀

### 磁盘 I/O 高

- 看读高还是写高，随机 I/O 还是顺序 I/O。
- 看是不是日志刷盘、MySQL 慢查询、消息堆积落盘导致。

### 网络问题

- 看延迟、丢包、重传、连接数、TIME_WAIT、backlog。
- 区分是应用超时，还是下游网络真的抖动。

## 3. 常见命令

```bash
top
htop
free -h
vmstat 1
iostat -x 1
sar -n DEV 1
ss -lnt
ss -s
lsof -p <pid>
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
dmesg | tail
journalctl -xe
```

如果是排进程级问题，还会补：

```bash
pidstat -p <pid> 1
strace -p <pid>
pmap -x <pid>
```

## 4. 高频面试题

### Q1. CPU 飙高怎么查？

**答：**

- 先用 `top` / `htop` 看是哪台机器、哪个进程高。
- 再用 `pidstat` 或线程视角看是不是某个线程打满。
- 如果是应用进程，再结合 profile、火焰图、日志时间点定位热点逻辑。

一句话版：

“先确定是不是系统级还是进程级 CPU 高，再定位到线程和热点函数，最后再看是流量问题、锁竞争还是代码热点。”

### Q2. 内存一直涨怎么查？

**答：**

- 先看是业务峰值期间的正常上涨，还是请求结束后也不回落。
- 再看缓存、连接、队列、goroutine 数是否持续增长。
- 如果怀疑泄漏，再抓堆快照或 profile 做对象对比。

### Q3. load average 很高，但 CPU 不高，可能是什么原因？

**答：**

这通常说明不一定是“纯算力不够”，也可能是：

- 磁盘 I/O 等待高
- 锁等待高
- 某些线程阻塞在不可中断睡眠

面试里可以直接说：

“load 高不等于 CPU 一定高，因为它统计的是可运行和不可中断等待任务数，I/O 卡住时也会把 load 顶上去。”

### Q4. 服务器突然 OOM 了，怎么分析？

**答：**

- 先查 `dmesg` 或系统日志，看是不是 OOM Killer 杀的。
- 再看哪个进程占用最大。
- 再倒推是缓存膨胀、消息堆积、内存泄漏，还是并发突增。

### Q5. 出现 `too many open files` 怎么办？

**答：**

- 先确认 fd 是被谁打满的：连接、文件、日志句柄还是 socket。
- 再看进程级和系统级 fd 限制。
- 临时可以提上限，长期要解决连接泄漏和资源不释放的问题。

## 5. 面试加分点

- 不只说“查日志”，而是能说出先看哪些系统指标。
- 不只说“扩容”，而是能区分临时止血和根因修复。
- 能把“CPU、内存、磁盘、网络、连接、依赖”这几个维度讲成一套方法论。

## 6. 30 秒总答模板

“Linux 故障诊断我一般先看现象，再看资源，再缩小到进程和依赖。比如 CPU 高我会先确认是系统级还是单进程问题；内存高会区分缓存膨胀和泄漏；load 高但 CPU 不高时会优先怀疑 I/O 或锁等待。我的思路不是上来拍脑袋改代码，而是先把瓶颈维度判断清楚，再做止血和长期优化。”

## 7. fzh 公司面经补充

> 来源：`fzh/` 公司面经中抽取的 Linux / OS 排障基础追问。

### Q6. 有哪些方式可以查看线程切换指标？

**答：**

- `vmstat` 看系统级上下文切换次数，例如 `cs` 字段。
- `pidstat -w -p <pid>` 看指定进程的上下文切换。
- `perf stat` 看更细的 CPU 调度和上下文切换指标。
- Java 进程还可以结合线程 dump、JFR、async-profiler 观察线程状态和锁竞争。

面试里要能解释：上下文切换过高通常说明线程过多、锁竞争、频繁阻塞唤醒或 IO 等待明显。

### Q7. `top` 命令重点看哪些指标？用户态 / 系统态 CPU 高说明什么？

**答：**

- 看 load average，判断任务排队压力。
- 看 `%us`，用户态 CPU 高通常说明业务代码计算忙。
- 看 `%sy`，系统态 CPU 高通常说明系统调用、网络 IO、锁、内核处理开销大。
- 看 `%wa`，IO wait 高说明磁盘或存储链路可能是瓶颈。
- 看内存、swap、进程 CPU 和 RES，判断是否内存紧张或单进程异常。

### Q8. 线程数过多会对进程造成什么影响？

**答：**

- 增加上下文切换，CPU 花更多时间在线程调度上。
- 占用更多线程栈内存，可能推高进程内存。
- 放大锁竞争和调度延迟。
- 对 IO 密集型任务可能短期提升吞吐，但如果下游容量不足，会造成排队和雪崩。

所以排查时要把线程数、CPU、上下文切换、队列长度、下游 RT 放在一起看。

---

## Linux 基础命令与排障

### Linux 系统常见命令有哪些？

**文件操作：**

| 命令 | 全称 / 来源 | 含义 |
| --- | --- | --- |
| `ls` | List | 列出目录内容 |
| `-la` | `--long --all` | 长格式，显示隐藏文件 |
| `cd` | Change Directory | 切换目录 |
| `cp` | Copy | 复制 |
| `mv` | Move | 移动或重命名 |
| `rm` | Remove | 删除 |
| `-rf` | `--recursive --force` | 递归删除，且不交互确认 |
| `mkdir` | Make Directory | 创建目录 |
| `-p` | `--parents` | 自动创建不存在的父目录 |
| `cat` | concatenate | 原意为连接多个文件并输出 |
| `less` | "less is more" | 分页查看；`more` 的改进版 |
| `head` / `tail` | 头部 / 尾部 | 查看文件开头 / 结尾 |
| `-f` | `--follow` | 跟踪追加内容，常用于日志 |
| `find` | find | 查找文件 |
| `grep` | Global Regular Expression Print | 按正则搜索并输出匹配行 |

**进程管理：**

| 命令 | 全称 / 来源 | 含义 |
| --- | --- | --- |
| `ps` | Process Status | 查看进程状态 |
| `aux` | `a` + `u` + `x` | 所有终端的进程 + 用户格式 + 含无控制终端进程 |
| `top` | top | 动态显示最消耗资源的进程 |
| `htop` | interactive top | 更易交互的 `top` 实现 |
| `kill` | kill | 向进程发送信号，不一定是强制终止 |
| `-9` | `SIGKILL` | 内核强制终止，进程无法捕获或清理 |
| `killall` | kill all | 按进程名发送信号 |
| `nohup` | No Hangup | 忽略终端挂断信号 `SIGHUP` |
| `jobs` / `fg` / `bg` | jobs / Foreground / Background | 查看、切到前台、转到后台的 shell 作业 |

**网络相关：**

| 命令 | 全称 / 来源 | 含义 |
| --- | --- | --- |
| `netstat` | Network Statistics | 网络连接与统计信息 |
| `-tlnp` | TCP + listen + numeric + program | TCP 监听端口，数字显示，附带进程 |
| `ss` | Socket Statistics | socket 统计，常作为 `netstat` 替代品 |
| `-ant` | all + numeric + TCP | 全部 TCP 连接，数字显示地址和端口 |
| `curl` | Client URL | URL 客户端，常用于 HTTP 调试 |
| `wget` | World Wide Web Get | 下载网络资源 |
| `ping` | 来自声纳回声的拟声词 | 用 ICMP 检查可达性和时延 |
| `telnet` | Teletype Network | 早期远程终端协议，也可粗略测试 TCP 端口 |

**系统信息：**

| 命令 | 全称 / 来源 | 含义 |
| --- | --- | --- |
| `df` | Disk Free | 挂载点的可用磁盘空间 |
| `-h` | `--human-readable` | 以 GB、MB 等易读单位显示 |
| `du` | Disk Usage | 目录或文件占用空间 |
| `-sh` | summary + human-readable | 汇总目录大小并用易读单位显示 |
| `free` | free | 内存使用概览 |
| `uname` | Unix Name | 系统名称和内核信息 |
| `-a` | `--all` | 显示全部可用系统信息 |
| `uptime` | uptime | 运行时长、负载和登录用户数 |
| `who` | who | 当前登录用户 |

**文本处理：**

| 命令 | 全称 / 来源 | 含义 |
| --- | --- | --- |
| `awk` | Aho, Weinberger, Kernighan | 按列处理文本 |
| `sed` | Stream Editor | 流编辑器，常用于替换和过滤 |
| `sort` | sort | 排序 |
| `uniq` | unique | 合并或筛除相邻重复行，常配合 `sort` |
| `wc` | Word Count | 统计行、词、字节数 |

---

### 服务器 CPU 100% 如何定位问题？

**一句话总结：**先用 `top` / `ps` 找高 CPU 进程，再继续定位到线程、函数栈和具体代码路径，不要一上来就猜业务逻辑。

**排查步骤：**

```bash
# 1. 找出 CPU 占用高的进程
top -c
ps aux --sort=-%cpu | head

# 2. 找出进程中占用高的线程
top -Hp <pid>
ps -mp <pid> -o THREAD,tid,time

# 3. 查看线程堆栈
printf '%x\n' <tid>                     # 线程 ID 转为十六进制
jstack <pid> | grep <tid_hex> -A 30    # Java
pstack <pid>                            # C/C++
```

**Go 程序排查：**

```bash
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
# 在 pprof 中可使用：top、list funcName、web

curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out
```

常见原因包括：死循环、正则表达式回溯、频繁 GC、锁竞争和无限递归。

---

### Linux 磁盘满了怎么定位？CPU 高负载又该怎么排查？

**一句话总结：**磁盘满先看“哪个分区满、哪个目录大、哪个文件大”；CPU 高负载先看“哪个进程高、哪个线程高、热点函数在哪”。

**磁盘满排查：**

先确认哪个挂载点满了：

```bash
df -h
```

例如 `/dev/vda1` 的 `Use%` 为 `99%` 且挂载在 `/`，就说明根分区快满了。接着从大目录往下找：

```bash
du -sh /* 2>/dev/null | sort -hr | head
du -sh /var/* 2>/dev/null | sort -hr | head

find /var/log -type f -size +500M -ls | sort -k7 -hr | head
ls -lh /var/log
tail -n 200 /var/log/messages
```

还必须检查“文件已删除、但仍被进程打开”的情况：

```bash
lsof | grep deleted
```

文件虽然删了，只要进程没有关闭该文件，磁盘空间就不会立即释放。

**CPU 高负载排查：**

```bash
# 先看整体负载和高 CPU 进程
uptime
top -c
ps aux --sort=-%cpu | head

# 再定位到线程
top -Hp <pid>
ps -mp <pid> -o THREAD,tid,time

# 看线程栈或 profile
printf '%x\n' <tid>
jstack <pid>      # Java
pstack <pid>      # C/C++

# Go 服务
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
```

面试里可以直接这样答：

> Linux 磁盘满了，我会先用 `df -h` 看哪个分区满，再用 `du -sh` 和 `find` 往下找大目录、大文件，日志和临时文件最常见；CPU 高负载我会先用 `top -c` 或 `ps aux --sort=-%cpu` 找高 CPU 进程，再定位到线程和函数栈，Java 看 `jstack`，Go 看 `pprof`。

---

### 端口占用怎么排查？

- 最直接的思路是先找“是谁占了这个端口”。
- 常用命令有 `lsof -i :端口`、`ss -lntp`、`netstat -lntp`。
- 查到 PID 后，再结合 `ps -ef | grep PID` 看具体进程，是不是残留进程或退出不彻底。
- 如果没有明显活进程，还要继续看连接状态，例如大量 `TIME_WAIT`、`CLOSE_WAIT` 是否造成短时间不能直接复用。

---

### 服务部署失败、上游调用也失败了，应该怎么排查？

我一般按五层排：

- **进程层**：进程起来没有，退出码是什么，启动命令有没有报错。
- **端口层**：端口是否监听成功，有没有被别的进程占用。
- **依赖层**：数据库、Redis、配置中心、DNS、下游服务是否可达。
- **资源层**：CPU、内存、磁盘、文件句柄、权限是否异常。
- **日志层**：服务启动日志、系统日志、容器日志、错误栈里到底报了什么。

常见排查命令：

```bash
# 1. 看进程
ps aux | grep service-name
systemctl status service-name

# 2. 看端口
ss -lntp | grep 8080
lsof -i :8080

# 3. 看日志
tail -f app.log
journalctl -u service-name -n 200 --no-pager
docker logs --tail 200 -f container-name

# 4. 看依赖是否可达
curl http://downstream/health
nc -zv mysql-host 3306
ping redis-host

# 5. 看资源
top -c
free -h
df -h
ulimit -n
```

完整顺序是：先确认服务没起来还是没监听成功；再区分上游是连接失败、超时、5xx 还是业务报错；然后看启动日志与错误栈，同时检查依赖、配置、端口、磁盘和文件句柄。容器场景还要结合 `docker logs`、健康检查与退出码。

面试里可以这样答：

> 我会先看进程和端口，判断服务是没起来还是没监听成功；再看启动日志、容器日志和系统日志；同时检查依赖可达性和资源情况，例如端口冲突、配置缺失、磁盘满、数据库连不上。日志是核心证据，但排障不能只盯日志，还要结合进程、网络和依赖一起看。

---

### 如何快速排查一段代码运行异常的问题？

- 先将问题分为：代码逻辑错误、依赖 / 环境错误、性能 / 资源问题。
- 常见顺序：先看错误日志和关键输入，再最小化复现，然后检查最近改动；必要时增加日志、打点或 profile。
- 线上问题还要结合 CPU / 内存、goroutine / 线程数、连接数、队列堆积和外部依赖延迟。

一句话答法：**排查问题最怕一开始就陷进代码细节，先做分类、先缩小范围，效率最高。**
