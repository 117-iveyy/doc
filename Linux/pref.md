```table-of-contents
```
## 1 安装perf和火焰图
### 1.1 安装perf

- `perf` 是 Linux 内核工具，通常随内核源码或工具包发布，安装命令因发行版而异
```bash
sudo apt-get update
sudo apt-get install linux-tools-common linux-tools-generic  # 安装通用版本
# 若需匹配当前内核版本（推荐）：
sudo apt-get install linux-tools-$(uname -r)  # uname -r 查看内核版本
```
- 验证安装
```bash
perf --version
```
### 1.2 安装火焰图工具

- 火焰图工具由 Brendan Gregg 开发，需从 GitHub 克隆
```bash
git clone https://github.com/brendangregg/FlameGraph.git
# 工具位于FlameGraph目录下
```

## 2 使用perf收集性能数据

- 常用的有
    - 监控已运行的进程
    - 监控新启动的程序
    - 实时查看热点函数（用于快速定位）

### 2.1 监控已运行的进程
    
    - 若程序已启动，通过进程 ID（PID）监控
```bash
# 基本用法：-g 记录调用栈，-p 指定PID，-F 采样频率（Hz，默认4000，建议99避免干扰），-o 输出数据文件
sudo perf record -g -p 12345 -F 99 -o perf.data
```

### 2.2 监控新启动的程序
    
    - 直接运行程序并监控
```bash
# 监控 ./myprogram 程序，参数同上述<br>
sudo perf record -g -F 99 -o perf.data -- ./myprogram [程序参数]|
```
        
### 2.3 实时查看热点函数
    
```bash
sudo perf top -g  # -g 显示调用栈
```
        

## 3 处理perf数据并生成火焰图

上述`perf` 记录的 `perf.data` 需转换为火焰图工具可识别的格式，步骤如下

### 3.1 解析 `perf.data` 为文本格式
- 用perf script解析二进制数据为可读的调用栈文本
```bash
sudo perf script -i perf.data > perf.unfolded  # -i 指定输入文件，输出到perf.unfolded|
```

-  输出内容示例（每行是一个调用栈，包含函数名、进程、采样次数等）
```bash
docker    7684 665755.641732:         16 cycles:P:
        ffffffffb1c1c7d8 native_sched_clock+0x28 ([kernel.kallsyms])
        ffffffffb1c1c8d9 sched_clock_noinstr+0x9 ([kernel.kallsyms])
        ffffffffb1c1fc3e local_clock_noinstr+0xe ([kernel.kallsyms])
        ffffffffb0b965a5 local_clock+0x15 ([kernel.kallsyms])
        ffffffffb0d95c17 calc_timer_values+0x27 ([kernel.kallsyms])
        ffffffffb0da132b perf_event_update_userpage+0x5b ([kernel.kallsyms])
        ffffffffb0a1596a perf_ibs_start+0xea ([kernel.kallsyms])
        ffffffffb0a15abf perf_ibs_add+0x4f ([kernel.kallsyms])
        ffffffffb0d9f59e event_sched_in+0xde ([kernel.kallsyms])
        ffffffffb0da1d1a merge_sched_in+0x1da ([kernel.kallsyms])
        ffffffffb0da2081 visit_groups_merge.constprop.0.isra.0+0x1b1 ([kernel.kallsyms])
        ffffffffb0da2489 ctx_groups_sched_in+0x69 ([kernel.kallsyms])
        ffffffffb0da2588 ctx_sched_in+0xa8 ([kernel.kallsyms])
        ffffffffb0da27a1 __perf_event_task_sched_in+0x131 ([kernel.kallsyms])
        ffffffffb0b56706 finish_task_switch.isra.0+0x1b6 ([kernel.kallsyms])
        ffffffffb1c27e94 __schedule+0x284 ([kernel.kallsyms])
        ffffffffb1c282f3 schedule+0x33 ([kernel.kallsyms])
        ffffffffb0c281c6 futex_wait_queue+0x66 ([kernel.kallsyms])
        ffffffffb0c289a5 __futex_wait+0x155 ([kernel.kallsyms])
        ffffffffb0c28aa4 futex_wait+0x74 ([kernel.kallsyms])
        ffffffffb0c247bd do_futex+0x16d ([kernel.kallsyms])
        ffffffffb0c24f55 __x64_sys_futex+0x95 ([kernel.kallsyms])
        ffffffffb0a06a3b x64_sys_call+0x117b ([kernel.kallsyms])
        ffffffffb1c197f1 do_syscall_64+0x81 ([kernel.kallsyms])
        ffffffffb1e00130 entry_SYSCALL_64_after_hwframe+0x78 ([kernel.kallsyms])
            5c502c61e403 runtime.futex.abi0+0x23 (/usr/bin/docker)
            5c502c5b50e7 runtime.notesleep+0x87 (/usr/bin/docker)
            5c502c5e5eec runtime.stopm+0x8c (/usr/bin/docker)
            5c502c5e79bc runtime.findRunnable+0xd9c (/usr/bin/docker)
            5c502c5e8ab1 runtime.schedule+0xb1 (/usr/bin/docker)
            5c502c5e8f65 runtime.park_m+0x285 (/usr/bin/docker)
            5c502c61a5b0 runtime.mcall+0x50 (/usr/bin/docker)
            5c502c6144ce runtime.gopark+0xce (/usr/bin/docker)
            5c502c5caadf runtime.bgsweep+0xdf (/usr/bin/docker)
            5c502c5beee5 runtime.gcenable.gowrap1+0x25 (/usr/bin/docker)
            5c502c61c601 runtime.goexit.abi0+0x1 (/usr/bin/docker)

```

 - 折叠调用栈（关键步骤）
	- 火焰图需要将函数调用的同一路径进行合并。用 stackcollapse-perf.pl 处理
```bash
# 工具位于FlameGraph目录下，将perf.unfolded转换为折叠格式
./stackcollapse-perf.pl ../perf.unfolded > perf.folded
```
- 生成火焰图
	- 用 `flamegraph.pl` 将折叠后的文件转换为 SVG 图像：
```bash
./flamegraph.pl perf.folded > my_flamegraph.svg
```
- 生成的 `my_flamegraph.svg` 可通过浏览器打开
## 4 火焰图解读

- 火焰图是 SVG 格式的交互式图像，核心含义：
	- **Y 轴**：函数调用栈深度（从上到下是调用关系，上层函数调用下层）。
	- **X 轴**：所有采样的总时间（或次数），每个方块宽度表示函数在 CPU 上的耗时占比（越宽越耗时）。
    - **颜色**：无特殊含义，仅用于区分不同函数。
    - **交互**：鼠标悬停可查看函数名、耗时占比；点击可放大局部调用栈。

- 解读：若某函数方块特别宽，说明它是 CPU 热点，需优先优化（如减少循环次数、优化算法）。