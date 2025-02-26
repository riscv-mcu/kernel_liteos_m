# Nuclei EvalSoC LiteOS移植使用说明

## Nuclei FPGA开发板简介

> [!NOTE]
>
> - 这个是用来测试Nuclei EvalSoC在LiteOS-M上面的移植。
> - LiteOS-M目前仅支持32位CPU，因此只能测试我们的32位的RISC-V CPU，例如n200/n300/n600/n900系列
> - 支持的Nuclei RISC-V CPU请务必配置ECLIC中断控制器

关于Nuclei FPGA开发板，请参见

- 目前可用的包括 DDR200T/KU060等 https://nucleisys.com/developboard.php
- 支持的是Nuclei EvalSoC, 专门用于Nuclei RISC-V CPU评估，详细参见 https://doc.nucleisys.com/nuclei_sdk/design/soc/evalsoc.html

## 文件结构

```
├── components                            # 可选组件
│   ├── cppsupport                        # C++支持
│   └── cpup                              # CPUP功能
├── kal                                   # 内核抽象层
│   └── posix                             # posix标准接口支持
├── kernel                                # 内核最小功能集支持
│   ├── arch                              # 内核指令架构层代码
│   │   ├── risc-v                        # risc-v架构的代码
│   │   │   └── nuclei                    # nuclei内核相关代码
│   │   │       └── gcc                   # gcc编译器相关代码
│   │   │           ├── nmsis             # nmsis内核标准
│   │   └── include                       # 对外接口存放目录
│   │       ├── los_arch_atomic.h         # 定义通用arch的原子操作
│   │       ├── los_arch_context.h        # 定义通用arch的上下文切换
│   │       ├── los_arch.h                # 定义通用arch初始化
│   │       └── los_arch_interrupt.h      # 定义通用arch中断
│   ├── include                           # 对外接口存放目录
│   │   ├── los_config.h                  # 功能开关和配置参数
│   │   ├── los_event.h                   # 事件
│   │   ├── los_liteos.h                  # liteos最小功能集对外提供的头文件
│   │   ├── los_memory.h                  # 堆内存管理
│   │   ├── los_mutex.h                   # 互斥锁
│   │   ├── los_queue.h                   # 队列
│   │   ├── los_scheduler.h               # 调度算法
│   │   ├── los_sem.h                     # 信号量
│   │   ├── los_task.h                    # 任务
│   │   └── los_timer.h                   # 定时器
│   └── src                               # 内核最小功能集源码
├── targets                               # 板级工程目录
│   ├── riscv_nuclei_eval_soc_gcc         # Nuclei Evalsoc相关代码
│       ├── GCC                           # 编译相关
│       ├── OS_CONFIG                     # 开发板配置功能开关和配置参数
│       ├── SoC                           # SOC相关代码
│       └── Src                           # application相关代码
├── utils                                 # 通用公共目录
    ├── include
    │   ├── los_compiler.h                # 编译工具配置，类型定义
    │   ├── los_debug.h                   # debug，printf相关
    │   ├── los_error.h                   # 错误定义
    │   └── los_list.h
    └── src
```

## 使用说明

软件需求：Ubuntu Linux环境

硬件需求：Nuclei EvalSoC bitstream

本示例将新建并运行两个任务，可以在控制台查看任务执行打印信息。

### 环境配置

- **工具链配置**

请直接下载最新的Nuclei Studio 2025.02 Linux版本，参见 https://nucleisys.com/download.php#tools

解压IDE以后，预装的gcc/openocd/qemu均位于 ``/path/to/NucleiStudio/toolchain``目录下，

关于 Nuclei Tools的使用文档，请阅读 https://doc.nucleisys.com/nuclei_tools/index.html

只需要设置好系统PATH即可，参考如下

~~~shell
# 解压下载好的Nuclei Studio IDE到你的开发环境下
# TODO: 请修改这个NUCLEI_TOOL_ROOT的真实路径
export NUCLEI_TOOL_ROOT=/path/to/NucleiStudio/toolchain
export PATH=$NUCLEI_TOOL_ROOT/gcc/bin:$NUCLEI_TOOL_ROOT/openocd/bin:$NUCLEI_TOOL_ROOT/qemu/bin:$NUCLEI_TOOL_ROOT/nucleimodel/bin:$PATH
~~~

> 注意：
>
> - 请务必下载并解压缩Linux版本的工具，不要下载windows版本工具。

- **驱动配置**

关于蜂鸟调试器的驱动安装，请参见 https://doc.nucleisys.com/nuclei_tools/ide/projectrun.html#linux


### 编译源码

> 为了方便编译，我们将编译缺失的 securec.h 头文件及c代码 全部拷贝到了 ``components/bounds_checking_function`` 目录下，并修改了Makefile.
> 这样通过make命令就可以直接构建了。

编译前请务必设置好工具链配置，确保gcc/openocd等工具在环境PATH中，测试命名为

~~~shell
riscv64-unknown-elf-gcc -v
openocd -v
~~~

配置路径后打开至代码根目录下的 ``target/riscv_nuclei_eval_soc_gcc/GCC`` 位置，输入如下指令开始编译：

```
make all
```

编译结束后部分参考输出如下：

```
riscv64-unknown-elf-size build/nuclei_evalsoc.elf
   text    data     bss     dec     hex filename
   41840    4864   41964   88668   15a5c build/nuclei_evalsoc.elf
riscv64-unknown-elf-objdump -D -S build/nuclei_evalsoc.elf > build/nuclei_evalsoc.dasm
riscv64-unknown-elf-objcopy -O ihex build/nuclei_evalsoc.elf build/nuclei_evalsoc.hex
riscv64-unknown-elf-objcopy -O binary -S build/nuclei_evalsoc.elf build/nuclei_evalsoc.bin
```

若编译前想清理工程，请使用如下指令：

```
make clean
```

### 下载调试、运行

调试或运行前请先使用蜂鸟调试器连接Nuclei FPGA开发板，确保已按照**环境配置**中驱动配置部分配置完成。

并且确保 bitstream 使用的是支持的Nuclei EvalSoC的bitstream（RV32）并且已经通过FPGA下载工具下载到FPGA开发板上。

且务必按照**工具链配置**流程配置好环境，并打开至代码根目录下的 ``target/riscv_nuclei_eval_soc_gcc/GCC`` 位置，

首先检查openocd是否能够通过蜂鸟调试器连接到开发板上：

~~~shell
make run_openocd
# 上面的命令运行后，应该可以看到类似如下的输出，且没有自动结束openocd
## Info : Examined RISC-V core; found 1 harts
## Info :  hart 0: XLEN=32, misa=0x4014952f
## [riscv.cpu] Target successfully examined
## ... ...
## Info : Listening on port xxxx for telnet connections
# 如果成功连接，则结束当前命令即可
# 如果看到其他的输出，则大概率是FPGA没有刷上bitstream，调试器驱动没装好等问题
~~~


如果前面openocd可以正常连接开发板，则说明CPU是正常的，然后请通过``CTRL-C``结束上面的命令
若想直接运行，请在调试时所在位置输入如下指令：

```shell
# 请确保openocd能连上开发板，以确保下载能够顺利
make upload
```

运行时可以查看串口打印内容，使用串口查看工具，这里以`minicom`为例，若未安装此工具可自行安装或使用其他串口查看工具。打开控制台，输入`minicom -D /dev/ttyUSB1 -b 115200`指令打开串口查看工具。

运行时参考输出如下：

```
Nuclei SDK Build Time: Jul 23 2025, 17:29:53
Download Mode: ILM
CPU Frequency 15998996 Hz
entering kernel init...
Entering scheduler
TaskSampleEntry1 running...
TaskSampleEntry2 running...
TaskSampleEntry2 running...
TaskSampleEntry1 running...
TaskSampleEntry2 running...
TaskSampleEntry2 running...
```

如果需要进行调试，则输入如下指令进入GDB调试：

```shell
make debug
# 等待到进入GDB调试界面时，请按照下面的步骤进行程序下载和调试
# 如果需要复位CPU和重新下载代码，从程序入口开始调试则执行如下命令
monitor reset halt
load
set disassemble-next-line on
# 按指令单步执行 si 按c语言单步执行 s
# 在main函数打断点 b main

# 直接运行 continue
# 如果不需要下载代码，只是连上去进行调试
# 则连上去以后，就是当前pc所在位置
info reg pc # 查看当前pc
# 查看backtrace 使用 bt
```

如果想要在 Nuclei Qemu上正常使用，则必须使用 Nuclei Qemu 2025.02以后的版本才行，当前版本还有问题，测试方法如下

可以临时使用修正版本的qemu，参见 https://gitee.com/link?target=https%3A%2F%2Fdrive.weixin.qq.com%2Fs%3Fk%3DABcAKgdSAFchHwEAaD

~~~shell
$ qemu-system-riscv32 --version
QEMU emulator version 9.0.4 (v9.0.4-96-gfca572fde0)
Copyright (c) 2003-2024 Fabrice Bellard and the QEMU Project developers
$ qemu-system-riscv32 -M nuclei_evalsoc,download=ilm -cpu nuclei-n300fd,ext= -smp 1 -icount shift=0 -nodefaults -nographic -serial stdio -kernel build/nuclei_evalsoc.elf
Nuclei SDK Build Time: Jul 23 2025, 17:29:53
Download Mode: ILM
CPU Frequency 999997767 Hz
entering kernel init...
Entering scheduler
TaskSampleEntry1 running...
TaskSampleEntry2 running...
TaskSampleEntry2 running...
TaskSampleEntry1 running...
TaskSampleEntry2 running...
TaskSampleEntry2 running...
~~~

也可以在 Nuclei Near Cycle Model上测试使用，运行方法如下

~~~shell
$ xl_cpumodel --cpu n300fd --download=ilm build/nuclei_evalsoc.elf

        SystemC 2.3.4-Accellera --- May 16 2024 16:02:03
        Copyright (c) 1996-2022 by all Contributors,
        ALL RIGHTS RESERVED
[XLMODEL-INFO] filename[0]: build/nuclei_evalsoc.elf
[XLMODEL-INFO] Executable segment 0: buf=0x557a8d0ee150, vaddr=0x80000000, paddr=0x80000000, len=0xa380
[XLMODEL-INFO] Data segment 0: buf=0x557a8d0f84e0, vaddr=0x90000000, paddr=0x90000000, len=0x1300
[XLMODEL-INFO] Executable profiling segments: lowpc=0x80000000, highpc=0x8000a380
[XLMODEL-INFO] Found the following .elf files:
[XLMODEL-INFO] build/nuclei_evalsoc.elf
[XLMODEL-INFO] Created Cluster0
[XLMODEL-INFO] start pc: 0x80000100
[XLMODEL-INFO] rv32 file isa: rv32imafdc
[XLMODEL-INFO] argv[0]: -t
[XLMODEL-INFO] argv[1]: -p1
[XLMODEL-INFO] argv[2]: -m0x70000000:0x90000000,0x20000000:0x10000000,0x60000000:0x8000000,0x10014000:0xc000,0x14000000:0x100000
[XLMODEL-INFO] argv[3]: +permissive-off
[XLMODEL-INFO] argv[4]: build/nuclei_evalsoc.elf
Nuclei SDK Build Time: Jul 23 2025, 17:29:53
Download Mode: ILM
CPU Frequency 1654456 Hz
CPU HartID: 0
entering kernel init...
Entering scheduler
TaskSampleEntry1 running...
TaskSampleEntry2 running...
TaskSampleEntry2 running...
TaskSampleEntry1 running...
TaskSampleEntry2 running...
TaskSampleEntry2 running...
~~~
