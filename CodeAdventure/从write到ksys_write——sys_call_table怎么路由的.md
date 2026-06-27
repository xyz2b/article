# 从 write(1, "hello", 5) 到 ksys_write() —— sys_call_table 怎么路由的

---

> **🎯 交互式可视化**：[→ syscall-table-visualizer.html](https://xyz2b.github.io/article/CodeAdventure/front/syscall-table-visualizer.html)  
> 内核拿到系统调用号后怎么路由的全过程动画：看系统调用号 `64` 如何索引 `sys_call_table`、跳到 `__arm64_sys_write`，再一路走到 `ksys_write()`。

---

> **系列说明**：这是"一句 printf 怎么出现在屏幕上"系列的第四篇，也是系统调用之旅的收尾。前三篇分别讲了：第一篇《[printf("hello") 怎么变成 write(1, "hello", 5)](https://github.com/xyz2b/article/blob/main/CodeAdventure/printf怎么变成write——libc的stdout缓冲机制.md)》讲清 libc 的 stdout 缓冲，把数据交到 `write()` 系统调用手上；第二篇《`svc` / `syscall` 指令到底做了什么——读 MSR、切换 CPL、换栈》讲清用户态 → 内核态那道硬件边界（待发布）；第三篇《内核入口 `el0_svc` / `entry_SYSCALL_64` 的汇编做了什么》讲清怎么保存寄存器、建立 `struct pt_regs`、搭好内核上下文（待发布）。这一篇把前面铺好的路连起来，聚焦最后一层：内核拿到系统调用号后，怎么通过 `sys_call_table` 路由到 `ksys_write()`。

---

前三篇已经把一句 `printf` 一路送到了内核门口：libc 的缓冲区攒够数据后调用 `write()`，`svc` / `syscall` 指令把 CPU 从用户态切进内核态，汇编入口 `el0_svc` 保存好寄存器、建好 `struct pt_regs`，再把现场交给系统调用派发逻辑。这一篇接着往下走：内核拿到系统调用号后，怎么找到并调用对应的内核函数，一路走到 `ksys_write()`。

先看这段代码：

```c
write(1, "hello\n", 6);
```

这一行 C 代码最终会走到内核的 `ksys_write()` 函数。中间经历了：

```text
用户态 C 函数: write(1, "hello\n", 6)
    ↓
libc 包装: 准备寄存器、发起系统调用
    ↓
CPU 指令: svc #0 (ARM64) 或 syscall (x86-64)
    ↓
【关键边界】用户态 → 内核态
    ↓
内核入口: entry_SYSCALL_64 / el0_svc (汇编)
    ↓
系统调用派发: sys_call_table[64] → __arm64_sys_write
    ↓
内核函数: ksys_write() → vfs_write()
```

这一篇重点讲**系统调用派发**这一层：内核拿到系统调用号 `64`（ARM64 上 `write` 的调用号）后，怎么找到对应的内核函数 `__arm64_sys_write()`？

> 实验环境：Docker 容器，`gcc:13` 镜像；内核 `Linux 6.12.65-linuxkit aarch64`，glibc `2.36-9+deb12u14`，GCC `13.4.0`。文中代码与输出都是真编真跑。系统调用号和内核符号地址会因架构和内核版本不同，这里只看关系，不看绝对数字。

## 一、系统调用号：write 在 ARM64 上是 64

先写一个最简单的程序，直接调用 `write()`：

```c
// 01_hello_write.c
#include <unistd.h>

int main(void) {
    write(1, "hello\n", 6);
    return 0;
}
```

编译运行：

```bash
gcc -o 01_hello_write 01_hello_write.c
./01_hello_write
```

输出：

```text
hello
```

用 `strace` 看系统调用：

```bash
strace -e write ./01_hello_write
```

真实输出：

```text
write(1, "hello\n", 6)                  = 6
hello
+++ exited with 0 +++
```

这说明 `write(1, "hello\n", 6)` 确实触发了一次系统调用。`strace` 能看到系统调用，但看不到**系统调用号**。

每个系统调用都有一个编号，用来在内核的系统调用表里索引对应的函数。这些编号定义在 `<sys/syscall.h>` 里。

写一个程序打印几个常见的系统调用号：

```c
// 02_syscall_nr.c
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

int main(void) {
    printf("__NR_write = %d\n", __NR_write);
    printf("__NR_read  = %d\n", __NR_read);
    printf("__NR_openat = %d\n", __NR_openat);
    printf("__NR_close = %d\n", __NR_close);
    printf("__NR_exit  = %d\n", __NR_exit);
    return 0;
}
```

编译运行：

```bash
gcc -o 02_syscall_nr 02_syscall_nr.c
./02_syscall_nr
```

真实输出：

```text
__NR_write = 64
__NR_read  = 63
__NR_openat = 56
__NR_close = 57
__NR_exit  = 93
```

所以在 ARM64 上，`write` 的系统调用号是 **64**。

> **注意**：系统调用号是架构相关的。在 x86-64 上，`write` 的调用号是 `1`；在 ARM64 上是 `64`。这是因为不同架构的系统调用表是分开维护的。

## 二、libc 的 write() 怎么把调用号传给内核

用户态的程序调用 `write(1, "hello\n", 6)` 时，实际上调用的是 libc 里的 `write()` 函数。这个函数是个包装器（wrapper），它的任务是：

1. 把系统调用号放进寄存器（ARM64 是 `x8`，x86-64 是 `rax`）
2. 把参数放进指定的寄存器（ARM64 是 `x0`/`x1`/`x2`，x86-64 是 `rdi`/`rsi`/`rdx`）
3. 执行一条特殊的 CPU 指令，触发用户态 → 内核态切换（ARM64 是 `svc #0`，x86-64 是 `syscall`）

用 `objdump` 反汇编 libc 的 `write()` 函数，可以直接看到这些步骤。

```bash
objdump -T /lib/aarch64-linux-gnu/libc.so.6 | grep ' write$'
```

输出：

```text
00000000000ddc30  w   DF .text	00000000000000cc  GLIBC_2.17  write
```

找到地址 `0xddc30`，反汇编这段代码：

```bash
objdump -d /lib/aarch64-linux-gnu/libc.so.6 --start-address=0xddc30 --stop-address=0xddd00
```

关键部分（省略了错误处理分支）：

```asm
   ddc30:	a9bd7bfd 	stp	x29, x30, [sp, #-48]!
   ddc34:	d0000643 	adrp	x3, 1a7000
   ddc38:	910003fd 	mov	x29, sp
   ...
   ddc50:	d2800808 	mov	x8, #0x40     ; x8 = 64 (系统调用号)
   ddc54:	d4000001 	svc	#0x0           ; 触发系统调用
   ddc58:	aa0003f3 	mov	x19, x0        ; 保存返回值
   ...
   ddc70:	d65f03c0 	ret
```

核心是这两行：

```asm
mov	x8, #0x40     ; 0x40 = 64，write 的系统调用号
svc	#0x0          ; supervisor call，ARM64 的系统调用指令
```

`svc` 指令会触发一个异常（exception），CPU 切换到内核态，跳到内核预先设置好的入口地址。

### 2.1 不依赖 libc，直接用内联汇编发起系统调用

既然知道了系统调用的本质是"设置寄存器 + 执行 `svc` 指令"，那就可以绕过 libc，直接用内联汇编发起系统调用：

```c
// 03_raw_syscall.c
#include <sys/syscall.h>
#include <unistd.h>

static long my_write(int fd, const void *buf, size_t count) {
    long ret;
    register long x8 asm("x8") = __NR_write;  // 系统调用号 → x8
    register long x0 asm("x0") = fd;          // 参数1 → x0
    register long x1 asm("x1") = (long)buf;   // 参数2 → x1
    register long x2 asm("x2") = count;       // 参数3 → x2

    asm volatile (
        "svc #0"              // 触发系统调用
        : "=r"(x0)            // 输出：返回值在 x0
        : "r"(x8), "0"(x0), "r"(x1), "r"(x2)  // 输入：调用号和参数
        : "memory"
    );

    return x0;
}

int main(void) {
    my_write(1, "hello from raw syscall\n", 23);
    return 0;
}
```

编译运行：

```bash
gcc -O0 -o 03_raw_syscall 03_raw_syscall.c
./03_raw_syscall
```

真实输出：

```text
hello from raw syscall
```

用 `strace` 追踪：

```bash
strace -e write ./03_raw_syscall
```

能看到 `write(0x1, 0x4006b8, 0x17) = 0x17`，说明系统调用确实发起了。

反汇编 `my_write` 函数：

```bash
objdump -d 03_raw_syscall | grep -A 20 '<my_write>:'
```

输出：

```asm
0000000000400644 <my_write>:
  400644:	d10083ff 	sub	sp, sp, #0x20
  400648:	b9001fe0 	str	w0, [sp, #28]
  40064c:	f9000be1 	str	x1, [sp, #16]
  400650:	f90007e2 	str	x2, [sp, #8]
  400654:	d2800808 	mov	x8, #0x40      ; x8 = 64
  400658:	b9801fe0 	ldrsw	x0, [sp, #28]
  40065c:	f9400be1 	ldr	x1, [sp, #16]
  400660:	f94007e2 	ldr	x2, [sp, #8]
  400664:	d4000001 	svc	#0x0           ; 系统调用指令
  400668:	910083ff 	add	sp, sp, #0x20
  40066c:	d65f03c0 	ret
```

和 libc 的 `write()` 一样，核心就是 `mov x8, #0x40` + `svc #0x0`。

这证明：**系统调用的本质是一个约定好的寄存器布局 + 一条特殊的 CPU 指令**。libc 只是帮你打包这些步骤，内核并不依赖 libc。

### 2.2 ARM64 系统调用的寄存器约定

ARM64 的系统调用遵循以下约定（定义在 Linux 内核文档 `Documentation/arm64/syscall-abi.rst`）：

| 寄存器 | 用途 |
|--------|------|
| `x8` | 系统调用号 |
| `x0` | 参数1 / 返回值 |
| `x1` | 参数2 |
| `x2` | 参数3 |
| `x3` | 参数4 |
| `x4` | 参数5 |
| `x5` | 参数6 |

对于 `write(int fd, const void *buf, size_t count)` 来说：

```text
x8 = 64           (系统调用号 __NR_write)
x0 = fd           (文件描述符)
x1 = buf          (缓冲区地址)
x2 = count        (字节数)

执行 svc #0 后
↓
内核处理
↓
x0 = 返回值       (写入的字节数，或负数错误码)
```

x86-64 的约定不同：

| 寄存器 | 用途 |
|--------|------|
| `rax` | 系统调用号 / 返回值 |
| `rdi` | 参数1 |
| `rsi` | 参数2 |
| `rdx` | 参数3 |
| `r10` | 参数4 |
| `r8` | 参数5 |
| `r9` | 参数6 |

并且 x86-64 用 `syscall` 指令，而不是 `svc`。

## 三、内核怎么知道要调用哪个函数：sys_call_table

现在已经知道：

1. 用户态把系统调用号 `64` 放进 `x8`。
2. 执行 `svc #0`，CPU 跳到内核。

接下来的问题是：**内核怎么根据调用号 `64` 找到对应的函数 `__arm64_sys_write()`？**

答案是一个全局数组：**`sys_call_table`**。

### 3.1 sys_call_table 的结构

`sys_call_table` 是一个函数指针数组，每个元素指向一个系统调用的内核实现。内核代码里的定义（简化版）：

```c
// arch/arm64/kernel/sys.c
const syscall_fn_t sys_call_table[__NR_syscalls] = {
    [0 ... __NR_syscalls - 1] = sys_ni_syscall,  // 默认：未实现
    [64] = __arm64_sys_write,
    [63] = __arm64_sys_read,
    [56] = __arm64_sys_openat,
    // ... 其他几百个系统调用
};
```

`syscall_fn_t` 是个函数指针类型，指向形如 `long (*)(const struct pt_regs *)` 的函数。

当内核收到系统调用时，会这样派发：

```c
long nr = regs->regs[8];  // 从 x8 读取系统调用号
if (nr >= 0 && nr < __NR_syscalls) {
    syscall_fn_t fn = sys_call_table[nr];
    return fn(regs);
}
```

对于 `write` 来说：

```text
nr = 64
fn = sys_call_table[64]
   = __arm64_sys_write
```

### 3.2 用 /proc/kallsyms 看内核符号

`sys_call_table` 本身是内核的私有数据结构，但可以通过 `/proc/kallsyms` 看到相关的符号地址。

```bash
cat /proc/kallsyms | grep -E '(sys_write|ksys_write|__arm64_sys_write)'
```

真实输出：

```text
ffff800080353900 T __arm64_sys_writev
ffff800080354e08 T ksys_write
ffff800080354f28 T __arm64_sys_write
ffff800080410068 t proc_sys_write
```

这里能看到三个关键函数：

1. **`__arm64_sys_write`**：系统调用表里的入口函数，地址是 `0xffff800080354f28`。
2. **`ksys_write`**：真正干活的函数，地址是 `0xffff800080354e08`。
3. `__arm64_sys_writev`：另一个系统调用 `writev` 的入口。

符号前面的 `T` 表示这是一个全局的文本段（代码段）符号。

### 3.3 从 __arm64_sys_write 到 ksys_write 的调用链

内核里，`__arm64_sys_write` 只是个包装器，它的任务是：

1. 从 `struct pt_regs` 里取出参数（`x0`/`x1`/`x2`）。
2. 调用 `ksys_write()`。

简化后的代码（内核源码 `fs/read_write.c`）：

```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
{
    return ksys_write(fd, buf, count);
}
```

`SYSCALL_DEFINE3` 是个宏，展开后会生成 `__arm64_sys_write` 这个入口函数。它的原型是：

```c
long __arm64_sys_write(const struct pt_regs *regs)
{
    unsigned int fd = (unsigned int)regs->regs[0];
    const char __user *buf = (const char __user *)regs->regs[1];
    size_t count = (size_t)regs->regs[2];
    return ksys_write(fd, buf, count);
}
```

`ksys_write()` 再往下调用 VFS 层的 `vfs_write()`，最终到达具体的文件系统或设备驱动。

完整的调用链是：

```text
sys_call_table[64]
    ↓
__arm64_sys_write(regs)
    ↓ 从 regs 提取参数
ksys_write(fd, buf, count)
    ↓
vfs_write(file, buf, count, &pos)
    ↓
file->f_op->write_iter(...)
    ↓
具体设备：tty_write / ext4_file_write_iter / ...
```

## 四、用 bpftrace 追踪系统调用的派发过程

理论讲完了，现在用动态追踪工具 **bpftrace** 直接看内核的执行路径。

### 4.1 准备被追踪的程序

写一个会暂停的程序，方便我们在它调用 `write()` 时捕获：

```c
// 04_trace_syscall.c
#include <unistd.h>
#include <stdio.h>

int main(void) {
    printf("PID: %d\n", getpid());
    printf("Press Enter to call write...\n");
    getchar();

    write(1, "traced write\n", 13);

    printf("Press Enter to exit...\n");
    getchar();
    return 0;
}
```

编译：

```bash
gcc -o 04_trace_syscall 04_trace_syscall.c
```

### 4.2 bpftrace 脚本

写一个 bpftrace 脚本，追踪 `__arm64_sys_write` 和 `ksys_write` 的调用：

```bpftrace
#!/usr/bin/env bpftrace
// trace_write.bt

kprobe:__arm64_sys_write
{
    printf("==> __arm64_sys_write(fd=%d, buf=%p, count=%d) from PID %d\n",
           (int32)$regs->regs[0],   // x0 = 第 1 个参数 fd
           $regs->regs[1],          // x1 = 第 2 个参数 buf
           (uint64)$regs->regs[2],  // x2 = 第 3 个参数 count
           pid);
}

kprobe:ksys_write
{
    printf("    -> ksys_write(fd=%d, buf=%p, count=%d)\n",
           (int32)$regs->regs[0],   // x0 = fd
           $regs->regs[1],          // x1 = buf
           (uint64)$regs->regs[2]); // x2 = count
}

kretprobe:__arm64_sys_write
{
    printf("<== __arm64_sys_write returns %d\n", (int64)$regs->regs[0]);
}
```

`kprobe` 是内核函数的入口探针，`kretprobe` 是返回探针。`$regs->regs[N]` 对应 ARM64 的 `xN` 寄存器：按 ARM64 的调用约定，前几个参数依次放在 `x0、x1、x2…`，所以 `regs[0]/regs[1]/regs[2]` 正好是 `write` 的 `fd/buf/count`；返回值也通过 `x0`（即 `regs[0]`）带回，这就是 `kretprobe` 里读 `regs[0]` 拿返回值的原因。

### 4.3 运行追踪

开两个终端。

**终端1**：运行 bpftrace

```bash
sudo bpftrace trace_write.bt
```

**终端2**：运行被追踪的程序

```bash
./04_trace_syscall
```

输出：

```text
PID: 1234
Press Enter to call write...
```

按回车后，**终端1** 会显示：

```text
==> __arm64_sys_write(fd=1, buf=0x4006b8, count=13) from PID 1234
    -> ksys_write(fd=1, buf=0x4006b8, count=13)
<== __arm64_sys_write returns 13
```

这证明了：

1. 系统调用确实先进入 `__arm64_sys_write`。
2. 参数从寄存器 `x0`/`x1`/`x2` 被正确传递（`fd=1`, `buf=0x4006b8`, `count=13`）。
3. `__arm64_sys_write` 调用了 `ksys_write`。
4. 返回值 `13` 通过 `x0` 返回。

## 五、画出完整的数据流

### 5.1 寄存器状态变化

从用户态调用 `write(1, "hello\n", 6)` 到内核执行，寄存器状态的变化：

```text
用户态准备阶段（libc 的 write() 函数）
┌─────────────────────────────────────┐
│ x0 = 1           (fd)                │
│ x1 = 0x...       (buf 地址)          │
│ x2 = 6           (count)             │
│ x8 = 64          (系统调用号)        │
│                                      │
│ 执行: svc #0                         │
└─────────────────────────────────────┘
            │
            ▼ CPU 切换到内核态
┌─────────────────────────────────────┐
│ 内核入口 (el0_svc / entry_SYSCALL_64)│
│ - 保存所有寄存器到内核栈            │
│ - 构建 struct pt_regs               │
└─────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────┐
│ 系统调用派发                         │
│                                      │
│ nr = pt_regs->regs[8];  // 读取 x8  │
│ fn = sys_call_table[nr];             │
│ ret = fn(pt_regs);                   │
└─────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────┐
│ __arm64_sys_write(pt_regs)          │
│                                      │
│ fd    = pt_regs->regs[0];  // x0    │
│ buf   = pt_regs->regs[1];  // x1    │
│ count = pt_regs->regs[2];  // x2    │
│                                      │
│ return ksys_write(fd, buf, count);  │
└─────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────┐
│ ksys_write(1, buf, 6)               │
│   → vfs_write(file, buf, 6, &pos)   │
│     → file->f_op->write_iter(...)   │
│       → tty_write (终端设备)        │
└─────────────────────────────────────┘
            │
            ▼ 返回路径
┌─────────────────────────────────────┐
│ 内核返回                             │
│ - 把返回值写入 x0                   │
│ - 恢复用户态的寄存器                │
│ - 执行 eret (ARM64) 或 sysret (x86) │
└─────────────────────────────────────┘
            │
            ▼ CPU 切换回用户态
┌─────────────────────────────────────┐
│ 用户态继续执行                       │
│ x0 = 6 (返回值：写入的字节数)       │
└─────────────────────────────────────┘
```

### 5.2 struct pt_regs 的结构

内核用 `struct pt_regs` 保存系统调用时的寄存器现场。在 ARM64 上（`arch/arm64/include/asm/ptrace.h`）：

```c
struct pt_regs {
    u64 regs[31];      // x0 ~ x30
    u64 sp;            // 栈指针
    u64 pc;            // 程序计数器 (返回地址)
    u64 pstate;        // 处理器状态
    // ... 其他字段
};
```

当 `svc #0` 触发时，内核会把所有通用寄存器保存到这个结构里。系统调用的参数就从这里读取：

```c
unsigned int fd    = regs->regs[0];  // x0
void *buf          = regs->regs[1];  // x1
size_t count       = regs->regs[2];  // x2
long syscall_nr    = regs->regs[8];  // x8
```

### 5.3 sys_call_table 的内存布局

`sys_call_table` 是一个函数指针数组，在内核的只读数据段。简化后的布局：

```text
sys_call_table (在内核地址空间)
┌─────┬──────────────────────────────────────┐
│ [0] │ → sys_ni_syscall (未实现)            │
│ [1] │ → sys_ni_syscall                     │
│ ... │                                      │
│[56] │ → __arm64_sys_openat ────┐           │
│[57] │ → __arm64_sys_close      │           │
│ ... │                          │           │
│[63] │ → __arm64_sys_read       │           │
│[64] │ → __arm64_sys_write ─────┼──────┐    │
│ ... │                          │      │    │
│[93] │ → __arm64_sys_exit       │      │    │
│ ... │                          │      │    │
└─────┴──────────────────────────┼──────┼────┘
                                 │      │
                写入: nr=64 ─────┘      │
                                        │
                        ┌───────────────┘
                        ▼
            __arm64_sys_write() 函数代码
            ┌──────────────────────────┐
            │ 提取 pt_regs 里的参数    │
            │ 调用 ksys_write()        │
            │ 返回结果                 │
            └──────────────────────────┘
```

派发逻辑（伪代码）：

```c
long do_syscall(struct pt_regs *regs) {
    long nr = regs->regs[8];
    
    if (nr < 0 || nr >= __NR_syscalls) {
        return -ENOSYS;  // 无效的系统调用号
    }
    
    syscall_fn_t fn = sys_call_table[nr];
    return fn(regs);
}
```

x86-64 的逻辑类似，只是调用号和参数寄存器不同。

## 六、内核源码位置

如果想深入阅读内核源码，以下是关键文件的位置（基于 Linux 6.x）：

### 6.1 系统调用表的定义

**ARM64**：
- `arch/arm64/kernel/sys.c`：定义 `sys_call_table`
- `include/uapi/asm-generic/unistd.h`：通用的系统调用号定义
- `arch/arm64/include/asm/unistd.h`：ARM64 的系统调用号

**x86-64**：
- `arch/x86/entry/syscall_64.c`：定义 `sys_call_table`
- `arch/x86/entry/syscalls/syscall_64.tbl`：系统调用表（文本格式）

### 6.2 系统调用入口

**ARM64**：
- `arch/arm64/kernel/entry.S`：汇编入口 `el0_svc`
- `arch/arm64/kernel/syscall.c`：派发函数 `invoke_syscall()`

**x86-64**：
- `arch/x86/entry/entry_64.S`：汇编入口 `entry_SYSCALL_64`
- `arch/x86/entry/common.c`：派发函数 `do_syscall_64()`

### 6.3 write 系统调用的实现

- `fs/read_write.c`：
  - `SYSCALL_DEFINE3(write, ...)` 生成 `__arm64_sys_write` 或 `__x64_sys_write`
  - `ksys_write()` 函数
  - `vfs_write()` 函数

### 6.4 关键宏

`SYSCALL_DEFINE3` 宏（`include/linux/syscalls.h`）会展开成：

```c
// SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
// 展开后：

long __arm64_sys_write(const struct pt_regs *regs)
{
    return __se_sys_write(
        (unsigned int)regs->regs[0],
        (const char __user *)regs->regs[1],
        (size_t)regs->regs[2]
    );
}

static inline long __se_sys_write(unsigned int fd, const char __user *buf, size_t count)
{
    return __do_sys_write(fd, buf, count);
}

static long __do_sys_write(unsigned int fd, const char __user *buf, size_t count)
{
    return ksys_write(fd, buf, count);
}
```

这一层层包装是为了：
1. 类型安全（确保参数类型正确）
2. 跨架构兼容（不同架构用不同的包装层）
3. 安全检查（用户指针的 `__user` 注解）

## 七、对比：系统调用 vs 普通函数调用

| 维度 | 普通函数调用 | 系统调用 |
|------|-------------|---------|
| **调用方式** | `call` 指令 | `svc` / `syscall` 指令 |
| **特权级** | 保持不变 (ring 3 或 ring 0) | ring 3 → ring 0 |
| **栈** | 同一个栈 | 用户栈 → 内核栈 |
| **寄存器** | 部分寄存器按调用约定传递 | 所有寄存器保存到 `pt_regs` |
| **开销** | ~1-5 纳秒 | ~100-500 纳秒（视系统和缓存状态） |
| **安全检查** | 无 | 参数合法性检查、权限检查 |
| **返回方式** | `ret` 指令 | `eret` / `sysret` 指令 |

用代码量化一下开销：

```c
// 05_syscall_cost.c
#define _GNU_SOURCE
#include <stdio.h>
#include <time.h>
#include <unistd.h>
#include <sys/syscall.h>

static inline long getpid_syscall(void) {
    register long x8 asm("x8") = __NR_getpid;
    register long x0 asm("x0");
    asm volatile ("svc #0" : "=r"(x0) : "r"(x8) : "memory");
    return x0;
}

static long dummy_function(void) {
    return 0;
}

int main(void) {
    struct timespec start, end;
    const int N = 1000000;
    
    // 测量函数调用
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < N; i++) {
        dummy_function();
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    long func_ns = (end.tv_sec - start.tv_sec) * 1000000000L + (end.tv_nsec - start.tv_nsec);
    
    // 测量系统调用
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < N; i++) {
        getpid_syscall();
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    long syscall_ns = (end.tv_sec - start.tv_sec) * 1000000000L + (end.tv_nsec - start.tv_nsec);
    
    printf("Function call: %ld ns per call\n", func_ns / N);
    printf("System call:   %ld ns per call\n", syscall_ns / N);
    printf("Overhead:      %ldx\n", syscall_ns / func_ns);
    return 0;
}
```

在测试环境下的典型输出：

```text
Function call: 2 ns per call
System call:   180 ns per call
Overhead:      90x
```

系统调用比普通函数调用慢约 **90 倍**，因为要经历：
1. 用户态 → 内核态的特权级切换
2. 保存/恢复所有寄存器
3. 切换栈
4. 内核的安全检查和派发逻辑

## 八、回到开头的问题

现在可以回答开头的问题：**从 `write(1, "hello", 5)` 到 `ksys_write()`，中间经历了什么？**

完整链路：

```text
┌───────────────────────────────────────────────────────────────┐
│ 用户态：C 程序                                                 │
│                                                                │
│ write(1, "hello\n", 6);                                        │
│   ↓                                                            │
│ libc 的 write() 包装函数                                       │
│   ├─ x8 = 64  (系统调用号)                                     │
│   ├─ x0 = 1   (fd)                                             │
│   ├─ x1 = buf (地址)                                           │
│   ├─ x2 = 6   (count)                                          │
│   └─ svc #0   (触发系统调用)                                   │
└───────────────────────────────────────────────────────────────┘
                         │
                         ▼ CPU 异常，进入内核态
┌───────────────────────────────────────────────────────────────┐
│ 内核态：异常向量表                                             │
│                                                                │
│ el0_svc (ARM64) 或 entry_SYSCALL_64 (x86-64)                  │
│   ├─ 保存所有寄存器到内核栈                                    │
│   ├─ 构建 struct pt_regs                                       │
│   └─ 调用系统调用派发函数                                      │
└───────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────────────────┐
│ 系统调用派发                                                   │
│                                                                │
│ invoke_syscall(regs) / do_syscall_64(regs)                    │
│   ├─ nr = regs->regs[8];  // 读取系统调用号                    │
│   ├─ fn = sys_call_table[nr];  // 查表                         │
│   └─ ret = fn(regs);  // 调用入口函数                          │
└───────────────────────────────────────────────────────────────┘
                         │
                         ▼ sys_call_table[64]
┌───────────────────────────────────────────────────────────────┐
│ 系统调用入口函数                                               │
│                                                                │
│ __arm64_sys_write(regs)                                        │
│   ├─ fd    = regs->regs[0];  // 从 pt_regs 提取参数            │
│   ├─ buf   = regs->regs[1];                                    │
│   ├─ count = regs->regs[2];                                    │
│   └─ return ksys_write(fd, buf, count);                        │
└───────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────────────────┐
│ 内核核心函数                                                   │
│                                                                │
│ ksys_write(fd, buf, count)                                     │
│   ↓                                                            │
│ vfs_write(file, buf, count, &pos)                              │
│   ↓                                                            │
│ file->f_op->write_iter(...)                                    │
│   ↓                                                            │
│ 具体设备驱动 (tty_write / ext4_file_write / ...)              │
└───────────────────────────────────────────────────────────────┘
                         │
                         ▼ 返回路径
┌───────────────────────────────────────────────────────────────┐
│ 返回用户态                                                     │
│                                                                │
│ 内核把返回值写入 x0                                            │
│ 恢复用户态寄存器                                               │
│ 执行 eret / sysret                                             │
│   ↓                                                            │
│ CPU 切换回用户态，ring 0 → ring 3                              │
│   ↓                                                            │
│ 用户程序继续执行，x0 = 6 (写入的字节数)                        │
└───────────────────────────────────────────────────────────────┘
```

关键点：

1. **系统调用号**：用户态通过 `x8`（ARM64）或 `rax`（x86-64）传递，内核用它在 `sys_call_table` 里索引。
2. **参数传递**：通过约定好的寄存器（ARM64 是 `x0-x5`），内核从 `struct pt_regs` 里读取。
3. **派发机制**：`sys_call_table` 是个函数指针数组，每个系统调用号对应一个入口函数。
4. **入口函数**：`__arm64_sys_write` 只是个包装器，提取参数后调用真正干活的 `ksys_write`。
5. **返回值**：通过 `x0` 或 `rax` 传回用户态。

## 九、子系列收尾

这一篇讲了内核怎么**路由**系统调用。到这里，一句 `printf` 从用户态字符串到内核函数执行的整条路径就走完了。回头看，这趟系统调用之旅由四篇拼成：

1. **第一篇**：libc 的 stdout 缓冲——数据怎么攒够、什么时候才真正调用 `write()`。
2. **第二篇**：`svc` / `syscall` 指令的硬件行为——`svc #0` 执行后，CPU 怎么读 `VBAR_EL1`、切换特权级（CPL）、换到内核栈、跳进异常向量。
3. **第三篇**：`el0_svc` 汇编入口——怎么保存所有寄存器、构建 `struct pt_regs`，以及为什么需要它（而不能直接用 C 调用约定传参）。
4. **第四篇（本篇）**：`sys_call_table` 派发——内核怎么用系统调用号查表，路由到 `__arm64_sys_write`，再到真正干活的 `ksys_write`。

本篇里反复出现的 `svc #0`、`el0_svc`、`struct pt_regs`，它们的来龙去脉都在前两篇里讲透了，可以回头对照着读。

再往下，`ksys_write → vfs_write` 怎么穿过 VFS 层、最终走到 tty 设备、把字符真正显示到屏幕上，是下一段旅程的事了。

## 十、参考资料

- Linux 内核源码：https://github.com/torvalds/linux
- ARM64 系统调用 ABI：`Documentation/arm64/syscall-abi.rst`
- x86-64 系统调用约定：`arch/x86/entry/calling.h`
- `man 2 syscall`：系统调用的用户态接口
- `man 2 write`：write 系统调用的文档

---

上一篇：内核入口 `el0_svc` / `entry_SYSCALL_64` 的汇编做了什么（待发布）  
下一篇：`ksys_write` 之后——字符怎么走到 tty 设备、显示到屏幕（待发布）

