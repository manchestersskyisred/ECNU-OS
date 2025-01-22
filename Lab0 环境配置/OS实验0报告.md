# 实验0 环境配置

## 吕佳鸿 10235501436

### 目的

本次实验是进行实验环境的搭建。实验环境主要由两个部分组成：QEMU用来模拟运行内核，以及一条编译工具链用于编译和测试内核。

### 流程

我的ubuntu虚拟机是arch架构，首先是xv6的依赖包安装，之后通过git安装安装RISC-V GNU编译器工具链，这一步用了很长时间，大约半个多小时（校园网下载实在慢）。接下来便是riscv-gnu-toolchain的依赖并编译，这一步时间更长（编译了快一个小时）。完事后安装qemu，并注意configure时要加上-disable-werror以防止因为编译时出现警告而停止编译。最后就是git clone xv6 并运行（这里注意把git协议换成https，否则clone不下来）。

### 问题

首先遇到的问题就是`./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="riscv64-softmmu"` 时会有包的缺失，这里按照系统报错的 `sudo apt install`相应的包就好

第二个是在安装完成后 我的riscv - gnu 工具链的版本是 13.2.0 qemu的版本是6.2.0 但是 

```shell
make qemu 

kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacyqemu-system-riscv64 -machine virt -bios none -kernel=false -drive file=fs.img,if=none,format=raw,id=x0evice virtio-blk-device,drive=xO,bus=virtio-mmio-bus.0
```

只输出到这里就会卡住 不会出现 `init starting : sh` ，参考了季子墨同学的建议 ，并查了一下 觉得可能是版本不兼容的问题，将qemu升级到9.1.0后 可以`make qemu` 成功

第三个问题是在升级到9.1.0时，需要将原有的包`rm -rf`掉 ，并把原有的路径也删除 ，再 `wget https://download.qemu.org/qemu-5.1.0.tar.xz` 下载再解压。并且也缺少依赖，如python3-venv 。需要install 并且还有一个ninja的包 在make时会显示报错 需要`sudo apt install` 并且将 make ,make install 改成 `sudo ninja -C build ,sudo ninja -C build install`(注意还是要configure)。ninja不要在cd后的目录下载，cd回主目录再下载

第四个就是为了适应实验条件根据助教介绍改回qemu.5.1.0  这里还是需要将原有的版本完全删干净再重新下载，并注意不要嵌套目录。这时cd到后clone的目录再make qemu即可成功

最后一个问题就是远程连接 我的ubuntu上缺少nettools 直接sudo install就行 `ifconfig , ip a`都可以返回ip地址 之后返回mac终端 `ssh username@ip地址`就可以远程连接 vscode里用相同的ip地址也可以远程连接

### Background补充

#### Risc-v 架构

RISC-V 是一个基于精简指令集（RISC）原则的全新开源指令集架构（ISA）。支持模块化可配置的指令子集， 采用了精简指令集的设计理念，专注于减少处理器需要执行的指令种类，从而简化硬件设计并提高处理效率。它的指令集简单、模块化，基础指令集很小，但可以通过增加标准或自定义扩展来适应不同的应用场景。

RISC-V架构提供31个用户可修改的通用(基本)寄存器，即x1到x31，以及一个额外的只读寄存器x0，硬连接到0。x0寄存器的一个常见用途是帮助将其他寄存器初始化为零。
• 共有31个通用寄存器。

• 其中7个是临时寄存器(t0−t6)。

• a0−a7用于函数参数。s0−s11用于保存寄存器或函数定义内。

• 一个堆栈指针，一个全局指针和一个线程指针寄存器。

• 一个返回地址寄存器(x1)，用于存储函数调用的返回地址。

• 一个程序计数器(pc)。PC保存着当前指令的地址。

#### QEMU

**QEMU** 是一款开源的仿真和虚拟化工具，它能够模拟各种不同的硬件平台和处理器架构，包括 x86、ARM、MIPS、PowerPC 和 RISC-V 等。它能够仿真一个完整的 RISC-V 系统，帮助开发者在没有物理硬件的情况下运行和测试 RISC-V 程序。特别是对于像 xv6 这样的操作系统项目，QEMU 提供了一种便捷的运行环境。

#### XV6

**xv6** 是一个简洁的 Unix 第六版（**V6 UNIX**）操作系统的现代化复刻版，它主要被用于操作系统教学。这个项目由 MIT 的计算机科学与人工智能实验室（CSAIL）开发，目的是帮助学生深入理解操作系统的基本原理。能够直接观察和实验操作系统的各个组件，比如进程管理、内存管理、文件系统和设备驱动等，提供了操作系统核心原理的清晰实现。





