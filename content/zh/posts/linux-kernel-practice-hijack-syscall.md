---
title: "Linux Kernel 实践(二)：劫持系统调用"
date: 2020-03-06T23:17:11+08:00
description:
draft: true
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: outer
tocLevels: ["h2", "h3", "h4"]
tags:
- Linux
- Kernel
- Module
series:
- Linux Kernel Practice
categories:
- 折腾
image:
---

通过劫持系统调用表，将原有系统调用替换成自定义系统调用。

<!--more-->

{{< alert theme="info" dir="ltr" >}}
使用系统为 Ubuntu，内核版本为 4.4.0-93-generic
{{< /alert >}}

{{< alert theme="warning" dir="ltr" >}}

劫持系统调用有风险，请不要在实体机上尝试。

{{< /alert >}}

## 前言

添加系统调用有两种方法

- 修改内核源代码，并重新编译内核

这种耗时耗力，比较麻烦，但是是在原有的系统调用中插入新的系统调用，不会出现冲突等问题。

- 通过内核模块重新映射系统调用地址

通过拦截系统调用表，将某个系统调用的地址修改成我们自定义的系统系统调用。

## 什么是系统调用表

在 Linux 中每个系统调用都有相应的系统调用号作为唯一的标识，内核维护一张系统调用表，`sys_call_table`。

在 64 位系统中，`sys_call_table` 定义在 [arch/x86/entry/syscall_64.c#L25](https://elixir.bootlin.com/linux/v4.4/source/arch/x86/entry/syscall_64.c#L25)

```c
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_64.h>
};
```

其中 `#include <asm/syscalls_64.h>` 为通过 [arch/x86/entry/syscalls/syscall_64.tbl](https://elixir.bootlin.com/linux/v4.4/source/arch/x86/entry/syscalls/syscall_64.tbl) 编译生成的，内容截取如下：

```c
__SYSCALL_COMMON(0, sys_read, sys_read)
__SYSCALL_COMMON(1, sys_write, sys_write)
```

所以 `sys_call_table` 展开如下：

```c
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    
	[0] = sys_read,
	[1] = sys_write,
	...
};
```

即 `sys_call_table` 是一个数组，索引为系统调用号，值为系统调用函数的起始地址。

## 获取 sys_call_table 地址

1. 通过 `/boot/System.map` 获取
2. 通过 `/proc/kallsyms` 获取
3. 通过暴力搜索获取



前面两种方式基本一致，都是通过读取文件并过滤的方式获取。

`/boot/System.map` 包含整个内核镜像的符号表。

`/proc/kallsyms` 不仅包含内核镜像符号表，还包含所有动态加载模块的符号表。

```bash
# /boot/System.map`
root@AliECS:~# cat /boot/System.map-$(uname -r) | grep sys_call_table
ffffffff81a001c0 R sys_call_table
ffffffff81a01520 R ia32_sys_call_table
# /proc/kallsyms`
root@AliECS:~# cat /proc/kallsyms | grep sys_call_table
ffffffff81a001c0 R sys_call_table
ffffffff81a01520 R ia32_sys_call_table
```

如果要在 LKM 中 使用的话，可以将地址写在宏定义上，再进行调用。

```c
#define SYS_CALL_TABLE  ffffffff81a001c0
```

但在不同的系统上都得进行修改并重新编译，非常麻烦。

### 暴力搜索

前面提到  `sys_call_table` 是一个数组，索引为系统调用号，值为系统调用函数的起始地址。

内核内存空间的起始地址 `PAGE_OFFSET` 变量和 `sys_close` 系统调用对我们是可见的（`sys_open ` / `sys_read`等并未导出）；系统调用号（即`sys_call_table`中的元素下标）在同一`ABI`（x86与x64属于不同ABI）中是高度后向兼容的；这个系统调用号我们也是可以直接引用的（如 `__NR_close` ）。

所以我们可以从内核空间起始地址开始，把每一个指针大小的内存假设成 `sys_call_table` 的地址，并用 `__NR_close` 索引去访问它的成员，如果这个值与 `sys_close` 的地址相同的话，就可以认为找到了 `sys_call_table` 的地址。

更多有关 `PAGE_OFFSET` 的内容请看：[ARM64 Linux 内核虚拟地址空间](https://geneblue.github.io/2017/04/02/ARM64 Linux 内核虚拟地址空间/)

下面来看搜索 `sys_call_table` 的函数：

`ULONG_MAX` 为 0xFFFFFFFFUL，即 `unsigned long` 的最大值

```c
unsigned long **get_sys_call_table(void)
{
  unsigned long **entry = (unsigned long **)PAGE_OFFSET;

  for (;(unsigned long)entry < ULONG_MAX; entry += 1) {
    if (entry[__NR_close] == (unsigned long *)sys_close) {
        return entry;
      }
  }

  return NULL;
}
```



## 劫持系统调用

### 写保护

写保护指的是写入只读内存时出错。 这个特性可以通过 [CR0](https://en.wikipedia.org/wiki/Control_register#CR0) 寄存器控制。

CR0 的第16位比特是写保护，设置时，即使权限级别为0（Linux 有4个权限级别，从0到3，0为最高级。等级0也被称为内核模式），CPU也不能写入只读页。

我们可以通过 [read_cr0](https://elixir.bootlin.com/linux/v4.4/ident/read_cr0) 和 [write_cr0](https://elixir.bootlin.com/linux/v4.4/ident/write_cr0) 这两个函数，可以读取和写入 CR0，同时通过 Linux 内核提供的接口 [set_bit](https://www.kernel.org/doc/htmldocs/kernel-api/API-set-bit.html) 和 [clear_bit](https://www.kernel.org/doc/htmldocs/kernel-api/API-clear-bit.html) 来操作比特。

关闭写保护，将第16个比特置为0。

```c
void disable_write_protection(void)
{
  unsigned long cr0 = read_cr0();
  clear_bit(16, &cr0);
  write_cr0(cr0);
}
```

开启写保护，将第16个比特置为1。

```c
void enable_write_protection(void)
{
  unsigned long cr0 = read_cr0();
  set_bit(16, &cr0);
  write_cr0(cr0);
}
```

### 模块代码

```c
/**
 * @file    nice.c
 * @author  WingLim
 * @date    2020-03-05
 * @version 0.1
 * @brief  读取及修改一个进程的 nice 值，并返回最新的 nice 值及优先级 prio 的模块化实现
*/

#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/pid.h>
#include <linux/sched.h>
#include <linux/syscalls.h>
#include <linux/uaccess.h>
#include <linux/unistd.h>

// 劫持 timer_gettime
#define the_syscall_num 224

MODULE_LICENSE("GPL");
MODULE_AUTHOR("WingLim");
MODULE_DESCRIPTION("A module to read or set nice value");
MODULE_VERSION("0.1");

asmlinkage long sys_setnice(pid_t pid, int flag, int niceval, int __user * prio, int __user * nice);
unsigned long **sys_call_table;
static int (*anything_saved)(void);

// 从内核起始地址开始搜索内存空间来获得 sys_call_table 的内存地址
unsigned long **get_sys_call_table(void)
{
  unsigned long **entry = (unsigned long **)PAGE_OFFSET;

  for (;(unsigned long)entry < ULONG_MAX; entry += 1) {
    if (entry[__NR_close] == (unsigned long *)sys_close) {
        return entry;
      }
  }
  return NULL;
}

// 禁用写保护
void disable_write_protection(void)
{
  unsigned long cr0 = read_cr0();
  clear_bit(16, &cr0);
  write_cr0(cr0);
}

// 恢复写保护
void enable_write_protection(void)
{
  unsigned long cr0 = read_cr0();
  set_bit(16, &cr0);
  write_cr0(cr0);
}
static int __init init_addsyscall(void)
{
    disable_write_protection();
    // 获取系统调用服务首地址
    sys_call_table = get_sys_call_table();
    // 保存原始系统调用的地址
    anything_saved = (int(*)(void)) (sys_call_table[the_syscall_num]);
    // 将原始的系统调用劫持为自定义系统调用
    sys_call_table[the_syscall_num] = (unsigned long*)sys_setnice;
    enable_write_protection();
    printk("hijack syscall success\n");
    return 0;
}

int get_prio(const struct task_struct *p)
{
        return p->prio - MAX_RT_PRIO;
}

asmlinkage long sys_setnice(pid_t pid, int flag, int nicevalue, int __user * prio, int __user * nice)
{
    struct pid * kpid;
        struct task_struct * task;
        int nicebef;
    int priobef;
        kpid = find_get_pid(pid); // 获取 pid
        task = pid_task(kpid, PIDTYPE_PID); // 返回 task_struct
        nicebef = task_nice(task); // 获取进程当前 nice 值
    priobef = get_prio(task); // 获取进程当前 prio 值

        if(flag == 1){
                set_user_nice(task, nicevalue);
                printk("nice value edit before：%d\tedit after：%d\n", nicebef, nicevalue);
                return 0;
        }
        else if(flag == 0){
                copy_to_user(nice, (const void*)&nicebef, sizeof(nicebef));
                copy_to_user(prio, (const void*)&priobef, sizeof(priobef));
                printk("nice of the process：%d\n", nicebef);
                printk("prio of the process：%d\n", priobef);
                return 0;
        }

        printk("the flag is undefined!\n");
        return EFAULT;
}

static void __exit exit_addsyscall(void) {
    disable_write_protection();
    sys_call_table[the_syscall_num] = (unsigned long*)anything_saved;
    enable_write_protection();
    printk("resume syscall\n");
}

module_init(init_addsyscall);
module_exit(exit_addsyscall);
```



#### 添加 `Makefile`

```makefile
obj-m+=hello.o
KDIR = /lib/modules/$(shell uname -r)/build

all:
    make -C $(KDIR) M=$(PWD) modules
clean:
    make -C $(KDIR) M=$(PWD) clean
```

#### 编译模块并启用

```bash
root@AliECS:~/dev/kernel/nice# make
make -C /lib/modules/4.4.0-93-generic/build/ M=/root/dev/kernel/nice modules
make[1]: Entering directory `/usr/src/linux-headers-4.4.0-93-generic'
  CC [M]  /root/dev/kernel/nice/nice.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /root/dev/kernel/nice/nice.mod.o
  LD [M]  /root/dev/kernel/nice/nice.ko
make[1]: Leaving directory `/usr/src/linux-headers-4.4.0-93-generic'

root@AliECS:~/dev/kernel/nice# insmod nice.ko
```



#### 模块测试代码

```c
/*test.c*/
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>
#define __NR_mysetnice 224 //系统调用号

int main() {
    pid_t tid;
    int nicevalue;
    int prio = 0;
    int nice = 0;
    tid = getpid();
    syscall(__NR_mysetnice,tid,0,-5,&prio,&nice);//read
    printf("pid: %d\nprio: %d\nnice: %d\n", tid, prio,nice);
    syscall(__NR_mysetnice,tid,1,-5,&prio,&nice);//set
    printf("pid: %d\nprio: %d\nnice: %d\n", tid, prio,nice);
    syscall(__NR_mysetnice,tid,0,-5,&prio,&nice);//read
    printf("pid: %d\nprio: %d\nnice: %d\n", tid, prio,nice);
    printf("*******************************\n");
    syscall(__NR_mysetnice,tid,0,-15,&prio,&nice);//read
    printf("pid: %d\nprio: %d\nnice: %d\n", tid, prio,nice);
    syscall(__NR_mysetnice,tid,1,-15,&prio,&nice);//set
    printf("pid: %d\nprio: %d\nnice: %d\n", tid, prio,nice);
    syscall(__NR_mysetnice,tid,0,-15,&prio,&nice);//read
    printf("pid: %d\nprio: %d\nnice: %d\n", tid, prio,nice);
    return 0;
}
```



#### 编译测试代码并测试

```bash
# 编译 test.c
root@AliECS:~/dev/kernel/nice# gcc test.c -o test
# 执行
root@AliECS:~/dev/kernel/nice# ./test
pid: 12872
prio: 20
nice: 0
pid: 12872
prio: 20
nice: 0
pid: 12872
prio: 15
nice: -5
*******************************
pid: 12872
prio: 15
nice: -5
pid: 12872
prio: 15
nice: -5
pid: 12872
prio: 5
nice: -15
# 查看模块输出信息
root@AliECS:~/dev/kernel/nice# tail /var/log/kern.log
Mar  7 03:52:47 AliECS kernel: [118009.435431] nice of the process：0
Mar  7 03:52:47 AliECS kernel: [118009.435434] prio of the process：20
Mar  7 03:52:47 AliECS kernel: [118009.435466] nice value edit before：0	edit after：-5
Mar  7 03:52:47 AliECS kernel: [118009.435475] nice of the process：-5
Mar  7 03:52:47 AliECS kernel: [118009.435476] prio of the process：15
Mar  7 03:52:47 AliECS kernel: [118009.435481] nice of the process：-5
Mar  7 03:52:47 AliECS kernel: [118009.435481] prio of the process：15
Mar  7 03:52:47 AliECS kernel: [118009.435485] nice value edit before：-5	edit after：-15
Mar  7 03:52:47 AliECS kernel: [118009.435494] nice of the process：-15
Mar  7 03:52:47 AliECS kernel: [118009.435495] prio of the process：5
```



## 参考

- [Linux系统调用流程](https://zhuanlan.zhihu.com/p/55214097)
- [Linux Rootkit 系列二：基于修改 sys_call_table 的系统调用挂钩](https://docs-conquer-the-universe.readthedocs.io/zh_CN/latest/linux_rootkit/sys_call_table.html)
- [Kernel Mode Hooking Tutorial](https://github.com/oblique/articles/blob/master/kernel_mode_hooking/tutorial.english.txt)
- [OS 实验一 | linux 内核编译及添加系统调用](https://zhuanlan.zhihu.com/p/31342840)