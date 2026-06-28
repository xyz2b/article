# 二、`svc` / `syscall` 指令到底做了什么——从 ring 3 到 ring 0 的那道硬件门

---

> **🗺️ 全流程全景可视化**：[→ syscall-panorama-visualizer.html](https://xyz2b.github.io/article/CodeAdventure/front/syscall-panorama-visualizer.html)
> 把本系列四篇串成一条流水线，盯住 `write(1,"hello\n",6)` 一路下沉：**libc 缓冲**攒够数据 → **`svc` 硬件门**跨过特权边界 → **内核入口汇编**保存现场建 `pt_regs` → **`sys_call_table`** 查表派发到 `ksys_write`，再一路 `__arm64_sys_write → ksys_write → vfs_write → tty_write` 把字节写到终端。六个工位全程在屏、寄存器条常驻，还标出代码在「用户 C / libc 汇编 / 硬件 / 内核汇编 / 内核 C」之间的切换，以及 `bl do_el0_svc` 那一步汇编把 `pt_regs` 交给 C 的交接点。建议先扫一遍全景，再看每篇细节。

---

> **🎯 交互式可视化**：[→ syscall-gate-visualizer.html](https://xyz2b.github.io/article/CodeAdventure/front/syscall-gate-visualizer.html)
> `svc` / `syscall` 执行的那一瞬间，CPU 硬件在内部做的几件事：切特权级、存返回地址与处理器状态、关中断、从入口寄存器取地址、换内核栈，再跳进向量表的全过程动画。

---

> **系列说明**：这是"一句 printf 怎么出现在屏幕上"系列的第二篇。第一篇《[printf("hello") 怎么变成 write(1, "hello", 5)](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/一、printf怎么变成write——libc的stdout缓冲机制.md)》讲清 libc 的 stdout 缓冲，把数据攒够、交到 `write()` 系统调用手上。这一篇接着往下走：`write()` 最后执行的那条 `svc` / `syscall` 指令，**到底是怎么把 CPU 从用户态送进内核态的**——为什么一条指令就能跨过特权边界，而你不能简单地 `jmp` 到内核地址。第三篇《[内核入口 el0_svc / entry_SYSCALL_64 的汇编做了什么](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/三、内核入口el0_svc做了什么——从异常向量到C函数.md)》接着讲：CPU 落到那个入口地址之后，那段汇编怎么保存现场、建好 `struct pt_regs`。第四篇《[从 write 到 ksys_write——sys_call_table 怎么路由的](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/四、从write到ksys_write——sys_call_table怎么路由的.md)》讲内核怎么查表路由到 `ksys_write`。

---

第一篇结尾，libc 的缓冲区攒够了数据，调用了 `write()`。这一篇盯住 `write()` 里最关键的那一条指令。

先看现象。一个最普通的 `write`：

```c
write(1, "hello", 5);   // 用户态
// ↓ 一条 svc / syscall 指令
// ↓ 瞬间进入内核态
// 内核的 el0_svc / entry_SYSCALL_64 开始执行
```

中间那一行汇编指令（ARM64 是 `svc #0`，x86-64 是 `syscall`）做了一件很神奇的事：执行它之前，CPU 是 ring 3（用户态、EL0），访问不了内核的任何一个字节；执行它之后，CPU 变成了 ring 0（内核态、EL1），正在跑内核的代码。

**问题来了：这一行指令凭什么能做到？为什么不能直接 `jmp` 到内核函数的地址，非要用这条特殊指令？**

这一篇就把这条指令掰开，看它在硬件层面到底做了哪几件事。

> **实验环境**：Docker 容器，`gcc:13` 镜像；内核 `Linux 6.12.65-linuxkit aarch64`，glibc `2.36-9+deb12u14`，GDB `13`。本机是 **ARM64**，所以所有反汇编、`/proc/kallsyms`、gdb 单步、SIGSEGV/SIGILL 实验都是真编真跑的 **arm64 实测**，主角是 `svc`。x86-64 的 `syscall` 在这台机器上跑不到（Docker 底层内核是 arm64，x86 二进制的系统调用被 QEMU 翻译成了 arm64），所以 x86 部分是**读 Intel SDM / Linux 6.12 源码做的对照讲解**，会明确标注。两条路在结构上高度同构，对照着看最清楚。符号地址会因内核版本/编译不同而变，这里只看关系，不看绝对数字。

## 一、先抓住这条指令：libc 的 `write` 里藏着 `svc`

空口说"有一条特殊指令"没意思，先把它从真实的 libc 里挖出来。反汇编 libc 的 `write()`：

```bash
ADDR=$(objdump -T /lib/aarch64-linux-gnu/libc.so.6 | awk '/ write$/{print $1; exit}')
objdump -d /lib/aarch64-linux-gnu/libc.so.6 \
    --start-address=0x$ADDR --stop-address=$((0x$ADDR + 0x40))
```

关键的两行（真实输出）：

```asm
   ddc50:	d2800808 	mov	x8, #0x40                  	// #64
   ddc54:	d4000001 	svc	#0x0
```

就这两条：

```asm
mov	x8, #0x40     ; x8 = 64，write 的系统调用号
svc	#0x0          ; supervisor call —— 真正跨过特权边界的那一条
```

`mov x8, #0x40` 只是普通的寄存器赋值，不稀奇。稀奇的是 `svc #0x0`。它的机器码是 `d4000001`，是 ARMv8 指令集里专门留出来的一条"陷入内核"指令。x86-64 上对应的是 `syscall`（机器码 `0F 05`）。

顺带一个常被误解的点：`svc #0x0` 后面那个立即数 `#0`，**Linux 根本不看**。系统调用号是从 `x8` 读的，不是从这个立即数读的。这个立即数能填 0~65535，但 Linux 的约定是统一填 0。（早期一些系统确实用过 `svc` 立即数传号，Linux 没这么做。）

这条指令不依赖 libc。你可以手写内联汇编，把"设好寄存器 + `svc`"这套动作自己拼成一个完整的小程序：

```c
// 22_step_svc.c —— 不经过 libc，手写 svc 发起一次 write
#include <unistd.h>

int main(void) {
    register long x8 asm("x8") = 64;            // __NR_write，系统调用号
    register long x0 asm("x0") = 1;             // fd
    register long x1 asm("x1") = (long)"hi\n";  // buf
    register long x2 asm("x2") = 3;             // count
    asm volatile("svc #0" : "+r"(x0) : "r"(x8), "r"(x1), "r"(x2) : "memory");
    // 执行后 x0 = 返回值（写出的字节数）
    return 0;
}
```

编译运行：

```bash
gcc -g -O0 -o 22_step_svc 22_step_svc.c
./22_step_svc
```

真实输出（`svc` 真把 "hi" 写到了 stdout）：

```text
hi
```

全程没碰 libc 的 `write()`，照样往屏幕写出了字。这证明：**系统调用的本质就是"约定好的寄存器布局 + 一条 `svc` 指令"，内核认的是这条指令，不是 libc。**（这个程序第五节还会用 gdb 单步它。）

那么——为什么必须是这条指令？

## 二、为什么不能直接 `jmp` 到内核地址？

最自然的疑问是：内核函数也是代码，也有地址（`/proc/kallsyms` 里都列着）。我直接 `call` 那个地址不就进内核了，何必绕一条特殊指令？

下面用两个实验，亲手撞一下这堵墙。

### 2.1 实验 A：直接 call 内核地址 → 段错误

先从 `/proc/kallsyms` 拿一个真实的内核地址。比如系统调用的入口 `el0t_64_sync`：

```bash
grep -E ' el0t_64_sync$' /proc/kallsyms
```

```text
ffff800080011340 t el0t_64_sync
```

注意这个地址 `0xffff800080011340`——高半区，最高几位全是 `f`。Linux 把地址空间劈成两半：低半区 `0x0000...` 给用户态，高半区 `0xffff...` 给内核。现在我们硬把这个内核地址当函数指针来调用（外面套一个信号处理器，免得程序直接崩掉看不到结果）：

```c
// 20_jmp_kernel.c —— 证明：用户态不能直接 jmp/call 进内核地址
#include <stdio.h>
#include <signal.h>
#include <setjmp.h>

static sigjmp_buf env;
static volatile int sig_caught;

static void handler(int sig) {
    sig_caught = sig;
    siglongjmp(env, 1);          // 从信号里跳回 main，继续往下打印
}

int main(void) {
    signal(SIGSEGV, handler);
    signal(SIGILL,  handler);

    // 一个内核空间地址（el0t_64_sync 入口，来自 /proc/kallsyms）
    // 内核高半区地址，用户态（EL0）无权访问
    void (*kernel_entry)(void) = (void (*)(void))0xffff800080011340UL;

    printf("准备直接 call 内核地址 %p ...\n", (void *)kernel_entry);
    fflush(stdout);

    if (sigsetjmp(env, 1) == 0) {
        kernel_entry();          // 直接跳进内核——本该“瞬间进内核态”？
        printf("居然回来了（不应该发生）\n");
    } else {
        printf("-> 被信号打断: %s (signo=%d)\n",
               sig_caught == SIGSEGV ? "SIGSEGV 段错误" :
               sig_caught == SIGILL  ? "SIGILL 非法指令" : "其它",
               sig_caught);
    }
    return 0;
}
```

编译运行：

```bash
gcc -O0 -o 20_jmp_kernel 20_jmp_kernel.c
./20_jmp_kernel
```

真实输出：

```text
准备直接 call 内核地址 0xffff800080011340 ...
-> 被信号打断: SIGSEGV 段错误 (signo=11)
```

**`call` 那一刻，CPU 去取那个地址上的指令，硬件一看：当前是 EL0（ring 3），却要去读一个标了"仅 EL1（ring 0）可访问"的内核页——权限不够，立刻抛出异常**。内核把这个异常翻译成 `SIGSEGV` 发回给进程。

页表项里每一页都有一位"特权位"（ARM64 是 `AP`/`UXN` 那套，x86 是页表项的 `U/S` 位）。内核页全部标成"supervisor only"。所以用户态连**读**内核代码都不行，更别说跳进去执行了。`jmp`/`call` 这条路，从硬件层面就被页表堵死了。

那这个"特权位"到底长在页表项的哪里？把一条页表项摊开看，它是一个 64 位的字，里头除了"这页映射到哪块物理内存"，还有一排权限/属性位。先看本机 ARM64 的末级页表项（只标出关键字段）：

```text
ARM64 末级页表项（Page Descriptor，64 位）

 高位 ◄──────────────────────────────────────────────────────► 低位
┌──────┬──────┬─────┬──────────────────┬──────┬───────┬─────┬───────┐
│ UXN  │ PXN  │  …  │   PFN 物理页帧号   │  AF  │  AP   │  …  │ Valid │
│ b54  │ b53  │     │      b47..12       │ b10  │ b7:6  │     │  b0   │
└──────┴──────┴─────┴──────────────────┴──────┴───────┴─────┴───────┘
   └ 执行权限 ┘         └ 地址映射 ┘       访问   └ 访问权限 ┘  有效位
```

| 字段 | 位 | 大致作用 | 和"特权墙"的关系 |
|---|---|---|---|
| `Valid` | b0 | 这一项是否有效（是否指向真实物理页） | 为 0 → 翻译直接失败 |
| `AP[2:1]` | b7:6 | 访问权限：EL0 能不能访问 + 可写/只读 | **核心**：`AP[1]=0` 表示"EL0 无权访问"，内核页就是靠它把用户态挡在外面 |
| `AF` | b10 | 访问标志（这页有没有被访问过） | 辅助内核统计/换页，和权限墙关系不大 |
| `PFN` | b47:12 | 物理页帧号（虚拟地址最终落到哪块物理内存） | 翻译的产物，不管权限 |
| `PXN` | b53 | 特权级不可执行（连 EL1 都不能在这页取指） | 防内核误把数据页当代码执行 |
| `UXN` | b54 | 非特权不可执行（EL0 不能在这页取指） | 内核代码页置 1 → 用户态连"执行"这条路也焊死 |

x86-64 的末级页表项（PTE）结构同理，管特权的那一位叫 `U/S`：

```text
x86-64 末级页表项（PTE，64 位）

 高位 ◄────────────────────────────────────────► 低位
┌──────┬──────────────────┬─────┬─────┬─────┬─────┐
│  XD  │   PFN 物理页帧号   │  …  │ U/S │ R/W │  P  │
│ b63  │      b51..12       │     │ b2  │ b1  │ b0  │
└──────┴──────────────────┴─────┴─────┴─────┴─────┘
  执行权限    └ 地址映射 ┘       特权  读写  存在
```

| 字段 | 位 | 大致作用 |
|---|---|---|
| `P`（Present） | b0 | 这页在不在内存里 |
| `R/W` | b1 | 0 = 只读，1 = 可写 |
| `U/S` | b2 | **核心**：0 = 仅 supervisor（ring 0）可访问，1 = user（ring 3）也行；内核页一律置 0 |
| `XD`/`NX` | b63 | 1 = 本页不可执行 |

两边对得很齐：ARM64 用 `AP[1]` 控制"EL0 能不能碰"，x86 用 `U/S` 控制"ring 3 能不能碰"——**就是这一位，在 MMU 翻译地址的那一拍把用户态挡下来的**。实验 A 撞的就是它。

> 这里只用到页表项"有这么一位特权位"的结论。至于 CPU 这套分页机制本身——页表为什么是多级的、MMU 一次地址翻译在硬件里怎么一步步走、TLB 怎么缓存、页表项里这套权限/特权位（`AP`/`UXN`/`U/S`、内核页凭什么对用户态不可见）具体怎么生效——够单开一篇细讲，后续会补，这里先点到为止。

### 2.2 实验 B：连"门在哪"都不让你知道 → 非法指令

也许你想：那我换个办法，先想办法读出内核的入口寄存器，再……？

内核入口地址存在一个特权寄存器里（ARM64 是 `VBAR_EL1`，下一节细讲）。我们试着在用户态读它：

```c
// 21_read_vbar.c —— 证明：EL0 连“内核入口寄存器”都读不到
#include <stdio.h>
#include <signal.h>
#include <setjmp.h>

static sigjmp_buf env;
static volatile int sig_caught;
static void handler(int sig) { sig_caught = sig; siglongjmp(env, 1); }

int main(void) {
    signal(SIGILL,  handler);
    signal(SIGSEGV, handler);
    unsigned long v = 0;

    printf("准备在 EL0 执行 mrs x0, VBAR_EL1（读异常向量基址）...\n");
    fflush(stdout);

    if (sigsetjmp(env, 1) == 0) {
        asm volatile("mrs %0, vbar_el1" : "=r"(v));   // 特权寄存器，EL0 无权读
        printf("VBAR_EL1 = 0x%lx（不应该读到）\n", v);
    } else {
        printf("-> 被信号打断: %s (signo=%d)\n",
               sig_caught == SIGILL ? "SIGILL 非法指令" : "其它", sig_caught);
    }
    return 0;
}
```

编译运行：

```bash
gcc -O0 -o 21_read_vbar 21_read_vbar.c
./21_read_vbar
```

真实输出：

```text
准备在 EL0 执行 mrs x0, VBAR_EL1（读异常向量基址）...
-> 被信号打断: SIGILL 非法指令 (signo=4)
```

**`mrs ..., vbar_el1` 这条指令，在 EL0 执行直接是"非法指令"（SIGILL）**。不是权限不够、不是读到 0——是这条指令在用户态根本不允许出现，CPU 当场判它无效。

两个实验合起来，把墙的形状画清楚了：

- 用户态**进不去**内核地址（实验 A：页表挡住）。
- 用户态**看不到**内核入口在哪（实验 B：特权寄存器读不了）。

既进不去、又不知道门在哪，用户态怎么可能"自己跳进内核"？所以必须有一道**受控的门**：由内核提前布置好，用户态只能用一条专门的指令去敲，敲完由 CPU 硬件把你领到内核**指定**的那一个入口、并顺手把特权级升上去。这道门，就是 `svc` / `syscall`。

## 三、门开在哪：入口地址寄存器（VBAR_EL1 / IA32_LSTAR）

`svc` 跳进内核，可它跳到**哪个**地址？这个地址不能由用户态指定（否则就能跳进内核任意位置，绕过所有检查），必须由内核**提前写死在一个特权寄存器里**。

### 3.1 ARM64：VBAR_EL1 → 异常向量表 `vectors`

ARM64 上，这个寄存器叫 `VBAR_EL1`（Vector Base Address Register），它指向一张**异常向量表**。内核启动时把表的基址写进 `VBAR_EL1`，之后任何异常（中断、缺页、`svc`……）发生时，CPU 都按"异常类型"算一个固定偏移，跳到 `VBAR_EL1 + 偏移`。

这张表在内核里有个符号名 `vectors`，能在 kallsyms 里看到：

```bash
grep -E ' vectors$| el0t_64_sync$' /proc/kallsyms
```

真实输出：

```text
ffff800080010800 T vectors        ← VBAR_EL1 指向这里
ffff800080011340 t el0t_64_sync   ← “从 64 位用户态来的同步异常”的处理入口
```

ARMv8 的向量表布局是固定的：16 个表项，每项 `0x80` 字节，按"来源 EL + 异常类型"分成 4 组。其中 **`svc` 从 64 位 EL0 触发的是"同步异常"，偏移量是 `0x400`**：

```text
VBAR_EL1 (= vectors)
偏移 0x000  ┐ 来自 Current EL (SP0)
偏移 0x200  ┤ 来自 Current EL (SPx)
偏移 0x400  ┤ 来自 Lower EL (AArch64)   ← svc 从 64 位用户态进来，落这一组
   +0x000   同步异常 (Synchronous)       ← svc 是同步异常，就是这一格！
   +0x080   IRQ
   +0x100   FIQ
   +0x180   SError
偏移 0x600  ┘ 来自 Lower EL (AArch32)
```

这张表为什么这么排？关键是它由**两个维度**相乘而来：**来源**（异常从什么状态来）决定落在哪一"组"，每组间隔 `0x200`；**类型**（这是哪一类异常）决定落在组内哪一"格"，每格间隔 `0x80`。4 组 × 4 格 = 16 个表项，每项 `0x80` 字节，整张表正好 `0x800`（2048）字节。

**维度一：异常从哪来（分 4 组，竖着看）。** 这里的核心概念是 **EL（Exception Level，异常级别）**：EL0 是用户态，EL1 是内核态，级别越高权限越大。判断分两步：

| 偏移 | 来源 | 大白话 |
|------|------|--------|
| `0x000` | Current EL with SP0 | 异常发生时 CPU **已在内核态（EL1）**，且用 EL0 的栈指针 |
| `0x200` | Current EL with SPx | 异常发生时 CPU **已在内核态（EL1）**，用 EL1 自己的栈指针 |
| `0x400` | Lower EL using AArch64 | 异常来自**更低级别（EL0 用户态）**，且该程序是 **64 位** |
| `0x600` | Lower EL using AArch32 | 异常来自**更低级别（EL0 用户态）**，且该程序是 **32 位** |

- 先看**异常是"原地发生"还是"从下面上来"**：CPU 本来就在 EL1 跑、当场触发异常（如内核代码缺页），叫 "Current EL"（前两组）；CPU 在 EL0 跑、触发异常要升到 EL1，叫 "Lower EL"（后两组）。`svc` 就属于后者——用户态主动喊一嗓子要进内核。
- 再看**细分依据**：两组 "Current EL" 按**栈指针**分（SP0 / SPx），是内核自己内部的事，与系统调用无关；两组 "Lower EL" 按**位宽**分（来的是 64 位还是 32 位用户程序），因为内核要兼容老的 32 位程序，处理方式不同。

`svc` 从一个 64 位用户程序发出 → 在 EL0、要升 EL1、程序是 AArch64 → 命中第三组，偏移 **`0x400`**。

**维度二：异常是什么类型（每组内分 4 格，横着看）。** 每格 `0x80` 字节：

| 组内偏移 | 类型 | 含义 |
|----------|------|------|
| `+0x000` | Synchronous（同步异常） | 由某条指令"当场"引发，如 `svc`、缺页、非法指令 |
| `+0x080` | IRQ | 普通硬件中断（外设来敲门） |
| `+0x100` | FIQ | 快速中断（更高优先级） |
| `+0x180` | SError | 系统错误（一般是硬件层面的严重错误） |

"同步"指这个异常和**某条具体指令**绑死，执行到它**必然、立刻**触发，位置可预测。`svc` 就是典型：你写了 `svc #0`，执行到它就一定触发异常。对比之下 IRQ/FIQ 是外设随机什么时候来的，跟你正跑哪条指令无关，所以是"异步"。`svc` 是同步异常 → 在组内落 **`+0x000`** 这一格。

两个维度一合：落点 = `0x400`（来自 64 位用户态）+ `0x000`（同步异常）= `0x400`。再举个对照：CPU 正在内核态（EL1，用 SPx 栈）跑着，突然来个硬件中断（IRQ），落点就是 `0x200` + `0x080` = `VBAR_EL1 + 0x280`。换个维度就换格——这正是这张表的妙处：**偏移量本身就编码了"谁来的 + 什么事"，CPU 不用查表判断，直接算地址跳过去**，省掉了软件分发的开销。

所以 `svc` 从 EL0 进来，CPU 硬件算出落点 = `VBAR_EL1 + 0x400`。这一格里放的是一小段跳转代码，`b`（branch）到真正的处理函数 `el0t_64_sync`——也就是我们上面 kallsyms 里看到的那个符号。

**这就和第三篇接上了**：第三篇的内核栈实测，最底下一帧正是 `el0t_64_sync`。这一篇负责把 CPU 送到 `vectors + 0x400` 这道门口，第三篇负责讲门后面那段汇编干了什么。

注意第二节实验 B：用户态读不了 `VBAR_EL1`。也就是说，**内核入口地址对用户态是不可见、更不可改的**。用户态唯一能做的，就是执行 `svc`，剩下"跳到哪"完全由这个用户态碰不到的寄存器说了算。门的钥匙攥在内核手里——这正是安全的根。

### 3.2 x86-64：IA32_LSTAR 等三个 MSR（源码/文档对照）

x86-64 没有"向量表"这层，`syscall` 指令更直接：它从一个 **MSR**（Model Specific Register，型号专属寄存器）里取入口地址，一步到位。涉及三个 MSR：

| MSR | 编号 | 内容 |
|-----|------|------|
| `IA32_LSTAR` | `0xC0000082` | `syscall` 的入口地址，内核启动时填成 `entry_SYSCALL_64` |
| `IA32_STAR` | `0xC0000081` | 内核态/用户态的 `CS`、`SS` 段选择子（决定切到 ring 0 用哪个段） |
| `IA32_FMASK` | `0xC0000084` | 进入内核时要从 `RFLAGS` 里清掉的位（含中断使能位 `IF`，即"关中断"） |

（还有 `IA32_EFER` 的 `SCE` 位，bit 0，得置 1 才允许用 `syscall` 指令。）

在一台**真 x86-64 机器**上，可以用 `rdmsr` 工具（root 权限）把 `IA32_LSTAR` 读出来，和 kallsyms 对一下（**以下为 x86 文档对照，非本机输出**）：

```text
# rdmsr -0 0xc0000082            # 读 IA32_LSTAR
ffffffff81e00000
# grep ' entry_SYSCALL_64$' /proc/kallsyms
ffffffff81e00000 T entry_SYSCALL_64
```

> 注意区分两个同名的东西：`rdmsr` **指令**是特权指令，ring 3 执行会 #GP；而这里用的 `rdmsr` **工具**（msr-tools）跑在用户态，它并不亲自执行 `rdmsr` 指令，而是 `open("/dev/cpu/0/msr")` 再 `pread(fd, &val, 8, 0xc0000082)`。这个设备背后是内核的 `msr` 模块，内核在 `read` 实现里（运行在 ring 0）**替你执行真正的 `rdmsr` 指令**，把结果回给用户态。所以亲手碰 MSR 的还是内核，用户态只是走内核开的口子（`/dev/cpu/N/msr`，且要 root）间接拿到值——这反而印证了"特权寄存器只有内核能碰"。

两个地址一模一样——**`IA32_LSTAR` 里存的就是 `entry_SYSCALL_64` 的地址**。`syscall` 一执行，CPU 把 `RIP` 直接设成这个值，跳进去。

把两边并排看，会发现是同一件事的两种实现：

| | ARM64（本机实测） | x86-64（文档对照） |
|---|---|---|
| 入口地址存哪 | `VBAR_EL1`（系统寄存器） | `IA32_LSTAR`（MSR） |
| 指向 | 向量表 `vectors`，`svc` 落 `+0x400` | 直接是 `entry_SYSCALL_64` |
| 用户态能读吗 | 不能（实验 B：SIGILL） | `rdmsr` **指令** ring 3 会 #GP；`rdmsr` **工具**得 root 且走 `/dev/cpu/N/msr` 让内核代读 |
| 怎么验证它指对了 | kallsyms 有 `vectors` + 架构定的 `0x400` 偏移 | `rdmsr 0xc0000082` == kallsyms 的 `entry_SYSCALL_64` |

**`IA32_LSTAR` 之于 x86，约等于 `VBAR_EL1` 之于 ARM64**：都是"只有内核能写、用户态碰不到"的入口地址寄存器。这就是为什么用户态没法自己决定跳进内核哪里——门牌号写在用户态够不着的寄存器里。

## 四、`svc` / `syscall` 执行的那一瞬，硬件做了哪几件事

入口地址有了，现在看指令执行的原子瞬间。`svc`（或 `syscall`）不是"跳转"那么简单，它是 CPU 在**一个不可打断的步骤里**同时干好几件事——任何一件没做、或被中间打断，特权边界就漏了。

### 4.1 ARM64：`svc` 触发同步异常，硬件一气呵成

`svc #0` 执行时，ARM64 硬件按异常机制，原子地完成这些动作：

```text
svc #0 执行瞬间（CPU 硬件自动，不可打断）
┌────────────────────────────────────────────────────────┐
│ ① 提权：EL0 → EL1（用户态 → 内核态）                    │
│ ② 存返回地址：ELR_EL1 ← svc 的下一条指令地址            │
│ ③ 存处理器状态：SPSR_EL1 ← 当前 PSTATE                  │
│ ④ 关中断：PSTATE.DAIF 置位（屏蔽 IRQ/FIQ 等）           │
│ ⑤ 记录原因：ESR_EL1.EC ← 0x15（“AArch64 的 SVC”）       │
│ ⑥ 换栈：SP 自动切到 SP_EL1（内核栈）                    │
│ ⑦ 跳转：PC ← VBAR_EL1 + 0x400                           │
└────────────────────────────────────────────────────────┘
```

逐条说人话：

- **① 提权**是核心目的——从这一刻起 CPU 是 EL1，能访问内核页、能执行特权指令了。
- **②③ 存现场**：返回地址进 `ELR_EL1`、处理器状态进 `SPSR_EL1`。为什么要硬件存？因为一旦进了内核，这俩寄存器会被覆盖，得先冻起来，将来 `eret` 返回时才知道回哪、恢复成什么状态。
- **④ 关中断**：刚进内核、栈还没完全搭好的当口，不能被中断打断，先把中断屏蔽掉，等内核站稳了再开。
- **⑤ 记录原因**：`ESR_EL1`（异常综合寄存器）里 `EC=0x15` 明确写着"这是一条 AArch64 的 SVC"。第三篇里 `el0t_64_sync_handler` 读的就是它，用来区分"这次进来到底是系统调用、缺页、还是别的异常"。
- **⑥ 换栈**：这是**重点**。`svc` 走异常机制，硬件自动把 `SP` 从用户栈切到 EL1 的内核栈 `SP_EL1`。注意是**硬件自动切**，汇编不用管。
- **⑦ 跳转**：`PC` 设成 `VBAR_EL1 + 0x400`，CPU 落到向量表那一格，开始跑 `el0t_64_sync`。

### 4.2 x86-64：`syscall` 偷懒，但留了个坑（源码/文档对照）

x86-64 的 `syscall` 干的事**目的一样、但更省**（以下据 Intel SDM）：

```text
syscall 执行瞬间（CPU 硬件自动）
┌────────────────────────────────────────────────────────┐
│ ① 提权：CPL 3 → 0                                        │
│ ② 存返回地址：RCX ← RIP（下一条指令）                   │
│ ③ 存处理器状态：R11 ← RFLAGS                            │
│ ④ 关中断：RFLAGS ← RFLAGS & ~IA32_FMASK（清掉 IF 等）   │
│ ⑤ 换段：CS、SS ← IA32_STAR 里的内核段选择子             │
│ ⑥ 跳转：RIP ← IA32_LSTAR（= entry_SYSCALL_64）          │
│ ✗ 不换栈！RSP 原封不动，还是用户栈                      │
└────────────────────────────────────────────────────────┘
```

和 ARM64 对比，最大的差异在 **② 和 ⑥/换栈**：

- ARM64 把返回地址/状态存进**专用系统寄存器**（`ELR_EL1`/`SPSR_EL1`）；x86 图省事，直接塞进**通用寄存器** `RCX`/`R11`——代价是这俩寄存器的原值就被冲掉了，所以 x86 的系统调用 ABI 特意规定 `rcx`、`r11` 是调用者保存、会被 `syscall` 破坏。
- **`syscall` 根本不换栈**。执行完，`RSP` 还指着用户栈！这就是上面图里那个红叉。

这里还藏着一个容易被忽略的细节：**第 ⑤ 步"换段"是怎么换的？** 64 位长模式是平坦内存模型，`CS`/`SS` 的 base 被硬件当 0、limit 也基本不查，所以这一步换的**不是地址窗口，而是特权级**——CPL（当前特权级）就等于 `CS` 选择子的低 2 位，从 `__USER_CS`（低 2 位 = 3）换成 `__KERNEL_CS`（低 2 位 = 0），CPU 就从 ring 3 进了 ring 0。

更妙的是它**怎么换的**。先补一个硬件常识：段寄存器有两半——你能用指令读写的**可见部分**（16 位选择子，即 GDT 下标），和 CPU 内部挂着的**隐藏部分**（描述符缓存，存着 base/limit/DPL/L 位等真正干活的属性）。

> **GDT（Global Descriptor Table，全局描述符表）**是内存里的一张表，每条 8 字节，描述一个段的属性（base、limit、特权级 DPL、是不是 64 位代码段等）。段选择子就是这张表的下标。`__KERNEL_CS`、`__USER_CS` 这些就是表里某几条的"编号"。GDT 的完整结构、它和 LDT/TSS 的关系、Linux 具体怎么排布它，**后续会单独写一篇详细讲**，这里只需知道"选择子 = GDT 下标，描述符 = 表里那一条"。

普通的段装载（远跳、`iret`、`mov ss, ax`）流程是「**拿选择子当下标去 GDT 读那条 8 字节描述符 → 校验 → 抄进隐藏部分**」，之后访存只看隐藏部分，不再查 GDT。也就是说 **GDT 只在"装载那一刻"被读一次，"使用时"用的是隐藏缓存**。

而 `syscall` 是会改 `CS` 的指令里**唯一一条不查 GDT 的**——这些活全在它自己的微码里完成，不调用别的指令：

```text
syscall 改段（微码内部，64 位模式，简化自 Intel SDM / AMD64 手册）
  CS.可见部分 ← IA32_STAR[47:32] AND 0xFFFC   # 取选择子号，低 2 位强制清 0 → CPL=0
  CS.隐藏部分 ← 直接写死常量：Base=0, Limit 拉满, L=1(64位代码段), DPL=0, P=1
  SS.可见部分 ← IA32_STAR[47:32] + 8
  SS.隐藏部分 ← 直接写死常量：Base=0, Limit 拉满, DPL=0, 可读写数据段
```

关键就在那两行"隐藏部分 ← 直接写死常量"：微码**没有发起一次访存去读 GDT 那条描述符**，而是把属性用**指令自带的固定值**直接灌进隐藏缓存；可见部分（选择子号）则取自 `IA32_STAR` 这个 MSR（也是寄存器，不是内存）。省掉的这次"读 GDT + 校验"访存，正是 `syscall` 比"中断进内核"快的原因之一。`sysret` 回用户态同理，反方向用 `STAR[63:48]` 填一套固定值。

那为什么 Linux 还得把 `__KERNEL_CS`、`__KERNEL_DS`、`__USER_CS`、`__USER_DS` 在 GDT 里排成固定间距？因为**别的进内核路径会真查 GDT**：中断/异常走 IDT→GDT、`iret` 返回、`mov` 重装段寄存器，这些都老老实实拿选择子号去 GDT 抄描述符。而它们用的选择子号跟 `syscall` 写进 `CS`/`SS` 的是**同一批数字**。要让两条路径自洽，就必须保证「`syscall` 硬编码出来的属性」和「拿同一个选择子号去 GDT 查出来的属性」**是一回事**——所以 GDT 里 `STAR[47:32]` 那个槽得正好是"64 位内核代码段、DPL 0"，`+8` 得正好是"内核数据段"。间距错一格，快路径（syscall 填固定值）和慢路径（iret 查 GDT）对同一个 `__KERNEL_CS` 就会给出两套段，特权边界立刻乱。

第三个差异——**换栈**——值得单独拎出来，因为它牵出一个经典误解。

### 4.3 重点澄清：`syscall` 不换栈，TSS.RSP0 不是给它用的

很多资料会说"系统调用时 CPU 通过 TSS 里的 RSP0 切到内核栈"。**对 x86 的 `syscall` 指令来说，这是错的。**

> **TSS（Task State Segment，任务状态段）**是 x86 里一块特殊的系统结构（在 GDT 里有一条专门的描述符指向它）。早年它是给硬件任务切换用的，64 位下那套基本废弃了，如今主要就剩两个用处：存各特权级的栈指针 `RSP0/RSP1/RSP2`（特权级切换时硬件从这里取新栈顶），以及中断栈表 `IST`。它和 GDT、`syscall` 的协作细节，**后续会和 GDT 那篇一起单独详细讲**，这里只需知道"`RSP0` = 进 ring 0 时硬件要用的内核栈顶，但只有走中断门那条路才会用到"。

- `syscall` 指令**自己不碰 `RSP`**。进内核第一脚，栈指针还在用户栈上。所以 x86 内核的 `entry_SYSCALL_64` 一开头必须**软件手动换栈**：先 `swapgs` 拿到 per-CPU 数据，再把内核栈顶 `movq` 进 `rsp`。（这段汇编正是第三篇的内容。）
- **TSS 里的 `RSP0` 是给"中断/异常"路径用的**：当 `int`、缺页、时钟中断等**经过 CPU 的中断门**进内核、且发生特权级切换时，硬件才会自动从 TSS 加载 `RSP0` 当新栈。`syscall` 走的是另一条更轻的路，不碰 TSS。
- ARM64 这边反而是硬件自动换栈（`SP_EL1`），因为 `svc` 走的就是异常机制，和中断同一套，硬件统一处理。

一句话记牢：

> **ARM64 `svc`：硬件自动换内核栈。x86 `syscall`：不换栈，得内核汇编自己换。TSS.RSP0 是中断/异常用的，与 `syscall` 无关。**

为什么 x86 要这么设计？因为 `syscall` 是 AMD 为了**快**才造的——能少做一件事就少做一件，换栈这种"软件几条指令就能补上"的活，干脆甩给内核，硬件只保证最不能省的那几样（提权、存返回点、关中断、跳到可信入口）。代价就是内核入口汇编得自己擦屁股，也埋了"TSS.RSP0 到底归谁用"这个坑。

## 五、实测：从用户态看，这趟 ring 之旅是"隐形"的

理论讲完，回到能跑的 arm64，用 gdb 单步亲眼看一眼 `svc` 前后。

主角就是第一节那个完整的 `22_step_svc.c`（手写 `svc` 发 `write(1,"hi\n",3)`）。先反汇编找到 `svc` 的地址：

```bash
gdb -q -batch -ex 'break main' -ex run -ex 'disassemble main' ./22_step_svc | grep -E 'svc|x8'
```

```text
=> 0x0000000000400644 <+0>:	mov	x8, #0x40
   0x0000000000400658 <+20>:	svc	#0x0      ← 本次构建里 svc 在这
```

再写个 gdb 批处理脚本，停在 `svc` 上打印寄存器，`stepi` 跨过它后再打印一次：

```gdb
# step.gdb —— 单步 svc，对比前后寄存器（地址按上面反汇编填）
set pagination off
break *0x400658
run
printf "===== svc 之前 =====\n"
printf "pc = 0x%lx\n", $pc
printf "sp = 0x%lx\n", $sp
printf "x8 (syscall nr) = %ld\n", $x8
printf "x0 (fd / arg1)  = %ld\n", $x0
x/1i $pc
stepi
printf "===== svc 之后（已从内核返回）=====\n"
printf "pc = 0x%lx   (前进了 %ld 字节)\n", $pc, $pc - 0x400658
printf "sp = 0x%lx   (用户栈视角不变)\n", $sp
printf "x0 (返回值)     = %ld\n", $x0
x/1i $pc
```

```bash
gdb -q -batch -x step.gdb ./22_step_svc
```

真实输出：

```text
===== svc 之前 =====
pc = 0x400658
sp = 0xfffffffff9a0
x8 (syscall nr) = 64
x0 (fd / arg1)  = 1
=> 0x400658 <main+20>:	svc	#0x0

===== svc 之后（已从内核返回）=====
pc = 0x40065c   (前进了 4 字节)
sp = 0xfffffffff9a0   (用户栈视角不变)
x0 (返回值)     = 3
=> 0x40065c <main+24>:	mov	w0, #0x0
```

盯住这组数字，有意思的地方在于**看不见的东西**：

- `pc` 从 `0x400658` 变成 `0x40065c`，正好 **+4 字节**——`svc` 是一条 4 字节指令，执行完就落到下一条。从用户态看，它就像一条普普通通的顺序指令。
- `sp` **完全没变**，还是 `0xfffffffff9a0`。可第四节明明说了"硬件换栈到 `SP_EL1`"啊？——换了，但那是 **EL1 的栈**，进内核用、出内核前又换回来了。`eret` 返回时把 `SP` 恢复成用户栈，所以**用户态这一侧的 `sp` 看起来纹丝不动**。
- `x0` 从 `1`（参数 fd）变成 `3`（返回值，写了 3 个字节）。**参数进、返回值出，共用 `x0` 这个槽位**。

换句话说，gdb 单步看到的，是一次"输入在 `x0`/`x8`、输出在 `x0`"、`pc` 只前进 4 字节的操作——**和一次普通函数调用几乎没有区别**。中间那趟"EL0→EL1→跑向量表→跑内核函数→`eret` 回来"的惊险旅程，被硬件包得严严实实，用户态**一点都感觉不到**。

这正是这道门设计精妙的地方：它对用户态暴露的接口极简（设几个寄存器、执行一条指令、从 `x0` 取结果），把特权切换、栈切换、现场保存这些脏活全藏在硬件和内核里。第三篇会掀开盖子，看门后面那段汇编是怎么把这趟旅程兜住的。

> 顺带回收一个疑问：既然用户态看 `svc` 像函数调用，那它和真正的 `call`/`bl` 差在哪？差在 `bl` 不改特权级、不换栈、不读 `VBAR_EL1`——它只是同一特权级内的跳转。`svc` 多做的那几件事（提权、换栈、关中断、跳可信入口），每一件都是 `bl` 给不了的，也正是第二节那两个实验所证明的"墙"的另一面。

## 六、指令支持：arm64 天生有，x86 是后加的扩展

用户的实测链里有一步"读 `/proc/cpuinfo` 确认 CPU 支持"。这里两个架构的故事不一样。

**x86-64**：`syscall` 是 AMD64 引入的扩展指令，不是每颗老 x86 都有，所以 `/proc/cpuinfo` 的 `flags` 里会列一个 `syscall` 标志，程序也能用 `cpuid` 查（**x86 文档对照**）：

```text
# grep -o 'syscall' /proc/cpuinfo | head -1     # x86-64 上
syscall
```

**ARM64**：`svc` 是 ARMv8 架构**强制要求**的基础指令，每颗 armv8 CPU 必然有，不需要、也没有对应的特性标志位。`/proc/cpuinfo` 的 `Features` 列的是各种**可选**扩展，里头不会有 `svc`，因为它根本不是可选项：

```text
Features : fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp
           cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 ... bti afp
```

（真实输出，截了一段。)这些 `aes`/`sha2`/`bf16` 之类才是"看 CPU 支不支持"的东西；`svc` 不在此列，因为它是地基。所以在 arm64 上，"确认 CPU 支持系统调用指令"这一步，答案是"架构保证有"。

## 七、返回路径：`eret` / `sysret`

进去的反面是出来。内核活干完，要从 ring 0 退回 ring 3，用的是另一条专门指令，刚好是进入动作的镜像。

- **ARM64**：`eret`（exception return）。它从 `ELR_EL1` 取回返回地址、从 `SPSR_EL1` 恢复处理器状态、把特权级切回 EL0、`SP` 换回用户栈——正好对应第四节 ②③⑥ 的逆操作。执行完，CPU 落回用户程序里 `svc` 的下一条指令，`x0` 已是返回值。
- **x86-64**：`sysret`（或 `sysretq`）。把当年 `syscall` 存进 `RCX`/`R11` 的 `RIP`/`RFLAGS` 装回去，`CPL` 切回 3。

这就是为什么第四节强调"硬件要把返回地址和状态先存好"：不存，`eret`/`sysret` 就无家可归。进入和返回是一对严格配对的动作，少一半都回不去。返回路径上内核汇编那些寄存器恢复细节，第三篇会展开，这里只点出指令本身的对称性。

## 八、回到开头：那一行指令到底做了什么

现在可以回答开篇那个问题了。`write(1,"hello",5)` 里那条 `svc #0`（或 x86 的 `syscall`），做的是这样一件事——它是一道**受控的硬件门**：

```text
┌─────────────────────────────────────────────────────────────┐
│ 用户态 (EL0 / ring 3)                                        │
│   x8 = 64(nr)  x0 = 1(fd)  x1 = buf  x2 = 5(count)           │
│   svc #0   ◄── 你只能敲这一下，跳到哪、怎么提权，你说了不算  │
└───────────────────────────┬─────────────────────────────────┘
                            │  CPU 硬件原子地完成：
                            │  ① EL0 → EL1（提权）
                            │  ② ELR_EL1 ← 返回地址
                            │  ③ SPSR_EL1 ← 处理器状态
                            │  ④ 关中断 (DAIF)
                            │  ⑤ ESR_EL1.EC = 0x15（标记“这是 SVC”）
                            │  ⑥ SP → SP_EL1（硬件换内核栈）
                            │  ⑦ PC ← VBAR_EL1 + 0x400 ◄ 内核写死的可信入口
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 内核态 (EL1 / ring 0)                                        │
│   落在向量表 vectors + 0x400 → el0t_64_sync（第三篇接手）    │
│   ... 内核干活 ...                                           │
│   eret  ◄── 进入动作的镜像：恢复 PC/状态/栈，切回 EL0        │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 用户态继续，x0 = 5（返回值），pc = svc 的下一条               │
└─────────────────────────────────────────────────────────────┘
```

为什么不能 `jmp`，到这里答案就齐了：

1. **`jmp` 不提权**。跳过去你还是 ring 3，碰内核页直接段错误（实验 A）。
2. **`jmp` 让你自己挑落点**。而内核绝不能让用户态随便跳进它代码的任意位置——那等于把后门大开。入口地址被锁在用户态读都读不到的寄存器里（实验 B），只能由 `svc`/`syscall` 按架构规则送你到**那唯一一个**入口。
3. **`jmp` 不换栈、不关中断、不存返回点**。而跨特权边界这几件事一件都不能少，还必须**原子**完成。只有专门的指令配合硬件才办得到。

所以 `svc`/`syscall` 不是"一种跳转的写法"，而是 CPU 体系结构里专门为"安全地跨过特权边界"造的一道门：**用户态只有敲门的权利，开门、领路、升权全归硬件按内核预设的规矩办。**

我们用真机数据钉死了这道门的几个关键面：

1. libc 的 `write` 里确实是 `mov x8,#64` + `svc #0`，且不依赖 libc 也能手写。
2. 直接 `call` 内核地址 → **SIGSEGV**，证明 `jmp` 这条路被页表堵死。
3. EL0 读 `VBAR_EL1` → **SIGILL**，证明入口地址对用户态不可见、更不可改。
4. kallsyms 里 `vectors`（VBAR_EL1 目标）和 `el0t_64_sync` 真实存在，`svc` 落点 = `vectors + 0x400`。
5. gdb 单步：`svc` 前后 `pc` 只 +4、`sp` 不变、`x0` 从 `1` 变 `3`——整趟 ring 之旅对用户态完全隐形。

至于 CPU 落到 `vectors + 0x400` 之后，那段汇编怎么把 31 个寄存器存成 `struct pt_regs`、怎么把现场交给 C 函数——那是第三篇《[内核入口 el0_svc / entry_SYSCALL_64 的汇编做了什么](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/三、内核入口el0_svc做了什么——从异常向量到C函数.md)》的事了。这一篇把 CPU 送到了门口，下一篇推门进去。

## 九、参考资料

- ARM 架构手册（ARM ARM）：异常向量表布局、`VBAR_EL1` / `ELR_EL1` / `SPSR_EL1` / `ESR_EL1`、`svc` 与 `eret` 语义
- Intel SDM Vol. 2/3：`SYSCALL` / `SYSRET` 指令、`IA32_LSTAR` / `IA32_STAR` / `IA32_FMASK` / `IA32_EFER.SCE`
- Linux 内核源码（v6.12）：
  - [`arch/arm64/kernel/entry.S`](https://github.com/torvalds/linux/blob/v6.12/arch/arm64/kernel/entry.S)：异常向量表 `vectors`、`el0t_64_sync`
  - [`arch/x86/entry/entry_64.S`](https://github.com/torvalds/linux/blob/v6.12/arch/x86/entry/entry_64.S)：`entry_SYSCALL_64`、入口手动换栈、`sysret`
- `man 2 syscall`：系统调用的用户态接口与各架构寄存器约定
- `Documentation/arm64/booting.rst` / `memory.rst`：ARM64 地址空间高低半区划分

---

上一篇：[printf("hello") 怎么变成 write(1, "hello", 5)——libc 的 stdout 缓冲机制](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/一、printf怎么变成write——libc的stdout缓冲机制.md)
下一篇：[内核入口 el0_svc / entry_SYSCALL_64 的汇编做了什么——从异常向量到 C 函数](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/三、内核入口el0_svc做了什么——从异常向量到C函数.md)

完整系列：

- **第一篇**：[printf → write（libc 缓冲层）](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/一、printf怎么变成write——libc的stdout缓冲机制.md)
- **第二篇（本文）**：`svc` / `syscall` 指令的硬件行为（从 ring3 到 ring0 的硬件门）
- **第三篇**：[`el0_svc` / `entry_SYSCALL_64` 汇编入口（从异常向量到 C 函数）](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/三、内核入口el0_svc做了什么——从异常向量到C函数.md)
- **第四篇**：[write → ksys_write（sys_call_table 派发）](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/四、从write到ksys_write——sys_call_table怎么路由的.md)
