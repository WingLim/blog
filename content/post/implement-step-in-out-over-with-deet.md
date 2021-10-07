---
title: "CS 110L: DEET 实现 step in, out 和 over"
author: WingLim
date: 2021-10-07T15:14:40+08:00
slug: cs-110l-implement-step-in-out-over-with-deet
tags:
- Debugger
- Rust
categories:
- Tech
---

DEET(Dodgy Eliminator of Errors and Tragedies) 是 CS 100L 的 [Project1](https://reberhardt.com/cs110l/spring-2020/assignments/project-1/)

在完成 7 个 Milestone 后，DEET 还提供了一些可选扩展，这篇文章用来记录实现 `Next Line` 的几个命令：

1. step in - `step`
2. step out - `finish`
3. step over - `next`

完整代码请看: [deet]([https://github.com/WingLim/cs110l-spr-2020-solution/tree/main/proj-1/deet](https://github.com/WingLim/cs110l-spr-2020-solution/tree/main/proj-1/deet))

## 打印源码

在开始之前先实现打印源码，方便后续查看当前执行到了哪一步

通过 `debug_data.get_line_from_addr()` 这个方法可以获取到当前执行的行信息

然后再读取源代码文件并打印

```rust
pub fn print_source(&self, line: &Line) {
    let path = line.file.clone();
    if let Ok(source) = fs::read_to_string(path) {
        let line_content = source.lines().nth(line.number - 1).unwrap();

        println!("\n{}{}", line.number, line_content);
    }
}
```

## 辅助函数

为了方便后续实现，抽象出几个辅助函数

```rust
/// 跳过断点
/// 1. 恢复 instruction 内容
/// 2. 修改 rip 指针到断点开始前
/// 3. 执行 instruction
/// 4. 恢复 断点
#[allow(mutable_borrow_reservation_conflict)]
fn step_over_breakpoint(&mut self) {
    let mut regs = ptrace::getregs(self.pid()).unwrap();
    let rip = self.get_rip().unwrap();
    if let Some(orig_byte) = self.breakpoints.get(&(rip - 1)) {
        self.write_byte(rip - 1, *orig_byte).unwrap();
        regs.rip = (rip - 1) as u64;
        ptrace::setregs(self.pid(), regs).unwrap();
        ptrace::step(self.pid(), None).unwrap();
        self.wait(None).unwrap();
        self.write_byte(rip - 1, 0xcc).unwrap();
    }
}

/// 获取当前 rip
fn get_rip(&self) -> Result<usize, nix::Error> {
    let regs = ptrace::getregs(self.pid())?;
    Ok(regs.rip as usize)
}

/// 单步执行 instruction
/// 1. 存在断点则跳过
/// 2. 不存在则直接单步执行
fn single_step_instruction(&mut self) {
    if self.breakpoints.contains_key(&self.get_rip().unwrap()) {
        self.step_over_breakpoint()
    } else {
        ptrace::step(self.pid(), None).unwrap();
        self.wait(None).unwrap();
    }
}
```

## step in

`step in` 原理比较简单，循环执行 `ptrace::step` 直到进入下一行代码

需要注意的是 `get_line_from_addr` 返回的是 `<Option<Line>>` ，在执行过程中可能会跳转到例如 `printf` 的代码中，此时在源码中没有对应的行，所以会返回 `None`。

因此我们需要使用 `unwrap_or()` 来处理这种情况

```rust
pub fn step_in(&mut self, debug_data: &DwarfData) {
    let line = debug_data.get_line_from_addr(self.get_rip().unwrap()).unwrap();

    while debug_data.get_line_from_addr(self.get_rip().unwrap()).unwrap_or(Line {
        file: "".to_string(),
        number: line.number,
        address: 0
    }).number == line.number {
        self.single_step_instruction();
    }

    let line_entry = debug_data.get_line_from_addr(self.get_rip().unwrap()).unwrap();
    self.print_source(&line_entry);
}
```

## step out

`step out` 则是在函数的返回地址处添加一个断点，通过 `ptrace::cont` 继续执行到该断点，再删除断点，即可跳出函数

返回地址存储在 `rbp + 8`，这里注意的是，我们要读取 `rbp + 8` 到 `rbp` 之间的内容，才是返回地址

```rust
pub fn step_out(&mut self) -> Result<Status, nix::Error> {
    let regs = ptrace::getregs(self.pid()).unwrap();
    let rbp = regs.rbp;
    let return_address = ptrace::read(self.pid(), (rbp + 8) as ptrace::AddressType).unwrap() as usize;

    let mut should_remove_breakpoint = false;
    if !self.breakpoints.contains_key(&return_address) {
        self.set_breakpoint(return_address);
        should_remove_breakpoint = true
    }

    let status = self.continue_run()?;

    if should_remove_breakpoint {
        self.remove_breakpoint(return_address);
    }

    Ok(status)
}
```

## step over

`step over` 则要复杂一些，我们很难知道下一行的地址，比如存在 `for` 循环这种需要来回跳转的

简单的做法就是先给当前函数的所有行和返回地址都添加断点，使用 `ptrace::cont` 继续执行到下一行，然后删除刚刚添加的断点

这里我们需要给 `debug_data` 添加一个新的函数 `get_function`，用于获取我们当前处在哪个函数中

```rust
pub fn get_function(&self, curr_addr: usize) -> Option<Function> {
    for file in &self.files {
        for func in &file.functions {
            if func.address <= curr_addr && (func.address + func.text_length) >= curr_addr {
                return Some(func.clone());
            }
        }
    }
    None
}
```

然后添加 `step_over` 函数到 `inferior`

```rust
pub fn step_over(&mut self, debug_data: &DwarfData) -> Result<Status, nix::Error> {
    // 获取当前所处的函数
    let func = debug_data.get_function(self.get_rip().unwrap()).unwrap();
    let func_entry = func.address;
    let func_end = func.address + func.text_length;

    // 获取函数入口行
    let line = debug_data.get_line_from_addr(func_entry).unwrap();
    let mut line_number = line.number;
    let mut load_address = line.address;
    // 获取当前行
    let start_line = debug_data.get_line_from_addr(self.get_rip().unwrap()).unwrap();
    // 用于存储临时断点
    let mut to_delete = Vec::new();

    while load_address < func_end {
        // 给每一行设置断点
        if load_address != start_line.address && !self.breakpoints.contains_key(&load_address) {
            self.set_breakpoint(load_address);
            to_delete.push(load_address);
        }
        line_number += 1;
        // 获取下一行的地址
        load_address = debug_data.get_addr_for_line(None, line_number).unwrap();
    }

    // 给返回地址添加断点
    let regs = ptrace::getregs(self.pid())?;
    let rbp = regs.rbp;
    let return_address = ptrace::read(self.pid(), (rbp + 8) as ptrace::AddressType).unwrap() as usize;
    if !self.breakpoints.contains_key(&return_address) {
        self.set_breakpoint(return_address);
        to_delete.push(return_address);
    }

    let status = self.continue_run()?;

    // 删除刚刚添加的断点
    for addr in to_delete {
        self.remove_breakpoint(addr);
    }

    Ok(status)
}
```

## 感谢

- [https://blog.tartanllama.xyz/writing-a-linux-debugger-dwarf-step/](https://blog.tartanllama.xyz/writing-a-linux-debugger-dwarf-step/)