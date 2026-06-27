# 三、内核入口 el0_svc / entry_SYSCALL_64 的汇编做了什么——从异常向量到 C 函数

---

> **🎯 交互式可视化**：[syscall-entry-visualizer.html](../front/syscall-entry-visualizer.html)
> `svc` / `syscall` 指令落地后，内核入口那段汇编怎么一步步建好现场：切换 per-CPU 指针、把寄存器全压进 `struct pt_regs`、再跳进 C 函数 `do_el0_svc()` 的全过程动画。可切 ARM64 / x86-64 两条路对照，逐步看 CPU 寄存器变化与内核栈上 `pt_regs` 一格格填满。

---

> **系列说明**：这是"一句 printf 怎么出现在屏幕上"系列的第三篇。前两篇分别讲了：第一篇《[printf("hello") 怎么变成 write(1, "hello", 5)](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/一、printf怎么变成write——libc的stdout缓冲机制.md)》讲清 libc 的 stdout 缓冲，把数据交到 `write()` 系统调用手上；第二篇《[`svc` / `syscall` 指令到底做了什么——从 ring 3 到 ring 0 的硬件门](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/二、syscall指令做了什么——从ring3到ring0的硬件门.md)》讲清那条指令的硬件行为，CPU 怎么从用户态跳到内核预设的入口地址。这一篇接着往下走：**内核在那个入口地址上放了什么代码？** 这段汇编怎么保存现场、建好 `struct pt_regs`，再把控制权交给 C 函数。最后一篇《[从 write 到 ksys_write——sys_call_table 怎么路由的](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/四、从write到ksys_write——sys_call_table怎么路由的.md)》接着讲内核怎么查表路由到 `ksys_write`。

---

上一篇讲到：`svc #0`（ARM64）或 `syscall`（x86-64）执行后，CPU 做了一连串硬件动作——切换特权级、关中断、把返回地址和处理器状态存进系统寄存器，然后**跳到一个内核预先设好的入口地址**。在 ARM64 上这个地址来自异常向量表基址寄存器 `VBAR_EL1`；在 x86-64 上来自 `IA32_LSTAR` MSR。

这一篇的问题就是接着上一篇的结尾：**内核在那个入口地址上，到底放了什么代码？**

答案是一段汇编。它必须在"C 代码能跑起来"之前，亲手把运行现场搭好——**为什么不能直接跳到 C 函数？因为 `svc` 跳进来的那一刻，CPU 处于一个"半成品"状态**：

- 特权级切到了内核（ARM64 从 EL0 到 EL1，x86-64 从 ring 3 到 ring 0），但**寄存器里装的还全是用户态的值**——`x0`~`x30` / `rax`/`rdi`/`rsi` 等通用寄存器，以及栈指针、per-CPU 指针，都还指向用户态的数据。
- 内核自己的上下文基准指针（ARM64 的 `sp_el0`、x86-64 的 `GS`）还没切过来，**内核连"当前任务是谁"都不知道**（访问 `current` 会读到错误地址）。
- 用户态寄存器里的值还散着，**没有一个统一的数据结构来存它们**——待会儿要返回用户态、处理信号、调试器 `ptrace` 查看现场，都需要这些值被完整保存下来。

所以，这段汇编做完三件事，C 函数才接得住：

1. **建立内核上下文**：切换到内核的 per-CPU / per-task 数据指针（ARM64 切 `sp_el0`，x86-64 用 `swapgs`）。
2. **保存所有寄存器**：把用户态的 31 个通用寄存器全压进内核栈，拼成一个 `struct pt_regs`。
3. **跳进 C 函数**：把 `pt_regs` 的指针作为参数，调用 `do_el0_svc()`（ARM64）/ `do_syscall_64()`（x86-64）。

下面我们在真机上一层层把它扒开。

> **实验环境**：Docker 容器，`gcc:13` 镜像；内核 `Linux 6.12.65-linuxkit aarch64`，glibc `2.36-9+deb12u14`。本机是 **ARM64**，所以所有 `/proc/kallsyms`、`bpftrace` 抓取都是真编真跑的 **arm64 实测**，主角是 `el0_svc`。x86-64 的 `entry_SYSCALL_64` 在这台机器上跑不到（Docker 用 QEMU 把 x86 二进制的系统调用翻译成了 arm64，底层内核仍是 arm64），所以 x86 部分是**读 Linux 6.12 源码 `arch/x86/entry/entry_64.S` 做的对照讲解**，会明确标注。两条路在结构上高度同构，对照着看反而最清楚。符号地址会因内核版本/编译不同而变，这里只看关系，不看绝对数字。

## 一、先把入口找出来：/proc/kallsyms

内核里每个函数都有名字和地址，`/proc/kallsyms` 把它们全列出来。先找跟系统调用入口相关的几个符号：

```bash
grep -iE "el0t_64_sync|el0_svc|do_el0_svc|invoke_syscall" /proc/kallsyms
```

真实输出：

```text
ffff800080011340 t el0t_64_sync
ffff800080023338 t invoke_syscall.constprop.0
ffff800080023430 T do_el0_svc
ffff8000812e2708 t el0_svc
ffff8000812e34f0 T el0t_64_sync_handler
```

这里有五个名字，按 `svc` 指令执行后内核的真实经过顺序，它们是：

- **`el0t_64_sync`**：异常向量表里"从 64 位 EL0 进来的同步异常"那一格（VBAR_EL1 + 0x400）。`svc` 触发同步异常后，CPU 第一脚踏进的就是这里。这是一段汇编入口代码，负责保存寄存器、建立 `pt_regs`。`t` 表示局部代码符号。
- **`el0t_64_sync_handler`**：向量表入口汇编跳过来的第一个 C 函数。它读取 **ESR_EL1 寄存器**（异常触发时硬件自动写入异常原因）的 **EC 字段**（bit[31:26]，异常类型编码），判断"这是哪种同步异常"——`0x15` 表示 SVC（系统调用），`0x24` 是数据访问异常（缺页），`0x3C` 是断点等。确认是 SVC 后，调到 `el0_svc`。
- **`el0_svc`**：`el0t_64_sync_handler` 确认异常类型是 SVC（系统调用）后，会再调回这个汇编函数做进一步处理，然后它调到 `do_el0_svc`。
- **`do_el0_svc`**：接收 `pt_regs` 指针，从 `regs->regs[8]` 读出系统调用号，准备派发。
- **`invoke_syscall`**：真正拿系统调用号去查 `sys_call_table` 并执行的函数（`.constprop.0` 是编译器常量传播优化后改的名字）。

在 x86-64 上，同一个 `grep` 会看到的是另一组名字：

```text
（x86-64，源码对照，非本机输出）
entry_SYSCALL_64          ← IA32_LSTAR 指向的入口
do_syscall_64             ← 派发函数
```

ARM64 的入口叫 `el0_svc`（EL0 来的 supervisor call），x86-64 的入口叫 `entry_SYSCALL_64`（`syscall` 指令的入口）。**名字不同，干的活几乎一样**。

## 二、入口链长什么样：用 bpftrace 抓一条真实的内核栈

光看符号名还不够，得证明这几个函数确实是"一条链"——`svc` 之后内核真的是从向量表一路走到系统调用实现的。用 `bpftrace` 在最末端的 `__arm64_sys_write` 上打个探针，把当时的内核调用栈打印出来，就能反推出整条路径。

先写个会暂停、然后调用 `write` 的小程序，方便我们从容地挂探针：

```c
// 10_entry_trace.c —— 暂停后调用 write，方便在内核入口捕获
#include <unistd.h>
#include <stdio.h>

int main(void) {
    printf("PID: %d\n", getpid());
    printf("Press Enter to call write...\n");
    getchar();

    long n = write(1, "hello\n", 6);   // x8=64, x0=1, x2=6 -> 返回 6

    fprintf(stderr, "write returned %ld\n", n);
    return 0;
}
```

```bash
gcc -O0 -o 10_entry_trace 10_entry_trace.c
```

bpftrace 脚本，抓到 `write` 进内核时的栈就立即打印并退出：

```c
// stack.bt
kprobe:__arm64_sys_write
/comm == "10_entry_trace"/
{
    printf("=== kernel stack at __arm64_sys_write ===\n%s\n", kstack);
    exit();
}
```

一个终端跑 `sudo bpftrace stack.bt`，另一个终端跑 `./10_entry_trace` 并按回车。真实输出：

```text
=== kernel stack at __arm64_sys_write ===

        __arm64_sys_write+0
        do_el0_svc+72
        el0_svc+40
        el0t_64_sync_handler+288
        el0t_64_sync+400
```

**这就是 `svc #0` 之后内核走过的完整路径，自底向上读**：

```text
el0t_64_sync           ← ① svc 触发同步异常，CPU 跳进向量表这一格（汇编）
    ↓
el0t_64_sync_handler   ← ② C 处理函数：读 ESR 判断异常类型
    ↓
el0_svc                ← ③ 确认是 SVC（系统调用）
    ↓
do_el0_svc             ← ④ 取出系统调用号，准备派发
    ↓
__arm64_sys_write      ← ⑤ 查表命中，进到 write 的内核实现
```

注意 `el0t_64_sync`（向量表入口）和它后面那段是**汇编**，`el0t_64_sync_handler` 往后才是 **C**。汇编与 C 的分界线，正是这一篇的主题：汇编必须先把现场搭好，C 才接得住。

> 小坑：在这个 linuxkit 内核上，bpftrace 给 `do_el0_svc` 会先打印一句 `WARNING: ... not traceable`，但 kprobe 实际是挂上的、数据照样抓得到，可以忽略。

x86-64 上对应的栈会是这样（源码对照）：

```text
（x86-64，结构对照）
__x64_sys_write
do_syscall_64
entry_SYSCALL_64       ← syscall 指令的入口，对应 arm64 的 el0t_64_sync/el0_svc
```

x86-64 没有"向量表"这一层——`syscall` 指令直接把 `rip` 设成 `IA32_LSTAR` 里存的 `entry_SYSCALL_64` 地址，一步到位。ARM64 因为 `svc` 是走异常机制，要先过一遍向量表（`el0t_64_sync`），再分流到 `el0_svc`。这是两条路第一个、也是最大的结构差异。

## 三、汇编做的第一件事：建立内核上下文

现在进到向量表入口那段汇编内部。`svc` 跳进来时，CPU 处于一个很"裸"的状态：特权级是内核了，但**寄存器里装的还全是用户态的值，连内核自己的 per-CPU 数据指针都还没指对**。所以汇编做的第一件事，是把"内核视角"的基准指针切回来。

### 3.1 ARM64：切换 sp_el0（per-task 指针）

ARM64 内核在 EL1（内核态）运行时，把 `sp_el0` 寄存器用作 **`current` 任务指针**（不当栈用，内核栈用 `sp_el1`）。但 `svc` 触发异常进入内核时，`sp_el0` 还装着**用户态的栈指针**（用户程序在 EL0 时用它当栈），所以入口汇编第一步要把它换成当前任务的 `task_struct` 指针。这段在 `kernel_entry` 宏里（`arch/arm64/kernel/entry.S`）：

```asm
    .macro  kernel_entry, el, regsize = 64
    ...
    .if \el == 0                       ; 仅当从 EL0（用户态）进来
    clear_gp_regs                      ; 清掉用户态可能残留的寄存器
    mrs x21, sp_el0                    ; 把用户态的 sp_el0 暂存到 x21
    ldr_this_cpu tsk, __entry_task, x20; 取出当前 CPU 上的当前任务指针
    msr sp_el0, tsk                    ; sp_el0 <- 当前任务（内核的 current）
    .endif
```

`mrs x21, sp_el0` 先把用户值保存好（待会儿要还原），`msr sp_el0, tsk` 把它换成内核认的 `current`。这一换，内核里所有 `current->...` 才指得对。

### 3.2 x86-64：SWAPGS（你给的核心概念之一）

x86-64 用的是另一套机制，但目的一模一样。x86 有个 `GS` 段寄存器，内核把它指向 **per-CPU 数据区**（`current_task`、内核栈顶、TSS 等都在里面）。用户态的 `GS` 指向 **TLS 线程局部存储**（线程私有变量），所以进内核第一条指令就是 `swapgs`——它把 **`IA32_GS_BASE` MSR**（当前生效的 `GS` 基址）和 **`IA32_KERNEL_GS_BASE` MSR**（内核预存的 per-CPU 数据区地址）对调，一条指令完成用户态 ↔ 内核态的上下文切换（源码 `arch/x86/entry/entry_64.S`）：

```asm
SYM_CODE_START(entry_SYSCALL_64)
    ...
    swapgs                                          ; 用户 GS <-> 内核 GS
    /* tss.sp2 is scratch space. */
    movq    %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2) ; 先把用户栈指针暂存
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp           ; 切到内核页表（KPTI）
    movq    PER_CPU_VAR(pcpu_hot + X86_top_of_stack), %rsp  ; rsp <- 内核栈顶
```

注意 x86-64 这里比 ARM64 多一步：**手动换栈**。`syscall` 指令本身**不切换栈指针**，`rsp` 进来时还是用户栈，所以必须 `movq ...top_of_stack, %rsp` 亲手把内核栈顶装进 `rsp`。而 ARM64 的 `svc` 走异常机制，硬件自动把栈切到了 EL1 的内核栈，汇编不用管这一步。

把这两件事并排看：

| 目的 | ARM64（实测路径） | x86-64（源码对照） |
|------|------------------|-------------------|
| 切到内核的"当前任务/per-CPU"指针 | `msr sp_el0, tsk` | `swapgs` |
| 切到内核栈 | `svc` 硬件自动切 SP_EL1 | `movq pcpu_hot+top_of_stack, %rsp`（手动） |
| 切内核页表（如开了页表隔离） | （ARM64 走 TTBR/向量蹦床） | `SWITCH_TO_KERNEL_CR3` |

**`SWAPGS` 之于 x86，约等于 `msr sp_el0, tsk` 之于 ARM64**：一句话切换内核的"上下文基准指针"，让后面的代码能正确找到 per-CPU / current。

## 四、第二件事：把寄存器全压进 struct pt_regs

上下文切好了，但用户态那 31 个通用寄存器里的值还散着——里面有系统调用号、有参数，待会儿还要原样还给用户态。汇编得把它们**整整齐齐存到内核栈上一块固定布局的内存里**，这块内存就是 `struct pt_regs`。

### 4.1 ARM64：连续的 stp 指令

紧接着 `kernel_entry` 宏，是一长串 `stp`（store pair，一次存两个寄存器）：

```asm
    stp x0, x1,   [sp, #16 * 0]    ; x0,x1  -> pt_regs->regs[0..1]
    stp x2, x3,   [sp, #16 * 1]    ; x2,x3  -> regs[2..3]
    stp x4, x5,   [sp, #16 * 2]
    stp x6, x7,   [sp, #16 * 3]
    stp x8, x9,   [sp, #16 * 4]    ; x8 = 系统调用号，存进 regs[8]
    stp x10, x11, [sp, #16 * 5]
    ...
    stp x28, x29, [sp, #16 * 14]   ; 一直存到 x29
```

`sp` 此刻指向内核栈上为 `pt_regs` 预留的空间。`x0,x1` 存到偏移 0、`x2,x3` 存到偏移 16……一路到 `x29`。后面还有指令把 `x30`（链接寄存器）、`sp`、`pc`、`pstate` 也存进去。存完，内核栈上就躺着一个完整的 `struct pt_regs`，结构是（`arch/arm64/include/asm/ptrace.h`）：

```c
struct pt_regs {
    u64 regs[31];   // x0 ~ x30，按下标一一对应
    u64 sp;
    u64 pc;
    u64 pstate;
    ...
};
```

关键点：**`regs[N]` 就是用户态 `xN` 的值，下标对得严丝合缝**。所以 `regs[8]` 是系统调用号，`regs[0]` 是第一个参数 / 返回值。第五节我们会用 bpftrace 直接读这个偏移，亲眼验证。

### 4.2 x86-64：手搭硬件帧 + PUSH_AND_CLEAR_REGS

x86-64 因为 `syscall` 指令"省事"——它不像中断那样自动把 `SS/RSP/RFLAGS/CS/RIP` 压栈，而是把返回地址塞进 `rcx`、把 `rflags` 塞进 `r11` 就走了。所以入口汇编得**手动把这个"硬件帧"补出来**，让 `pt_regs` 的头部长得跟中断进来时一样（源码 `entry_64.S`）：

```asm
    /* Construct struct pt_regs on stack */
    pushq   $__USER_DS      /* pt_regs->ss   */
    pushq   PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  /* pt_regs->sp = 之前暂存的用户栈 */
    pushq   %r11            /* pt_regs->flags = syscall 存到 r11 的 rflags */
    pushq   $__USER_CS      /* pt_regs->cs   */
    pushq   %rcx            /* pt_regs->ip = syscall 存到 rcx 的返回地址 */
    pushq   %rax            /* pt_regs->orig_ax = 系统调用号 */

    PUSH_AND_CLEAR_REGS rax=$-ENOSYS   ; 把 rdi/rsi/rdx/r10/r8/r9/... 全压进去
```

`pushq %rcx`（返回地址）和 `pushq %r11`（rflags）这两行，正好对应 `syscall` 指令把 `rip`/`rflags` 顺手存进 `rcx`/`r11` 的硬件行为——汇编在这里"接住"它们，放进 `pt_regs` 的标准位置。最后 `PUSH_AND_CLEAR_REGS` 把剩下的通用寄存器一股脑压栈，并顺手清零（防止把内核数据泄漏给推测执行攻击）。

存完，x86-64 的内核栈上也躺着一个 `struct pt_regs`，只是字段顺序和 ARM64 不同（`rax` 在 `orig_ax`，参数在 `di/si/dx/r10/...`）。

**两边殊途同归**：ARM64 用一串 `stp` 平铺直叙地存，x86-64 先补硬件帧再 `PUSH_AND_CLEAR_REGS`，但结果都是——**内核栈上出现一个完整的 `pt_regs`，把整个用户态现场冻结下来**。这块 `pt_regs` 就是接下来 C 函数的唯一输入。

## 五、第三件事：从汇编跳进 C，传的就是 pt_regs 指针

汇编搭好 `pt_regs`、切好上下文，最后一步是**跳进 C**。它怎么把那块 `pt_regs` 交给 C 函数？答案是按调用约定，把 `pt_regs` 的地址放进"第一个参数寄存器"。

x86-64 这一步特别直白（`entry_64.S`）：

```asm
    movq    %rsp, %rdi        ; rdi = 第一个参数 = 指向 pt_regs 的指针（栈顶就是 pt_regs）
    movslq  %eax, %rsi        ; rsi = 第二个参数 = 系统调用号（符号扩展）
    ...
    call    do_syscall_64     ; do_syscall_64(regs, nr)
```

`movq %rsp, %rdi` 是点睛之笔：刚才所有 `push` 都压在栈上，`rsp` 现在正好指向 `pt_regs` 的开头，把它拷进 `rdi`（x86-64 的第一个参数寄存器），`pt_regs*` 就传过去了。然后 `call do_syscall_64`——**从这条 `call` 开始，就是纯 C 的世界了**。

ARM64 这边由 `el0t_64_sync` 向量代码完成等价动作：把 `pt_regs` 指针放进 `x0`（第一个参数寄存器），调到 `el0t_64_sync_handler(regs)`，再分流到 `el0_svc(regs)` → `do_el0_svc(regs)`。

所以"汇编到 C"的交接，本质就是一句话：**把内核栈顶（=pt_regs 起始地址）装进第一个参数寄存器，然后 `call`。** C 函数 `do_syscall_64` / `do_el0_svc` 接过这个指针，整个用户态现场就尽在它掌握了。

这也回答了一个常被问到的问题：**为什么不能让用户态直接用普通 C 调用约定（参数放 `rdi/rsi/...`）调内核函数，非要中间隔一个 `pt_regs`？** 因为系统调用跨越了特权边界：内核不能信任用户寄存器、必须把它们冻结存档（用于出错恢复、`ptrace`、信号处理、`fork` 复制现场等），而且参数寄存器约定本身就和函数调用约定不同（比如 ARM64 系统调用第 4 参用 `x3`，但还有 `x8` 装调用号）。`pt_regs` 就是这道边界上的"标准交接表格"。

## 六、实测：进入时 x8=64（__NR_write），出来时 x0=返回值（=6）

理论说完，来验证最核心的那句话：**系统调用号是怎么从用户寄存器变成 `pt_regs` 字段、又怎么被 C 代码读到的**。

我们在派发函数 `do_el0_svc(struct pt_regs *regs)` 上挂探针。上一节讲了，这个 `regs` 参数是汇编把 `pt_regs` 指针放进 `x0` 传过来的。既然第四节确认了 `regs[N]` 在结构体最前面、`regs[8]` 偏移是 `8 × 8 = 64` 字节、`regs[0]` 偏移是 0，我们就直接按偏移把字段读出来：

```c
// entry.bt —— 在派发函数 do_el0_svc 上抓 pt_regs
// arg0 = struct pt_regs* ；regs[N] 偏移 = N*8
kprobe:do_el0_svc
/comm == "10_entry_trace"/
{
    $regs = arg0;
    $nr = *(uint64 *)($regs + 64);   // regs[8] = x8 = 系统调用号
    $x0 = *(int64  *)($regs + 0);    // regs[0] = x0 = 参数1 (fd)
    $x1 = *(uint64 *)($regs + 8);    // regs[1] = x1 = 参数2 (buf)
    $x2 = *(uint64 *)($regs + 16);   // regs[2] = x2 = 参数3 (count)
    if ($nr == 64) {                 // 只看 write
        @p[tid] = $regs;
        printf("ENTER do_el0_svc: x8(nr)=%d  x0(fd)=%d  x1(buf)=0x%lx  x2(count)=%d\n",
               $nr, $x0, $x1, $x2);
    }
}

kretprobe:do_el0_svc
/comm == "10_entry_trace"/
{
    if (@p[tid]) {
        $regs = @p[tid];
        $ret = *(int64 *)($regs + 0);  // 系统调用执行完，regs[0] 已被改写成返回值
        printf("LEAVE do_el0_svc: x0(ret)=%d\n", $ret);
        delete(@p[tid]);
    }
}
```

跑起来，让 `10_entry_trace` 调一次 `write(1, "hello\n", 6)`。真实输出：

```text
ENTER do_el0_svc: x8(nr)=64  x0(fd)=1  x1(buf)=0x400900  x2(count)=6
LEAVE do_el0_svc: x0(ret)=6
```

**这就是整篇文章最关键的一组数字**，逐项拆开：

- `x8(nr)=64`：进内核时，系统调用号 `64`（ARM64 上 `__NR_write`）确实躺在 `pt_regs->regs[8]` 里。第四节那条 `stp x8, x9, [sp, #16*4]` 把它存到这儿，C 代码现在原样读出来。
- `x0(fd)=1`：第一个参数 `fd=1`（标准输出），在 `regs[0]`。
- `x2(count)=6`：第三个参数 `count=6`，在 `regs[2]`。
- `x0(ret)=6`：系统调用执行完，**同一个 `pt_regs` 的 `regs[0]` 被改写成了返回值 6**（写出去 6 个字节）。

进来时 `regs[0]` 是参数 `fd=1`，出去时变成返回值 `6`——**参数和返回值共用 `x0`/`regs[0]` 这个槽位**，这正是寄存器调用约定的设计。x86-64 上则是 `rax` 兼任系统调用号（进）和返回值（出），所以你给的那条线"进入 `rax=1`（`__NR_write`），出来 `rax=5`（返回值）"，在 ARM64 上的等价物就是这里的"进入 `x8=64`，出来 `x0=6`"。

> x86-64 的 `write` 调用号是 `1`，ARM64 是 `64`；你举的"写 5 字节返回 5"和这里"写 6 字节返回 6"是同一回事，数字差异只因架构和字符串长度不同。**机制完全一致：调用号从约定寄存器进、经 `pt_regs` 落地、被 C 读出；返回值从同一结构体的某个字段写回、再装回寄存器还给用户态。**

顺带，那次运行里 bpftrace 还抓到另外两条 `write`：

```text
ENTER do_el0_svc: x8(nr)=64  x0(fd)=2  x1(buf)=0xffffe3...  x2(count)=17   ; fprintf 到 stderr
LEAVE do_el0_svc: x0(ret)=17
ENTER do_el0_svc: x8(nr)=64  x0(fd)=1  x1(buf)=0x312a...    x2(count)=38   ; stdout 缓冲一把 flush
LEAVE do_el0_svc: x0(ret)=38
```

那条 `fd=1, count=38` 很有意思：程序里两句 `printf`（"PID: ..." + "Press Enter..."）一共 38 个字节，被 libc 的 stdout 缓冲攒着，直到 `getchar()` 或程序退出才一次性 `write` 出来——这正是本系列第一篇讲的缓冲机制，在内核入口这一层又得到一次印证。

### C 代码是怎么读到 regs[8] 的：do_el0_svc 源码

我们按偏移 64 读出了系统调用号，内核自己也是这么干的，只是写法是结构体成员。`do_el0_svc` 的源码（`arch/arm64/kernel/syscall.c`）短得只有一行实质内容：

```c
void do_el0_svc(struct pt_regs *regs)
{
    el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);
    //                    ^^^^^^^^^^^^ 就是我们按偏移 64 读的那个 x8
}
```

`regs->regs[8]` 和我们 bpftrace 里的 `*(uint64*)(arg0 + 64)` 是同一个东西——一个用结构体成员名，一个用裸偏移。这就闭环了：**汇编用 `stp` 把 `x8` 存进 `regs[8]`，C 用 `regs->regs[8]` 把它读出来，作为系统调用号去查表。**

再往下，`el0_svc_common` → `invoke_syscall` 真正查表并调用，最后把返回值写回 `pt_regs`（`arch/arm64/kernel/syscall.c`）：

```c
static void invoke_syscall(struct pt_regs *regs, unsigned int scno, ...)
{
    long ret;
    if (scno < sc_nr) {
        syscall_fn_t syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
        ret = __invoke_syscall(regs, syscall_fn);   // 调到 __arm64_sys_write
    } else {
        ret = do_ni_syscall(regs, scno);
    }
    syscall_set_return_value(current, regs, 0, ret); // 把返回值写回 regs->regs[0]
}
```

`syscall_set_return_value(...)` 把 `ret` 写进 `regs->regs[0]`——这就是我们在 `kretprobe` 里看到 `regs[0]` 从 `1` 变成 `6` 的那一刻。（查表路由 `sys_call_table[64] → __arm64_sys_write` 的细节是本系列最后一篇的主题，这里只看返回值怎么落回 `pt_regs`。）

## 七、返回路径：eret / SYSRET 前的寄存器恢复

C 函数跑完，返回值已经写回 `pt_regs->regs[0]`（x86 是 `pt_regs->ax`）。现在要原路返回用户态：**把 `pt_regs` 里的寄存器一个个倒回 CPU，再执行返回指令**。这是进入路径的镜像。

ARM64 用 `kernel_exit` 宏，最后一条是 `eret`（`arch/arm64/kernel/entry.S`）：

```asm
    ...
    ldr lr, [sp, #S_LR]          ; 恢复 x30
    add sp, sp, #PT_REGS_SIZE    ; 弹掉 pt_regs，栈复位
    ...
    eret                         ; 异常返回：恢复 PC/PSTATE，切回 EL0（用户态）
```

`eret` 会从系统寄存器里恢复返回地址和处理器状态，把特权级切回 EL0，CPU 落回用户程序里 `svc` 的下一条指令，此时 `x0` 已经是返回值。

x86-64 走 `sysret` 路径，是 `entry_SYSCALL_64` 进入动作的精确逆操作（`entry_64.S`）：

```asm
syscall_return_via_sysret:
    POP_REGS pop_rdi=0        ; 把 pt_regs 里的通用寄存器倒回去
    ...
    movq    RSP-RDI(%rdi), %rsp  ; 恢复用户栈指针
    ...
    swapgs                   ; GS 换回用户态（与入口的 swapgs 配对）
    sysretq                  ; 返回用户态，rcx->rip, r11->rflags
```

注意末尾的 `swapgs`——它和第三节入口处的 `swapgs` 严格配对：进来换成内核 `GS`，出去换回用户 `GS`。`sysretq` 则把当年 `syscall` 存进 `rcx`/`r11` 的 `rip`/`rflags` 装回去，CPU 跳回用户态。

**对称之美**：入口 `swapgs → 换栈 → 建 pt_regs → call C`，出口就是 `恢复 pt_regs → 换栈 → swapgs → sysret`。ARM64 同理：入口 `msr sp_el0 → stp 存 regs`，出口 `ldp 恢复 → eret`。进出严格镜像，这样才能保证用户程序"感觉不到"自己刚被冻结又解冻过一回。

## 八、两条路的完整对照：ARM64 el0_svc 与 x86-64 entry_SYSCALL_64

把前面章节散落的代码片段拼成两张完整的流程图，并排对照。

### 8.1 ARM64：el0t_64_sync → el0_svc 一条龙

ARM64 的入口路径（`arch/arm64/kernel/entry.S`，本机实测）：

```asm
// 异常向量表入口（VBAR_EL1 + 0x400）
el0t_64_sync:
    kernel_entry 0                   ; ① 展开后是：
    // —— kernel_entry 宏内部 ——
    sub sp, sp, #PT_REGS_SIZE        ; 在内核栈（SP_EL1，硬件已切好）上预留 pt_regs 空间
    stp x0, x1,   [sp, #16 * 0]      ; ② 把用户态寄存器连续存进 pt_regs
    stp x2, x3,   [sp, #16 * 1]
    ...
    stp x8, x9,   [sp, #16 * 4]      ; x8 = 系统调用号，存进 regs[8]
    ...
    stp x28, x29, [sp, #16 * 14]
    mrs x21, sp_el0                  ; 暂存用户态的 sp_el0
    ldr_this_cpu tsk, __entry_task, x20  ; 取出当前任务指针
    msr sp_el0, tsk                  ; ③ sp_el0 <- current（内核上下文基准）
    
    mov x0, sp                       ; ④ x0 = pt_regs*（第一个参数）
    bl  el0t_64_sync_handler         ; ⑤ 跳到 C 函数，判断异常类型
    // —— el0t_64_sync_handler 内部（C）——
    //     读 ESR_EL1，确认是 SVC（0x15）
    //     → 调回汇编 el0_svc → 再调 C 的 do_el0_svc(regs)
    //     → invoke_syscall → sys_call_table[64] → __arm64_sys_write
    //     → syscall_set_return_value 写回 regs[0]
    
    kernel_exit 0                    ; —— 返回路径 ——
    // —— kernel_exit 宏内部 ——
    msr sp_el0, x21                  ; 恢复用户态的 sp_el0
    ldp x0, x1,   [sp, #16 * 0]      ; 恢复所有寄存器（regs[0] 已是返回值）
    ldp x2, x3,   [sp, #16 * 1]
    ...
    add sp, sp, #PT_REGS_SIZE        ; 弹掉 pt_regs
    eret                             ; 返回用户态：恢复 PC/PSTATE，切回 EL0
```

### 8.2 x86-64：entry_SYSCALL_64 一条龙

x86-64 的入口路径（`arch/x86/entry/entry_64.S`，源码对照）：

```asm
SYM_CODE_START(entry_SYSCALL_64)
    swapgs                                       ; ① 切到内核 GS（per-CPU 基准）
    movq %rsp, PER_CPU_VAR(... TSS_sp2)          ;   暂存用户栈指针
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp        ;   切内核页表（KPTI）
    movq PER_CPU_VAR(pcpu_hot + X86_top_of_stack), %rsp  ; ② 换到内核栈

    /* ③ 在内核栈上手搭 struct pt_regs */
    pushq $__USER_DS        /* ss    */
    pushq PER_CPU_VAR(... TSS_sp2)  /* sp = 用户栈 */
    pushq %r11              /* flags（syscall 存的） */
    pushq $__USER_CS        /* cs    */
    pushq %rcx              /* ip（syscall 存的返回地址） */
    pushq %rax              /* orig_ax = 系统调用号 */
    PUSH_AND_CLEAR_REGS rax=$-ENOSYS  ; rdi/rsi/rdx/r10/... 全压栈并清零

    movq %rsp, %rdi         ; ④ rdi = pt_regs*
    movslq %eax, %rsi       ;   rsi = 系统调用号
    call do_syscall_64      ; ⑤ 跳进 C，从此是 C 的世界
    // —— do_syscall_64 内部（C）——
    //     nr = regs->orig_ax;
    //     → sys_call_table[nr] → __x64_sys_write
    //     → regs->ax = 返回值
    
    ...
syscall_return_via_sysret:  ; —— 返回路径，进入动作的逆操作 ——
    POP_REGS pop_rdi=0      ; 恢复通用寄存器
    movq RSP-RDI(%rdi), %rsp  ; 恢复用户栈指针
    ...
    swapgs                  ; GS 换回用户态
    sysretq                 ; 返回用户态
SYM_CODE_END(entry_SYSCALL_64)
```

### 8.3 逐项对照表

| 阶段 | ARM64（本机实测） | x86-64（源码对照） |
|------|------------------|-------------------|
| 入口符号 | `el0t_64_sync` → `el0_svc` | `entry_SYSCALL_64` |
| 怎么进来的 | `svc` 触发异常，过向量表 | `syscall` 直跳 `IA32_LSTAR` |
| 切内核上下文指针 | `msr sp_el0, tsk` | `swapgs` |
| 切内核栈 | 硬件自动（SP_EL1） | `movq ...top_of_stack, %rsp`（手动） |
| 建 pt_regs | `stp x0..x29` 连续存 | 手搭硬件帧 + `PUSH_AND_CLEAR_REGS` |
| 系统调用号在哪 | `regs[8]`（来自 x8） | `orig_ax`（来自 rax） |
| 传给 C | `x0 = pt_regs*` | `movq %rsp, %rdi` |
| 进 C 派发 | `do_el0_svc(regs)` | `do_syscall_64(regs, nr)` |
| 写回返回值 | `syscall_set_return_value` → `regs[0]` | → `pt_regs->ax` |
| 返回用户态 | `eret` | `sysretq` |

一句话总结这张表：**ARM64 因为走异常机制，硬件帮它切了栈、但多了一层向量表；x86-64 的 `syscall` 更"轻"，少了向量表、但栈和 GS 都得软件自己切。各有各的活，但"建 pt_regs → 跳 C"这个核心骨架两边完全一样。**

## 九、回到开头：内核入口地址上放的是什么

现在可以完整回答开头的问题了。`svc` / `syscall` 把 CPU 送到内核入口地址，那个地址上放的，是一段**承上启下的汇编**——它在 C 代码能跑之前，亲手把现场搭好：

```text
┌──────────────────────────────────────────────────────────────┐
│ 用户态：write(1, "hello\n", 6)                                 │
│   x8=64(nr)  x0=1(fd)  x1=buf  x2=6(count)                     │
│   svc #0  ───────────────┐                                     │
└──────────────────────────┼────────────────────────────────────┘
                           ▼ CPU 切 EL0→EL1，跳 VBAR_EL1
┌──────────────────────────────────────────────────────────────┐
│ 汇编入口 el0t_64_sync / el0_svc                                │
│  ① msr sp_el0, tsk        切到内核 current（≈ x86 swapgs）     │
│  ② stp x0..x29, [sp]      把 31 个寄存器存成 struct pt_regs    │
│       └─ x8 → regs[8]（系统调用号就位）                        │
│  ③ x0 = &pt_regs；调 C    把现场指针交出去                     │
└──────────────────────────┬────────────────────────────────────┘
                           ▼  call（汇编→C 的分界线）
┌──────────────────────────────────────────────────────────────┐
│ C 派发 do_el0_svc(regs)                                        │
│   nr = regs->regs[8];     // = 64，正是我们 bpftrace 读到的    │
│   → el0_svc_common → invoke_syscall                           │
│       → sys_call_table[64] → __arm64_sys_write（下一篇）       │
│   syscall_set_return_value(regs, ret);  // 写回 regs[0]       │
└──────────────────────────┬────────────────────────────────────┘
                           ▼ 返回路径（进入的镜像）
┌──────────────────────────────────────────────────────────────┐
│ 汇编出口 kernel_exit                                           │
│   ldp ... 从 pt_regs 恢复寄存器                                │
│   eret  ───────────────┐  恢复 PC/PSTATE，切 EL1→EL0           │
└────────────────────────┼───────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ 用户态继续，x0 = 6（返回值）                                   │
└──────────────────────────────────────────────────────────────┘
```

我们用真机数据钉死了这条链上最关键的几个节点：

1. `/proc/kallsyms` 证明了入口符号 `el0t_64_sync` / `el0_svc` / `do_el0_svc` 真实存在。
2. bpftrace 抓的内核栈证明了 `el0t_64_sync → el0_svc → do_el0_svc → __arm64_sys_write` 是一条真实的调用链。
3. 按 `pt_regs` 偏移读出的 `x8(nr)=64 / x0(fd)=1 / x2(count)=6`，证明了**汇编用 `stp` 存进去的寄存器，正是 C 用 `regs->regs[8]` 读出来的系统调用号**。
4. 返回时 `regs[0]` 从参数 `1` 变成返回值 `6`，证明了**参数和返回值共用同一个寄存器槽位**，由 `syscall_set_return_value` 写回。

至于内核拿到 `nr=64` 之后，怎么在 `sys_call_table` 里查到 `__arm64_sys_write`、再走到 `ksys_write`——那是本系列最后一篇《[从 write 到 ksys_write——sys_call_table 怎么路由的](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/四、从write到ksys_write——sys_call_table怎么路由的.md)》的事了。这一篇铺好的 `struct pt_regs`，正是那一篇查表派发的输入。

## 十、参考资料

- Linux 内核源码：https://github.com/torvalds/linux （本文片段基于 v6.12）
- ARM64 入口汇编：`arch/arm64/kernel/entry.S`（`kernel_entry` / `kernel_exit` 宏、向量表）
- ARM64 系统调用派发：`arch/arm64/kernel/syscall.c`（`do_el0_svc` / `el0_svc_common` / `invoke_syscall`）
- ARM64 异常分流：`arch/arm64/kernel/entry-common.c`（`el0t_64_sync_handler`）
- x86-64 入口汇编：`arch/x86/entry/entry_64.S`（`entry_SYSCALL_64` / `syscall_return_via_sysret`）
- x86-64 系统调用派发：`arch/x86/entry/common.c`（`do_syscall_64`）
- `struct pt_regs` 定义：`arch/arm64/include/asm/ptrace.h` / `arch/x86/include/asm/ptrace.h`

---

上一篇：[`svc` / `syscall` 指令到底做了什么——从 ring 3 到 ring 0 的硬件门](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/二、syscall指令做了什么——从ring3到ring0的硬件门.md)
下一篇：[从 write 到 ksys_write——sys_call_table 怎么路由的](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/四、从write到ksys_write——sys_call_table怎么路由的.md)
