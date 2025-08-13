```table-of-contents
```
## 1 strace介绍
`strace` 是 Linux 系统中一款强大的调试与分析工具，主要用于**跟踪用户态程序与内核之间的系统调用（syscall）和信号传递**。通过记录程序执行过程中发起的所有系统调用（如文件读写、网络连接、进程控制等）及其返回结果，`strace` 能帮助开发者定位程序异常（如文件找不到、权限问题、资源耗尽等）、分析程序行为（如具体访问了哪些文件 / 端口）等

## 2 strace 基本原理

程序运行时，用户态与内核态的交互通过 “系统调用” 完成（例如 `open()` 打开文件、`connect()` 建立网络连接）。`strace` 通过内核的 `ptrace` 机制，在程序执行系统调用时拦截并记录其详细信息，包括：
- 系统调用名称（如 `open`、`read`、`sendto`）；
- 传入的参数（如文件路径、缓冲区地址、端口号）；
- 返回值（成功时的结果或失败时的错误码，如 `ENOENT` 表示文件不存在）；
- 耗时（可选，通过 `-T` 选项显示）。

## 3 apt包安装strace
```bash
sudo apt update  # 更新包索引
sudo apt install strace  # 安装strace
```
## 4 查看是否安装成功
```bash
strace --version  # 若输出版本信息，说明已安装
```

## 5 下载源码
```bash
git clone https://github.com/strace/strace.git
```
### 5.1 源码编译和安装

```bash
./configure  # 配置编译选项
make         # 编译
sudo make install  # 安装到系统
```

## 6 **核心用法与常用选项**

`strace` 的基本语法：
```bash
strace [选项] 命令/程序 [程序参数]  # 跟踪新启动的程序
strace [选项] -p <PID>             # 跟踪已运行的进程（PID为进程ID）
```

### 6.1 常用的选项及场景
#### 6.1.1 基础跟踪：跟踪新程序或已有进程

- **跟踪新启动的程序**（如跟踪 `ls` 命令）：
```bash
strace ls /tmp  # 输出ls命令执行过程中所有的系统调用
```

- **跟踪已运行的进程**（如跟踪 PID=12345 的程序）：
```bash
strace -p 12345  # 实时输出该进程的系统调用
```
提示：按 `Ctrl+C` 停止跟踪，不会影响被跟踪进程的运行。

#### 6.1.2 输出重定向：避免终端刷屏

系统调用可能非常频繁（如循环中的 `read`），直接输出到终端会刷屏，可用 `-o` 选项保存到文件

```bash
strace -o trace.log ./myprogram  # 将所有输出保存到trace.log
```

#### 6.1.3 跟踪子进程：-f 选项

很多程序会通过 `fork()` 或 `clone()` 创建子进程（如 shell 脚本、服务程序），默认情况下 `strace` 只跟踪主进程。用 `-f` 选项可同时跟踪所有子进程
```bash
strace -f -o trace_all.log ./parent_program  # 跟踪主进程及所有子进程
```

#### 6.1.4 过滤特定系统调用：-e 选项
`strace` 默认输出所有系统调用，若只想关注特定类型（如文件操作、网络调用），可用 `-e` 过滤

- **只跟踪文件相关调用**（`open`、`read`、`write`、`close` 等）
```bash
strace -e trace=file ./myprogram  # 仅显示文件操作相关的系统调用
```

- **只跟踪网络相关调用**（`connect`、`send`、`recv`、`bind` 等）

```bash
strace -e trace=network ./myprogram  # 仅显示网络操作相关的系统调用
```

- **只跟踪指定的系统调用**（如只看 `open` 和 `close`）：
```bash
strace -e trace=open,close ./myprogram  # 仅显示open和close调用
```

- **过滤信号**（程序收到的信号，如 `SIGINT`、`SIGTERM`）：
```bash
strace -e signal=all ./myprogram  # 显示程序收到的所有信号
```


#### 6.1.5 时间分析：-tt 和 -T 选项

- `-tt`：在每行输出前添加精确到微秒的时间戳，便于分析调用顺序和耗时点：
```bash
strace -tt ./myprogram  # 输出示例：15:30:45.123456 open("/etc/hosts", O_RDONLY) = 3
```

- `-T`：显示每个系统调用的耗时（单位秒），用于定位耗时较长的调用：
```bash
strace -T ./myprogram  # 输出示例：open("/etc/hosts", O_RDONLY) = 3 <0.000123>
```
结合使用：`strace -tt -T -o trace_time.log ./myprogram`，可详细分析时间分布。

#### 6.1.6 其他实用选项
- `-s <num>`：默认情况下，`strace` 会截断长字符串（如文件路径、缓冲区内容），`-s` 可指定显示长度（默认 32 字节）
```bash
strace -s 1024 ./myprogram  # 字符串最多显示1024字节
```

- `-v`：显示系统调用参数的详细信息（如结构体成员），适合深入分析复杂调用（如 `stat`）
```bash
strace -v ./myprogram  # 详细输出参数信息
```

- `-c`：统计系统调用的总次数、耗时、错误率，生成汇总报告（不显示具体调用过程）
```bash
strace -c ./myprogram  # 输出示例：按调用次数排序的统计结果
```

## 7 **解读 strace 输出**
### 7.1 `strace` 的输出格式大致为
```bash
系统调用名(参数) = 返回值 [耗时]
```

示例解析
```bash
open("/etc/passwd", O_RDONLY) = 3 <0.000056>
```
- `open`：系统调用名称（打开文件）；
- `("/etc/passwd", O_RDONLY)`：参数（要打开的文件路径，只读模式）；
- `= 3`：返回值（成功，文件描述符为 3）；
- `<0.000056>`：耗时（通过 `-T` 显示，单位秒）。
### 7.2 错误情况示例
```bash
open("/etc/unknown", O_RDONLY) = -1 ENOENT (No such file or directory)
```
- 返回值 `-1` 表示调用失败；
- `ENOENT` 是错误码，括号内是对应的描述（文件不存在）。
## 8 **典型使用场景**
### 8.1 调试 “文件找不到” 问题

程序报错 “找不到文件” 但路径正确时，用 `strace` 跟踪 `open` 调用，查看程序实际尝试打开的路径（可能有隐藏的相对路径或环境变量影响）
```bash
strace -e trace=open ./myprogram  # 查看程序所有open调用，定位失败的文件路径
```
### 8.2 分析网络连接失败原因

程序无法连接服务器时，跟踪 `connect` 或 `bind` 调用，查看目标地址、端口是否正确，或是否因权限 / 防火墙导致失败
```bash
strace -e trace=connect ./client  # 查看客户端连接的目标IP:端口，及错误原因
```

### 8.3 排查程序异常退出
程序无提示退出时，跟踪 `exit_group` 或信号（如 `SIGSEGV` 段错误）
```bash
strace -e trace=exit_group,signal=all ./myprogram  # 查看退出原因或收到的信号
```

### 8.4 统计程序系统调用开销
用 `-c` 选项生成统计报告，分析哪些调用最频繁或耗时最长
```bash
strace -c ./myprogram  # 输出示例：
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- ----------------
#  60.00    0.000060           6        10           read
#  30.00    0.000030           3        10           write
#  10.00    0.000010          10         1           open
```

## 9 **注意事项**

1. **性能开销**：`strace` 通过 `ptrace` 跟踪系统调用，会显著降低程序运行速度（可能慢几倍到几十倍），**不建议在生产环境高负载场景使用**。
2. **权限要求**：跟踪其他用户的进程需 `root` 权限（`sudo strace -p <PID>`）。
3. **局限性**：
    - 只能跟踪用户态程序的系统调用，无法跟踪内核态函数（需用 `perf` 或 `eBPF`）；
    - 对于静态链接的程序或无系统调用的纯计算程序，作用有限。


### 9.1 ROS2使用
```bash
ros2 run --prefix 'strace -f -e trace=file' <包名> <节点名>
```



















