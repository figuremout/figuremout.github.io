---
title: Linux 内核添加自定义系统调用
tags:
- linux
- kernel
date: 2024-04-27
---

网上的教程[[1](#ref:1)]大多使用老版本内核，许多内容已经不再适用了。本文依托于我的项目 [mycall](https://github.com/figuremout/mycall) 进行讲解，旨在把自己踩过的坑全部记录下来，**具体实现请参考源码**。实验在 ubuntu 20.04 amd64 虚拟机（内核版本 5.15.0-105-generic）中进行，如有错误或建议（如不适用于 6.x 内核），欢迎在我的 [Github Page repo](https://github.com/figuremout/figuremout.github.io) 提 issue。

本文将先后介绍给内核添加自定义系统调用的两种方式：
- 通过内核模块将系统调用插入正在运行的内核中。
- 将系统调用添加到内核源码中，再重新编译安装内核。


# 通过内核模块添加系统调用
这个方法最简单但是坑也最多，因为内核开发组显然不希望我们通过内核模块修改/覆盖系统调用[[2](#ref:2)]，并且做了诸多限制，为此我们只能用一些 trick。

## 定义系统调用
从 Linux 4.17 开始，x86 下系统调用服务例程只接收 `struct pt_regs *` 一个参数[[9](#ref:9)]。因此系统调用的定义为如下形式：

```c
asmlinkage long sys_mycall(struct pt_regs *regs)
{
    ...
}
```

根据 Linux x86 calling convention，Linux 系统调用通过寄存器传递参数：`rax` 存储系统调用号，`rdi` 存储第一个参数, etc. `pt_regs` 就是一个包含了寄存器值的结构体[[10](#ref:10)]，需要从中读取参数。

## 获取系统调用表地址
要插入系统调用，首先需要能够找到系统调用表的地址 `sys_call_table`。可惜 2.6 版本以后内核就不再 export `sys_call_table` 了[[3](#ref:3)]，只能寻求其他办法：

- 从 System.map 读取

编译内核时生成的内核符号表中包含系统调用表的地址，可以通过以下命令获取：
```bash
cat /boot/System.map-$(uname -r) | grep sys_call_table
```

但是若内核开启了 KASLR (Kernel Address Space Layout Randomization)，实际地址会和 System.map 中记录的不同，如果非要用这个方法就得关掉内核的 KASLR 特性。

- 从 /proc/kallsyms 读取

其中包含当前运行内核的符号表，通过以下命令获取系统调用表地址：
```bash
sudo cat /proc/kallsyms | grep sys_call_table
```

但是每次主机重启该地址都会发生变化，所以不要把里面的地址硬编码到代码里，而是先手动执行上述命令获取系统调用表地址，再在 `insmod` 时通过 module param 传递进内核模块中。

然而更好的方式是让模块在加载时通过 `kallsyms_lookup_name()` 函数自动去获取系统调用表地址。这个函数可以在运行时查询到内核中所有符号的地址，包括 non-exported symbols 如 `sys_call_table`。但是模块绕过内核的 export system 去访问 non-exported symbols 很容易被滥用，所以从内核 5.7 开始不再 export 这个函数了[[4](#ref:4)]。不过这条路并没有被封死，我们还是可以通过 kprobes 来提取该地址。

- **放置 kprobes（推荐）**

利用 kprobes 可以追踪到 `kallsyms_lookup_name()` 函数的地址[[5](#ref:5), [6](#ref:6)]。其实也可以直接追踪 `sys_call_table`，不过根据 [[6](#ref:6)] 的描述，我这里还是先获取到 `kallsyms_lookup_name()` 函数，再利用该函数去查询系统调用表地址。以下是一个完整的 module 示例：
```C
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>

unsigned long *sys_call_table_addr;

// Find the address of sys_call_table address through kprobes
static struct kprobe kp = { .symbol_name = "kallsyms_lookup_name" };
typedef unsigned long (*kallsyms_lookup_name_t)(const char *name);
kallsyms_lookup_name_t kallsyms_lookup_name_my;

static int __init mymod_init(void) {
    register_kprobe(&kp);

    // pr_alert("Found at 0x%px \n", kp.addr);
    kallsyms_lookup_name_my = (kallsyms_lookup_name_t)kp.addr;
    sys_call_table_addr = (unsigned long *)kallsyms_lookup_name_my("sys_call_table");

    return 0;
}

static void __exit mymod_exit(void) {
    unregister_kprobe(&kp);
}

MODULE_LICENSE("GPL");

module_init(mymod_init);
module_exit(mymod_exit);
```

## 选择系统调用号
**注意：自定义 syscall 的调用号必须在范围内**。因为一旦内核编译完成后，其系统调用表大小已确定下来，如果在其后追加，很容易造成内存溢出问题，所以只能拦截替换现有的 syscall [[7](#ref:7), [8](#ref:8)]。

那么如何确定系统调用表的大小呢？首先需要下载当前运行内核的源码。
- 内核源码 arch/x86/entry/syscalls/syscall_64.tbl 中有定义的系统调用表。
- 编译内核后，arch/x86/include/generated/uapi/asm/unistd_64.h 中的 `__NR_syscalls` 就是系统调用表大小，这个宏只是表示这个表的大小，并不是真正的系统调用个数。
- 编译内核后，arch/x86/include/generated/asm/syscalls_64.h 中有完整的系统调用表（行数等于 `__NR_syscalls`，如果对应序号的系统调用不存在，那么就是初始值 `sys_ni_syscall`，表示没有实现的系统调用，调用该系统调用号直接返回错误码 `-ENOSYS`）。

可以选择在系统调用表长度范围内且没有定义的系统调用号（虽然也可以拦截已定义的系统调用，但是有造成系统不稳定的风险）作为我们系统调用的插入位置，如 `335`。

为了模块的灵活性，我选择将系统调用号作为 module param 传入，并且设置为可读：
```c
module_param(MYCALL_NUM, int, S_IRUGO); // perm: 0444 readable
```

如此一来，插入模块后测试程序就可以直接从 /sys/module/mymod/parameters/MYCALL_NUM 读取系统调用号，而不需要硬编码或手动传入。

## 插入系统调用
修改系统调用表需要关闭内存的写保护：
1. 将 cr0 的 Write Protect 关闭 (第17位是WP位)。
2. 修改系统调用表。
3. 恢复 cr0。

# 将系统调用编译进内核
我选择了和当前运行内核版本相近的 linux-5.15.157 版本内核源码进行实验。

1. 下载内核源码。
```bash
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.157.tar.xz
sudo tar -xvf linux-5.15.157.tar.xz -C /usr/src
```

2. 创建系统调用 sys_mycall。
```bash
cd /usr/src/linux-5.15.157
mkdir mycall
```

创建 mycall/mycall.c，包含系统调用的实现。**注意**：这里定义系统调用需要用 `SYSCALL_DEFINEx` 宏[[11](#ref:11), [12](#ref:12)]。
```C
#include <linux/kernel.h>
#incldue <linux/syscalls.h>

SYSCALL_DEFINE1(mycall, char __user *, buf)
{
        ...
        printk("Hello world\n");
        return 0;
}
```

创建 mycall/Makefile，内容如下：
```Makefile
obj-y:=mycall.o
```

3. 将 `mycall/` 添加到内核 Makefile 中 `core-y` 的末尾（6.x 内核变了，需要改 Kbuild [[13](#ref:13)]）。
```Makefile
core-y                  += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ mycall/
```

4. 将 sys_mycall 添加到系统调用表 arch/x86/entry/syscalls/syscall_64.tbl。
```
335     64      mycall                  sys_mycall
```

5. 将 sys_mycall 添加到头文件 include/linux/syscalls.h 末尾但 `#endif` 之前。
```C
...
asmlinkage long sys_mycall(char __user * buf);
#endif
```

6. 编译内核。

首先安装编译所需的包：
```bash
sudo apt-get install gcc
sudo apt-get install libncurses5-dev
sudo apt-get install bison
sudo apt-get install flex
sudo apt-get install libssl-dev
sudo apt-get install libelf-dev
sudo apt-get update
sudo apt-get upgrade
```

配置内核：由于我们需要使用 kprobes 和 kallsyms 特性，所以需要检查确认 `CONFIG_KPROBES=y`, `CONFIG_KALLSYMS=y`[[6](#ref:6)]。
```bash
# 保存到 .config
sudo make menuconfig
```

编译内核：
```bash
sudo make -j$(nproc)
```

7. 安装内核。
```bash
sudo make modules_install install
```
若重启后报错 `error: the initrd is too big` 无法进入系统，可以使用 `INSTALL_MOD_STRIP=1` 减小 initrd 大小：
```bash
sudo make INSTALL_MOD_STRIP=1 modules_install install
```

# References
<span id="ref:1"/>[1] https://medium.com/anubhav-shrimal/adding-a-hello-world-system-call-to-linux-kernel-dad32875872

<span id="ref:2"/>[2] https://lists.kernelnewbies.org/pipermail/kernelnewbies/2017-July/018091.html

<span id="ref:3"/>[3] https://unix.stackexchange.com/questions/424119/why-is-sys-call-table-predictable

<span id="ref:4"/>[4] https://lwn.net/Articles/813350/

<span id="ref:5"/>[5] https://github.com/xcellerator/linux_kernel_hacking/issues/3

<span id="ref:6"/>[6] https://stackoverflow.com/questions/70930059/proper-way-of-getting-the-address-of-non-exported-kernel-symbols-in-a-linux-kern

<span id="ref:7"/>[7] https://www.jianshu.com/p/a4ae5ec55732

<span id="ref:8"/>[8] https://stackoverflow.com/questions/2394985/linux-kernel-add-system-call-dynamically-through-module?rq=4

<span id="ref:9"/>[9] https://stackoverflow.com/a/72677965/24741118

<span id="ref:10"/>[10] https://www.torch-fan.site/2023/05/03/pt-regs%E5%B0%8F%E7%AC%94%E8%AE%B0/

<span id="ref:11"/>[11] https://stackoverflow.com/questions/66800646/unable-to-add-a-custom-hello-system-call-on-x64-ubuntu-linux

<span id="ref:12"/>[12] https://stackoverflow.com/questions/53735886/how-to-pass-parameters-to-linux-system-call

<span id="ref:13"/>[13] https://stackoverflow.com/questions/76262123/adding-a-system-call-to-linux-kernel-6
