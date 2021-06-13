---
title: "Linux Kernel 实践(一)：Hello LKM"
author: WingLim
date: 2020-03-06T18:18:09+08:00
slug: linux-kernel-practice-hello
description:
draft: false
tags:
- Linux
- Kernel
- Module
categories:
- Learn
---

实现一个简单的 Linux Kernel Module 并通过自定义参数输出信息。

<!--more-->
使用系统为 Ubuntu，内核版本为 4.4.0-93-generic

## 什么是内核模块

Loadable Kernel Modules（LKM）即可加载内核模块，LKM可以动态地加载到内存中，无须重新编译内核。所以它经常被用于一些设备的驱动程序，例如声卡，网卡等等。

内核模块和一般的 C 语言程序不同，它不使用 `main()` 函数作为入口，并且有如下区别：

- 非顺序执行：内核模块使用初始化函数来进行注册，并处理请求，初始化函数运行后就结束了。 它可以处理的请求类型在模块代码中定义。
- 没有自动清理：内核模块申请的所有内存，必须要在模块卸载时手动释放，否则这些内存会无法使用，直到重启，也就是说我们需要在模块的卸载函数（也就是下文写到的退出函数）中，将使用的内存逐一释放。
- 会被中断：内核模块可能会同时被多个程序/进程使用，构建内核模块时要确保发生中断时行为一致和正确。想了解更多请看：[Linux 内核的中断机制](https://www.cnblogs.com/linfeng-learning/p/9512866.html)
- 更高级的执行特权：通常分配给内核模块的CPU周期比分配给用户空间程序的要多。编写内核模块时要小心，以免模块对系统的整体性能产生负面影响。
- 不支持浮点：在Linux内核里无法直接进行浮点计算，因为这样做可以省去在用户态与内核态之间进行切换时保存/恢复浮点寄存器 FPU的操作。

## 构建前的准备

通过包管理安装 Linux 内核头文件

```bash
$ sudo apt update
$ apt-cache search linux-headers-$(uname -r)
$ apt install linux-headers-$(uname -r)
```



## 开始写代码

### 引入头文件

```c
#include <linux/init.h> // 用于标记函数的宏
#include <linux/module.h> //加载内核模块到内核使用的核心头文件
#include <linux/kernel.h> // 包含内核使用的类型、宏和函数
```

### 定义模块信息

```c
MODULE_LICENSE("GPL");              // 许可类型，它会影响到运行时行为 
MODULE_AUTHOR("WingLim");      // 作者，当使用 modinfo 命令时可见 
MODULE_DESCRIPTION("A simple Linux driver to say hello.");  // 模块描述，参见 modinfo 命令 
MODULE_VERSION("0.1");              // 模块版本 
```

如果没有定义 `MODULE_LICENSE` ，在编译和加载模块时会报 `WARNING: modpost: missing MODULE_LICENSE()`

`MODULE_LICENSE` 可以选用 “GPL”，“GPL v2”，“GPL and additional rights”，“Dual BSD/GPL”，“Dual MPL/GPL”，“Proprietary” 这几个许可证。更多说明请看：[linux/module.h#L209](https://elixir.bootlin.com/linux/v4.4/source/include/linux/module.h#L209)

### 初始化函数

`static` 限制这个函数的可见范围为当前 C 文件

`__init` 表示该函数仅在初始化阶段使用，之后释放使用的内存资源：[init.h#L7](https://elixir.bootlin.com/linux/v4.4/source/include/linux/init.h#L7)

`@return` 执行成功返回 0

在内核中我们使用 `printk()` 来打印信息.。`printk()` 和 `printf()` 语法一样，但需要先定义消息类型。可用的消息类型可以到 [linux/kern_levels.h#L7-#L23](https://elixir.bootlin.com/linux/v4.4/source/include/linux/kern_levels.h#L7) 查看

```c
static int __init helloModule_init(void){
   printk(KERN_INFO "Hello LKM!\n");
   return 0;
}
```

### 退出函数

`__exit` 表示如果这个代码用于一个内置的驱动程序(而不是LKM)，则不需要这个函数。

```c
static void __exit helloModule_exit(void){
   printk(KERN_INFO "Goodbye LKM!\n");
}
```

### 初始化&退出模块

定义在 [linux/module.h#L75-#L98](https://elixir.bootlin.com/linux/v4.4/source/include/linux/module.h#L75)

```c
module_init(helloModule_init);
module_exit(helloModule_exit);
```

#### 汇总

```c
/*hello.c*/
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("WingLim");
MODULE_DESCRIPTION("A simple Linux driver to say hello.");
MODULE_VERSION("0.1");

static int __init helloModule_init(void){
   printk(KERN_INFO "Hello LKM!\n");
   return 0;
}

static void __exit helloModule_exit(void){
   printk(KERN_INFO "Goodbye LKM!\n");
}

module_init(helloModule_init);
module_exit(helloModule_exit);
```



## 编译

添加 `Makefile`

```makefile
obj-m+=hello.o
KDIR = /lib/modules/$(shell uname -r)/build

all:
    make -C $(KDIR) M=$(PWD) modules
clean:
    make -C $(KDIR) M=$(PWD) clean
```

注意：`Makefile` 的基本语法如下，如果缩进不是 `<TAB>` 的话，会报错。

```
<target>: [ <dependency > ]*
       [ <TAB> <command> <endl> ]+
```



```bash
# 查看当前文件
root@0xDayServer:~/dev/kernel/hello# ls -l
total 8
-rw-r--r-- 1 root root 466 Mar  6 22:53 hello.c
-rw-r--r-- 1 root root 154 Mar  6 22:54 Makefile

# 编译
root@0xDayServer:~/dev/kernel/hello# make
make -C /lib/modules/4.4.0-93-generic/build/ M=/root/dev/kernel/hello modules
make[1]: Entering directory `/usr/src/linux-headers-4.4.0-93-generic'
  CC [M]  /root/dev/kernel/hello/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /root/dev/kernel/hello/hello.mod.o
  LD [M]  /root/dev/kernel/hello/hello.ko
make[1]: Leaving directory `/usr/src/linux-headers-4.4.0-93-generic'

# 编译后生成的模块文件
root@0xDayServer:~/dev/kernel/hello# ls
hello.c  hello.ko  hello.mod.c  hello.mod.o  hello.o  Makefile  modules.order  Module.symvers
```



## 测试模块

### 查看模块信息

```bash
root@0xDayServer:~/dev/kernel/hello# modinfo hello.ko
filename:       /root/dev/kernel/hello/hello.ko
version:        0.1
description:    A simple Linux driver to say hello.
author:         WingLim
license:        GPL
srcversion:     093C7851C912088AEE5F77C
depends:        
vermagic:       4.4.0-93-generic SMP mod_unload modversions
```

### 加载模块

```bash
root@0xDayServer:~/dev/kernel/hello# insmod hello.ko
root@0xDayServer:~/dev/kernel/hello# lsmod
Module                  Size  Used by
hello                  16384  0 
```

### 卸载模块

```bash
root@0xDayServer:~/dev/kernel/hello# rmmod hello
```



### 查看 printk() 输出信息

1. 使用 `dmesg` 命令

```bash
root@0xDayServer:~/dev/kernel/hello# dmesg
[100339.744628] Hello LKM!
[100432.211044] Goodbye LKM!
```

2. 查看内核日志

```bash
root@0xDayServer:~/dev/kernel/hello# tail /var/log/kern.log
Mar  6 22:58:16 0xDayServer kernel: [100339.744628] Hello LKM!
Mar  6 22:59:49 0xDayServer kernel: [100432.211044] Goodbye LKM!
```



## 自定义参数

将 `hello.c` 修改如下

```c
/*hello.c*/
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("WingLim");
MODULE_DESCRIPTION("A simple Linux driver to say hello.");
MODULE_VERSION("0.1");

static char *name = "LKM";
module_param(name, charp, S_IRUGO);
MODULE_PARM_DESC(name, "The name to display in /var/log/kern.log");

static int __init helloModule_init(void){
   printk(KERN_INFO "Hello %s!\n", name);
   return 0;
}

static void __exit helloModule_exit(void){
   printk(KERN_INFO "Goodbye %s!\n", name);
}

module_init(helloModule_init);
module_exit(helloModule_exit);
```



### 解析

#### `static char *name = "LKM";`

声明了一个全局静态字符指针变量 `name`，默认值为`"LKM"`

在内核模块中应该尽量避免使用全局变量，因为全局变量会被整个内核共享。所以应该使用 `static` 来限制变量在模块中的作用域，如果一定要使用全局变量的话，最好给这个变量加上前缀，以确保它在内核中是唯一的。

#### `module_param(name, type, permissions)` 

定义在 [linux/moduleparam.h#L125](https://elixir.bootlin.com/linux/v4.4/source/include/linux/moduleparam.h#L125)

`name` 名字：向用户显示的参数名称和模块中的变量名称

`type` 参数类型：byte, short, ushort, int, uint, long, ulong, charp, bool, invbool

`permissions` 权限：值为 `0` 时，禁用该项，`0444` 所有人可读，`0644` root用户可写，这里的写法和文件权限一致。

#### MODULE_PARM_DESC

参数描述，会显示在 `modinfo` 中

### 调用

```bash
root@0xDayServer:~/dev/kernel/hello# insmod hello.ko name=World
root@0xDayServer:~/dev/kernel/hello# dmesg
[103386.179203] Hello World!
```



## 参考

- [Writing a Linux Kernel Module — Part 1: Introduction](http://derekmolloy.ie/writing-a-linux-kernel-module-part-1-introduction/)
- [linux 下的浮点运算](http://abcdxyzk.github.io/blog/2018/01/08/kernel-fpu-1/)
- [Linux Device Drivers : 3rd Edition](https://book.douban.com/subject/1493443/)