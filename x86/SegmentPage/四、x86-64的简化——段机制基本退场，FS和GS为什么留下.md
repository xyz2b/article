# 四、x86-64 的简化：段机制基本退场，FS/GS 为什么留下

---

> **系列说明**：这是"x86 的内存管理是怎么一步步演进来的"系列第四篇。整个系列想把三样东西从 **CPU 硬件的视角**讲透：**段寄存器机制**、**GDT / TSS 这些 CPU 层面的表和寄存器**、**页表的硬件翻译机制**。主线是一条演进史——从 1978 年的 8086，到 80386 的保护模式，再到今天的 x86-64。
>
> 六篇的安排是：[第一篇 8086 实模式的段寄存器（为什么会有"段"这东西）](<一、8086的段寄存器——16位寄存器怎么够到1MB内存.md>)；[第二篇 保护模式的段（选择子、GDT、段描述符的位结构）](<二、保护模式的段——选择子、GDT与64位段描述符.md>)；[第三篇 特权级与门，以及 TSS 当年的"本来用途"](<三、特权级与门，以及TSS的本来用途.md>)；**第四篇（本文）** x86-64 的简化（段基本废弃，FS/GS 为什么留下）；[第五篇 内核里的 GS / swapgs 与现代 TSS](<五、内核里的GS和swapgs，与现代TSS.md>)；[第六篇 页表的 CPU 机制（CR3、page walk、PTE、KPTI）](<六、页表的CPU机制——CR3、page walk、PTE、TLB与KPTI.md>)。

---

前三篇我们一路看下来，段机制的身份换了好几次。

在 8086 实模式里，段寄存器是个**凑高位的补丁**：`段 << 4 + 偏移` 拼出 20 位物理地址。到了 80386 保护模式，段寄存器变成**选择子**，CPU 拿它查 GDT/LDT，描述符里有 base、limit、type、DPL，段机制一下子长成了一套能做边界检查、权限检查、受控跨级和硬件任务切换的复杂系统。

但操作系统后来没有沿着 Intel 早期设想的"分段操作系统模型"继续走下去。它们很快发现：**内存隔离更适合交给分页，任务切换更适合交给软件，段最好被摊平成透明背景。**

所以在 32 位保护模式里，现代系统通常已经把段做成平坦模型：

```
   代码段 base = 0, limit = 4GB
   数据段 base = 0, limit = 4GB

   线性地址 = 段基址 + 偏移 = 0 + 偏移 = 偏移
```

段还在，但它几乎不改变地址。真正决定"这块虚拟地址能不能访问、映射到哪个物理页"的，是页表。

到了 x86-64，CPU 干脆把这个事实写进架构：**在 64 位长模式下，CS/DS/ES/SS 的 base 和 limit 基本被硬件忽略，普通内存访问不再靠分段做地址划分。**

这里先解释一下"64 位长模式"这个词。**长模式（long mode）** 是 x86-64 CPU 用来运行 64 位体系的模式；它不是"程序很长"的意思，而是相对于早期的实模式、保护模式来说，进入了 x86-64 扩展后的运行状态。严格说，长模式里还分 **64-bit mode** 和 **compatibility mode**：前者是真正执行 64 位代码，后者用来兼容执行 32 位/16 位保护模式程序。本文后面说"长模式"时，默认指正在执行 64 位代码的那条路径，也就是 CS 描述符的 L 位为 1 的 64-bit mode。

但奇怪的是，六个段寄存器里有两个没被一起埋掉：**FS 和 GS**。

它们的选择子仍然可以存在，但更重要的是：x86-64 给 FS/GS 的 base 开了单独通道。FS base 和 GS base 不再主要从普通段描述符里来，而是可以从专门的 MSR 或指令设置。用户态线程本地存储（TLS）常用 FS；内核 per-CPU 数据常用 GS。也就是说，段机制作为"内存分段"退场了，但留下了两个**带隐藏基址的快速指针寄存器**。

这一篇就讲这次简化：长模式到底忽略了段描述符的哪些字段，FS/GS 为什么例外，`__thread` 变量为什么会编译成 `%fs:偏移`，以及这件事和第五篇的 `swapgs`、现代 TSS 怎么接上。

## 一、长模式先把地址空间拉大：寄存器够用了，段没必要再凑高位

8086 需要段，是因为 16 位寄存器够不着 1MB。80386 继续强化段，是因为它想让 CPU 靠描述符做保护。

x86-64 面对的问题已经完全不同。

通用寄存器变成 64 位，`RAX/RBX/RCX/.../RIP/RSP` 都能放很大的地址。更准确地说，当前主流 x86-64 并不是把 64 位虚拟地址全用满，而是使用**规范地址（canonical address）**：常见实现是 48 位虚拟地址，开启 LA57 后可以到 57 位。

以最常见的 48 位虚拟地址为例：

```
   64 位寄存器里的虚拟地址（48 位 canonical）

   bit 63                         48 47                         0
   ┌────────────────────────────────┬────────────────────────────┐
   │  bit47 的符号扩展               │       有效虚拟地址位        │
   └────────────────────────────────┴────────────────────────────┘

   如果 bit47 = 0，高 16 位必须全是 0
   如果 bit47 = 1，高 16 位必须全是 1
```

这个地址空间已经远远超过 32 位时代的 4GB，更不用说 8086 的 1MB。第一篇那个"寄存器位数不够"的矛盾，在 x86-64 这里彻底消失了。

同时，保护模式里段的另一个主要用途——边界和权限——也被分页接走了。页表可以以 4KB 页为粒度控制：

```
   虚拟页是否存在：P 位
   能不能写：R/W 位
   用户态能不能访问：U/S 位
   能不能执行：NX 位
   映射到哪个物理页框：PTE 里的物理地址位
```

页表的粒度更细，和按需分配、写时复制、文件映射、共享库、内存回收这些现代 OS 机制配合得也更自然。一个段动辄覆盖一大片连续线性地址，不适合表达现代进程地址空间里那些零散的 VMA。

所以 x86-64 的选择很现实：

```
   地址空间够大了，不需要段来凑高位。
   保护和隔离交给页表，不需要普通段做边界。
   于是普通分段被简化成近似透明。
```

这就是长模式下段机制退场的背景。

## 二、长模式下，CS/DS/ES/SS 的 base 和 limit 被忽略

先把最核心的硬件规则摆出来。

在 64 位模式下，对普通内存寻址来说：

```
   CS/DS/ES/SS 的 base 被当成 0
   CS/DS/ES/SS 的 limit 检查不再用于限制地址范围

   有效地址 = 指令算出来的 offset
   线性地址 = offset
```

也就是说，下面这些 32 位保护模式里很重要的字段，到了 64 位代码里基本不再参与普通地址生成：

```
   描述符里的 base
   描述符里的 limit
   G 粒度位对 limit 的放大效果
   向上/向下扩展数据段的边界意义
```

举个对比：

```
   32 位保护模式：

     mov eax, [0x12345678]

     如果默认段是 DS：
       线性地址 = DS.hidden_base + 0x12345678
       同时检查 0x12345678 是否在 DS.limit 内


   64 位长模式：

     mov eax, [0x12345678]

     默认段仍然可以说是 DS，但：
       DS.base 被硬件当成 0
       DS.limit 不限制这次访问
       线性地址 = 0x12345678
       后面交给分页检查
```

这就是"平坦模型"从操作系统习惯变成架构规则。32 位时代，内核还得认真把 GDT 里的代码段/数据段做成 `base=0, limit=4GB`，才能让段透明；64 位时代，对 CS/DS/ES/SS 来说，即使描述符里写了别的 base/limit，硬件也不拿它们做普通地址计算。

但这不等于段描述符完全没用了。长模式下仍然保留了一些和"地址划分"无关的字段：

```
   CS 选择子仍然决定 CPL：
     CPL = CS.selector & 3

   代码段描述符里的 L 位仍然重要：
     L=1 表示 64 位代码段

   DPL 仍然参与特权级判断：
     用户态还是 Ring 3，内核态还是 Ring 0

   P/type 等合法性检查仍然存在：
     你不能加载一个不存在或类型不对的段
```

所以准确说法不是"x86-64 没有段"。更准确的是：

```
   x86-64 仍然有段寄存器、选择子、GDT、段描述符。
   但在 64 位模式下，普通代码/数据段不再用 base/limit 做地址划分。
```

段从"地址翻译主角"退成了"保存特权级和少数模式位的外壳"。

## 三、六个段寄存器，谁死谁活

把六个段寄存器放到同一张图里，长模式下的状态大致是这样：

```
   x86-64 64 位模式下的六个段寄存器

   ┌────────┬───────────────────────┬──────────────────────────────┐
   │ 寄存器 │ base/limit 对寻址       │ 还剩什么用处                 │
   ├────────┼───────────────────────┼──────────────────────────────┤
   │ CS     │ base=0, limit 忽略      │ CPL、L 位、代码段属性         │
   │ SS     │ base=0, limit 忽略      │ 栈段选择子、特权级相关检查    │
   │ DS     │ base=0, limit 忽略      │ 基本成了兼容外壳             │
   │ ES     │ base=0, limit 忽略      │ 基本成了兼容外壳             │
   │ FS     │ base 保留，limit 忽略   │ 用户态 TLS 常用               │
   │ GS     │ base 保留，limit 忽略   │ 内核 per-CPU 常用             │
   └────────┴───────────────────────┴──────────────────────────────┘
```

这张表就是第四篇的核心结论。

CS 不能死，因为 CPU 还要知道当前代码是不是 64 位代码、当前 CPL 是多少。SS 也不能完全死，因为一些栈相关、返回相关、特权级相关的规则还依赖它。DS/ES 在 64 位用户程序里基本存在感很低，大部分时间只是为了兼容指令编码和架构状态。

真正特殊的是 FS/GS。

在 32 位保护模式里，FS/GS 和 DS/ES 一样，本质上也是普通数据段寄存器。它们的 base 通常来自 GDT/LDT 描述符。到了 x86-64，普通段 base 被忽略，但 Intel/AMD 特意给 FS/GS 留了例外：

```
   线性地址 = FS.base + offset     （使用 FS 段覆盖时）
   线性地址 = GS.base + offset     （使用 GS 段覆盖时）
```

于是你会在反汇编里看到这种指令：

```asm
mov    eax, DWORD PTR fs:0x28
mov    rax, QWORD PTR gs:0x0
```

这里的 `fs:`、`gs:` 不是什么注释。它们是真的告诉 CPU：这次地址计算要加上 FS.base 或 GS.base。

这里顺手把"指令里的地址计算"也补一下。x86 的内存操作数经常长这样：

```asm
mov eax, [rbx + rcx*4 + 0x20]
```

方括号里的部分不是线性地址的最终来源，而是 CPU 先算出来的**有效地址（effective address）**，也就是段内偏移。它的通用形式可以写成：

```
   有效地址 offset = base_reg + index_reg * scale + displacement
```

对应到上面那条指令：

```
   base_reg      = rbx
   index_reg     = rcx
   scale         = 4
   displacement  = 0x20

   offset = rbx + rcx * 4 + 0x20
```

假设当时：

```
   rbx = 0x10000000
   rcx = 3
```

那 CPU 先算：

```
   offset = 0x10000000 + 3 * 4 + 0x20
          = 0x1000002c
```

在普通寻址里，长模式把 DS/ES/SS 的 base 当成 0，所以线性地址就是这个 offset：

```
   线性地址 = 0 + 0x1000002c
            = 0x1000002c
```

如果指令带了 FS 覆盖，比如：

```asm
mov eax, fs:[rbx + rcx*4 + 0x20]
```

前半段地址计算完全一样，CPU 仍然先得到 `offset = 0x1000002c`。不同的是后面要再加 FS.base：

```
   线性地址 = FS.base + offset
```

如果：

```
   FS.base = 0x00007f0000000000
```

那么这次真正交给分页的线性地址就是：

```
   0x00007f0000000000 + 0x1000002c
 = 0x00007f001000002c
```

所以，`base_reg + index_reg * scale + displacement` 只是在算"这次访问离段基址多远"；普通段的段基址被压成 0，FS/GS 的段基址还活着。

流程可以画成这样：

```
   普通寻址：

     指令里的地址计算
       base_reg + index_reg * scale + displacement
                 │
                 ▼
            有效地址 offset
                 │
                 ▼
            线性地址 offset
                 │
                 ▼
              交给分页


   FS/GS 寻址：

     指令里的地址计算
       base_reg + index_reg * scale + displacement
                 │
                 ▼
            有效地址 offset
                 │
                 ▼
       FS.base 或 GS.base + offset
                 │
                 ▼
              线性地址
                 │
                 ▼
              交给分页
```

注意最后一步仍然是分页。FS/GS base 只是给线性地址加了一个基址，不能绕过页表。最终这个线性地址能不能访问、能不能写、用户态能不能碰、能不能执行，还是由页表和当前特权级决定。

## 四、FS/GS base 从哪里来：MSR 和 FSGSBASE

第二篇讲过，保护模式的段基址通常来自 GDT/LDT 描述符。加载段寄存器时，CPU 查描述符，把 base/limit/权限缓存进段寄存器的隐藏部分。

x86-64 对 FS/GS 做了一层特别处理：FS.base 和 GS.base 可以来自专门的 **MSR（Model Specific Register）**。

几个关键 MSR 是：

```
   IA32_FS_BASE         = 0xC0000100
   IA32_GS_BASE         = 0xC0000101
   IA32_KERNEL_GS_BASE  = 0xC0000102
```

其中：

- **IA32_FS_BASE**：保存 FS base。用户态 TLS 常用它。
- **IA32_GS_BASE**：保存当前 GS base。
- **IA32_KERNEL_GS_BASE**：给 `swapgs` 准备的备用 GS base，第五篇重点讲。

MSR 只能由内核用 `rdmsr/wrmsr` 这类特权指令直接读写。用户态不能自己随便写 MSR，否则它就能把 FS/GS base 指到不该碰的地方，至少会破坏内核对线程状态的管理。

但用户程序确实需要设置 TLS base。Linux 给用户态提供了系统调用接口：`arch_prctl`。

常见用法是：

```c
arch_prctl(ARCH_SET_FS, fs_base);
arch_prctl(ARCH_GET_FS, &fs_base);
```

在 x86-64 Linux 上，线程库创建线程时，会给每个线程准备自己的 TLS/TCB 区域，然后让内核把这个线程的 FS base 设置过去。之后用户态代码访问线程本地变量，就可以靠 `%fs:偏移` 快速完成，不需要每次系统调用。

后来 x86 又加了 FSGSBASE 指令，让软件能直接读写 FS/GS base：

```
   RDFSBASE / WRFSBASE
   RDGSBASE / WRGSBASE
```

这些指令是否允许用户态执行，由控制寄存器里的开关和内核策略决定。现代 Linux 在合适的 CPU 和配置下可以开放它们，这样线程库切换 FS base 或读取 FS base 就不一定非要走系统调用。但从机制上讲，重点不变：

```
   x86-64 把 FS/GS base 提升成了特殊 CPU 状态。
   它不再只是普通段描述符里的 32 位 base。
```

这一步非常关键。64 位地址空间里，FS/GS base 本身需要是 64 位；而老式段描述符的 base 只有 32 位。靠 MSR 单独保存，才能让 FS/GS 指向完整的 64 位虚拟地址。

这里容易产生一个误会：既然已经是 64 位长模式了，FS/GS 的 base 不就天然是 64 位吗？不是。长模式没有把老式段描述符扩展成"64 位 base 描述符"。老式描述符格式还在，里面的 base 字段仍然只有 32 位；只是 FS/GS 被架构单独开了后门，真正用于地址计算的 base 从专门的 MSR 里取。

先看 32 位保护模式下的老式结构：

```
   FS / GS 段寄存器

   +------------------+--------------------------------------+
   | 可见部分          | CPU 内部隐藏缓存                      |
   +------------------+--------------------------------------+
   | selector: 16 bit  | base: 32 bit                         |
   |                  | limit: 20 bit，经粒度扩展后可覆盖 4GB |
   |                  | attributes: type / DPL / P / D/B / G |
   +------------------+--------------------------------------+
```

这里的 `selector` 指向 GDT/LDT 里的段描述符：

```
   FS selector
        │
        ▼
   GDT / LDT 段描述符（老式格式，64 bit）

   +---------------------------------------------------------------+
   | base[31:24] | flags | limit[19:16] | attr | base[23:16] | ... |
   +---------------------------------------------------------------+
   | base[15:0]  | limit[15:0]                                    |
   +---------------------------------------------------------------+

   拼出来：

     base  = base[15:0] + base[23:16] + base[31:24]  => 32 bit
     limit = limit[15:0] + limit[19:16]              => 20 bit
```

所以 32 位保护模式里的 FS/GS 路径是：

```
   FS/GS selector
        │
        ▼
   GDT/LDT descriptor
        │
        ├── base: 32 bit
        ├── limit
        └── attributes
        │
        ▼
   线性地址 = base32 + offset32
```

到了 64 位长模式，外面看起来还是 FS/GS 段寄存器，`selector` 也还是 16 位，但真正用于 FS/GS 寻址的 base 换了来源：

```
   FS / GS 段寄存器（长模式）

   +------------------+------------------------------------------+
   | 可见部分          | 架构状态 / 隐藏状态                      |
   +------------------+------------------------------------------+
   | selector: 16 bit  | base: 64 bit                             |
   |                  | limit: 基本不用于普通地址范围检查          |
   |                  | attributes: 仍保留一部分类型/权限相关信息  |
   +------------------+------------------------------------------+
```

这个 64 位 base 不是从老式段描述符拼出来的，而是从 MSR 取：

```
   IA32_FS_BASE MSR
   +---------------------------------------------------------------+
   | FS_BASE[63:0]                                                 |
   +---------------------------------------------------------------+

   IA32_GS_BASE MSR
   +---------------------------------------------------------------+
   | GS_BASE[63:0]                                                 |
   +---------------------------------------------------------------+
```

长模式下的 FS/GS 路径可以画成这样：

```
   FS/GS selector
        │
        ▼
   GDT/LDT descriptor
        │
        └── 如果装载的是非空 selector，type、DPL、P 等信息仍会参与检查

   IA32_FS_BASE / IA32_GS_BASE MSR
        │
        └── base: 64 bit
        │
        ▼
   线性地址 = base64 + offset64
```

再把两边压成一张对照表：

```
   32 位保护模式：

     FS/GS selector -> GDT/LDT descriptor -> base32
     地址 = base32 + offset32

   64 位长模式：

     FS/GS selector -> 非空时仍可查 GDT/LDT descriptor 做权限/属性检查
     IA32_FS_BASE / IA32_GS_BASE MSR -> base64
     地址 = base64 + offset64
```

这里说的 MSR 保存 64 位，是指这些 MSR 通过 `rdmsr/wrmsr` 读写的是 64 位值，读出来放在 `EDX:EAX` 这对 32 位寄存器里。只是"保存 64 位"不等于任意 64 位数字都是合法虚拟地址；真实 CPU 通常要求 canonical address，常见有效虚拟地址宽度是 48 位，新一些机器也可能是 57 位。

一句话记住：**长模式支持 64 位 FS/GS base，但不是因为老式段描述符变成 64 位了，而是因为 FS/GS 的 base 被单独放进了 64 位 MSR。**

## 五、为什么偏偏留下 FS/GS：它们是便宜的"当前线程指针"

问题来了：既然普通段都该退场，为什么还要留下 FS/GS？

因为操作系统和运行时有一个非常高频的需求：**快速找到"当前执行上下文"的一小块私有数据。**

对用户态来说，这块数据就是线程本地存储，也就是 TLS。每个线程都有自己的一份：

```
   线程 A:
     errno
     __thread 变量
     pthread 自己的线程控制块 TCB
     栈保护 canary 等运行时数据

   线程 B:
     errno
     __thread 变量
     pthread 自己的线程控制块 TCB
     栈保护 canary 等运行时数据
```

同一个全局符号，比如 `errno`，在线程 A 和线程 B 里不能指向同一份内存。它必须"看起来像全局变量"，但实际每个线程一份。

最直接的办法是：每个线程有一个 TLS 基址，线程本地变量都按固定偏移放在这个基址附近。

```
   线程 A 的 FS.base ─────┐
                          ▼
                    ┌──────────────┐
                    │ TCB          │
                    ├──────────────┤
                    │ errno        │  offset -0x10
                    ├──────────────┤
                    │ tls_counter  │  offset -0x14
                    ├──────────────┤
                    │ 其他 TLS      │
                    └──────────────┘

   线程 B 的 FS.base ─────┐
                          ▼
                    ┌──────────────┐
                    │ TCB          │
                    ├──────────────┤
                    │ errno        │  offset -0x10
                    ├──────────────┤
                    │ tls_counter  │  offset -0x14
                    ├──────────────┤
                    │ 其他 TLS      │
                    └──────────────┘
```

代码里访问 `tls_counter` 时，编译器不需要知道当前是哪一个线程。它只要生成：

```asm
mov eax, DWORD PTR fs:固定偏移
```

当前线程是谁，由 FS.base 决定。线程 A 运行时 FS.base 指向 A 的 TLS，线程 B 运行时 FS.base 指向 B 的 TLS。同一条机器指令，在不同线程里自动访问不同内存。

这很便宜：

- 不需要从某个全局变量里先读"当前线程指针"。
- 不需要函数调用。
- 不需要系统调用。
- 不需要额外占用一个通用寄存器。
- 地址计算由 CPU 原生支持，和普通 load/store 在同一条执行路径上完成。

这就是 FS/GS 被留下来的现实原因。它们不再是"分段内存模型"的主角，而是变成了两个专用的、每 CPU/每线程都能快速切换的**基址寄存器**。

## 六、`__thread` 怎么变成 `%fs:偏移`

看一个最小例子：

```c
// tls.c
#define _GNU_SOURCE

#include <asm/prctl.h>
#include <pthread.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

__thread int tls_counter;

static unsigned long get_fs_base(void)
{
    unsigned long fs;

    syscall(SYS_arch_prctl, ARCH_GET_FS, &fs);
    return fs;
}

int inc_tls(void)
{
    tls_counter++;
    return tls_counter;
}

static void *thread_main(void *arg)
{
    long id = (long)arg;
    const char *name = id == 1 ? "thread A" : id == 2 ? "thread B" : "main";
    unsigned long fs = get_fs_base();
    void *addr = &tls_counter;
    intptr_t offset = (intptr_t)addr - (intptr_t)fs;

    tls_counter = (int)(id * 1000);

    printf("%s: fs.base=0x%lx tls_counter=%p offset=%+ld value=%d\n",
           name, fs, addr, (long)offset, inc_tls());
    printf("%s: fs.base=0x%lx tls_counter=%p offset=%+ld value=%d\n",
           name, fs, addr, (long)offset, inc_tls());

    return NULL;
}

int main(void)
{
    pthread_t t1;
    pthread_t t2;

    pthread_create(&t1, NULL, thread_main, (void *)1);
    pthread_create(&t2, NULL, thread_main, (void *)2);

    thread_main((void *)3);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    return 0;
}
```

在 x86-64 Linux 上，如果用比较简单的本地执行模型编译：

```bash
gcc -O2 -pthread tls.c -o tls
./tls
```

我在 `linux/amd64` 的 `gcc:13` 容器里实际跑到一组输出：

```text
main: fs.base=0x7fffff7c2640 tls_counter=0x7fffff7c263c offset=-4 value=3001
thread B: fs.base=0x7ffffeddb6c0 tls_counter=0x7ffffeddb6bc offset=-4 value=2001
main: fs.base=0x7fffff7c2640 tls_counter=0x7fffff7c263c offset=-4 value=3002
thread B: fs.base=0x7ffffeddb6c0 tls_counter=0x7ffffeddb6bc offset=-4 value=2002
thread A: fs.base=0x7fffff5dc6c0 tls_counter=0x7fffff5dc6bc offset=-4 value=1001
thread A: fs.base=0x7fffff5dc6c0 tls_counter=0x7fffff5dc6bc offset=-4 value=1002
```

具体地址每次运行都会变，输出顺序也可能因为线程调度而变化。重点看三列：三个线程的 `offset` 都是 `-4`，因为 `tls_counter` 在 TLS 布局里的位置是固定的；`fs.base` 按线程变化；`value` 也各自从 1000、2000、3000 往上加，互不影响。

TLS 的关键不是变量名有魔法，而是这一条公式：

```
   tls_counter 的地址 = 当前线程 FS.base + tls_counter 的固定 offset
```

如果只看 `inc_tls()` 这一小段生成的汇编，可以再编译成 `.s`：

```bash
gcc -O2 -S -fno-pic tls.c -o tls.s
```

你通常会看到类似这样的汇编：

```asm
inc_tls:
    movl    %fs:tls_counter@tpoff, %eax
    addl    $1, %eax
    movl    %eax, %fs:tls_counter@tpoff
    ret
```

不同 GCC/Clang 版本、是否 PIE、是否动态链接、TLS 模型不同，具体写法会变。动态 TLS 场景可能会出现 `__tls_get_addr` 调用。但核心不变：

```
   TLS 变量地址 = 当前线程 TLS base + 这个变量的固定偏移
```

在 x86-64 Linux 用户态，"当前线程 TLS base"通常就是 FS.base。

所以 `__thread` 的表面语义是：

```c
每个线程都有自己的 tls_counter
```

落到机器层面就是：

```asm
fs:某个偏移
```

这个偏移可能是负数，也可能通过重定位表达。很多 ABI 会把线程控制块放在 FS.base 指向的位置，TLS 变量放在它附近。于是一个 TLS 访问可以画成这样：

```
   C 代码：

     tls_counter++

          │
          ▼
   编译器生成：

     mov/add/mov  %fs:offset

          │
          ▼
   CPU 地址生成：

     线性地址 = FS.base + offset

          │
          ▼
   页表翻译：

     线性地址 -> 物理地址
```

这也解释了为什么普通程序崩溃时，反汇编里经常能看到 `%fs:0x28`、`%fs:0x10` 之类的访问。那通常不是在访问什么神秘硬件，而是在访问当前线程的运行时数据。比如栈保护 canary、线程控制块字段、`errno` 背后的线程局部区域，都可能经由 FS 访问。

## 七、一个用户态实验：读出 FS base，再用 `%fs:0` 访问它

下面这个小实验可以在 x86-64 Linux 上跑。它做三件事：

1. 用 `arch_prctl(ARCH_GET_FS)` 读当前线程的 FS base。
2. 用内联汇编执行 `movq %fs:0, reg`。
3. 打印两者，看 `FS.base+0` 这个地址里放了什么。

```c
// fsbase.c
#define _GNU_SOURCE
#include <asm/prctl.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

static unsigned long get_fs_base(void)
{
    unsigned long fs;
    if (syscall(SYS_arch_prctl, ARCH_GET_FS, &fs) != 0) {
        return 0;
    }
    return fs;
}

int main(void)
{
    unsigned long fs = get_fs_base();
    unsigned long fs0;

    asm volatile("movq %%fs:0, %0" : "=r"(fs0));

    printf("FS.base      = 0x%016lx\n", fs);
    printf("*(FS.base+0) = 0x%016lx\n", fs0);
    return 0;
}
```

编译运行：

```bash
gcc -O2 fsbase.c -o fsbase
./fsbase
```

在 glibc 的常见布局下，你很可能看到两行相等或高度相关：

```text
FS.base      = 0x00007f2e3b7fc740
*(FS.base+0) = 0x00007f2e3b7fc740
```

这里分两层看就清楚了：

```
   movq %fs:0, reg

   访问地址 = FS.base + 0
   读出的值 = 内存[FS.base + 0]
```

在 glibc 的常见 TLS/TCB 布局里，可以直接这么理解：

```
   FS.base 指向当前线程的 TCB
   TCB 起始位置放着一个指向自己的地址

   所以：
   内存[FS.base + 0] = FS.base
```

所以两行相等，不是 CPU 规定 `%fs:0` 的值必须等于 `FS.base`；而是运行时把 TCB 的第一个字段设置成了 TCB 自己的地址。CPU 只负责按 `FS.base + offset` 去读内存。

如果你想直接看编译器生成的 TLS 访问，可以再写一个：

```c
// tls_asm.c
#include <stdio.h>

__thread int x;

int f(void)
{
    return x + 1;
}

int main(void)
{
    x = 41;
    printf("f() = %d\n", f());
    return 0;
}
```

生成汇编：

```bash
gcc -O2 -S -fno-pic tls_asm.c -o tls_asm.s
grep -n "fs:" tls_asm.s
```

预期能看到类似：

```asm
movl    %fs:x@tpoff, %eax
```

这条指令就是本篇说的 FS base 机制在日常 C 代码里的落点。

## 八、FS 和 GS 的分工：用户 TLS 与内核 per-CPU

在 x86-64 Linux 的常见约定里：

```
   用户态：
     FS base -> 当前线程 TLS/TCB

   内核态：
     GS base -> 当前 CPU 的 per-CPU 数据区
```

为什么内核喜欢用 GS 指 per-CPU 数据？

先解释一下 **per-CPU**：它就是"每个 CPU 各有一份"的内核数据。不是每个线程一份，也不是整个系统共享一份，而是 CPU0 有 CPU0 的副本，CPU1 有 CPU1 的副本。当前代码跑在哪个 CPU 上，就访问哪个 CPU 自己的那份。

内核需要这种数据，是因为很多状态天然跟 CPU 绑定，而且访问频率很高。比如：

```
   当前 CPU 编号
   当前正在运行的 task 指针
   调度器运行队列的一部分
   内核栈相关信息
   中断/抢占计数
   每 CPU 统计数据
```

如果这些状态都放进一个全局共享结构，每次访问都要先算"我是几号 CPU"，再去数组里取，甚至还要考虑并发争用。per-CPU 的思路是：给每个 CPU 准备一块自己的数据区，内核代码写一条固定偏移访问，就能拿到当前 CPU 的那份。

于是 GS 很适合：

```asm
mov    rax, qword ptr gs:offset_of_current_task
```

同一条指令在 CPU0 上用 CPU0 的 GS.base，在 CPU1 上用 CPU1 的 GS.base，自动访问各自的 per-CPU 区域。

可以画成这样：

```
   CPU0 的 GS.base ─────┐
                        ▼
                  ┌──────────────┐
                  │ CPU0 per-CPU │
                  │ current      │
                  │ runqueue     │
                  │ stats        │
                  └──────────────┘

   CPU1 的 GS.base ─────┐
                        ▼
                  ┌──────────────┐
                  │ CPU1 per-CPU │
                  │ current      │
                  │ runqueue     │
                  │ stats        │
                  └──────────────┘
```

这和用户态 TLS 是同一种思想：

```
   用户态 TLS：
     当前线程是谁，由 FS.base 决定

   内核 per-CPU：
     当前 CPU 是谁，由 GS.base 决定
```

区别是：用户态线程切换时，内核/线程库要保证 FS.base 对应新线程；CPU 之间的 per-CPU 区域则通常由每个 CPU 自己持有不同的 GS base。第五篇会专门讲一个关键指令：`swapgs`。它负责在用户态 GS 和内核 GS 之间切换，让内核入口能安全拿到 per-CPU 数据。

## 九、长模式下，段退场后地址翻译变成什么样

把这一篇和前两篇的地址流程放在一起，对比会很清楚。

第一篇的实模式：

```
   段寄存器值 << 4
          +
        偏移
          │
          ▼
      20 位物理地址
```

第二篇的 32 位保护模式：

```
   段选择子
      │
      ▼
   查 GDT/LDT
      │
      ▼
   描述符 base + offset
      │
      ▼
   线性地址
      │
      ▼
   分页 -> 物理地址
```

第四篇的 x86-64 长模式：

```
   普通 CS/DS/ES/SS 访问：

     offset
       │
       ▼
     线性地址
       │
       ▼
     分页 -> 物理地址


   FS/GS 访问：

     FS.base / GS.base + offset
       │
       ▼
     线性地址
       │
       ▼
     分页 -> 物理地址
```

所以长模式不是把"段 + 页"里的页删掉了，而是把普通段地址计算压扁了：

```
   以前：
     逻辑地址 -> 分段 -> 线性地址 -> 分页 -> 物理地址

   x86-64 常规路径：
     虚拟地址/线性地址 -> 分页 -> 物理地址

   x86-64 FS/GS 路径：
     FS/GS base + offset -> 线性地址 -> 分页 -> 物理地址
```

从程序员视角看，"虚拟地址"这个词在 x86-64 上通常就指线性地址。分页才是虚拟地址到物理地址的核心翻译机制。这也是第六篇要讲 CR3、page walk、PTE、TLB 的原因：到了现代 x86-64，真正热的地址翻译路径已经主要在 MMU 和页表那里。

## 十、一个容易踩的误区：段没了，不等于 GDT 没了

很多资料会说"x86-64 不用分段了"。这句话作为直觉没问题，但容易让人误会成：GDT、选择子、段描述符都没了。

实际不是。

长模式启动本身仍然需要准备 GDT，至少要有一个 64 位代码段描述符。这个描述符里的 **L 位**必须为 1，CPU 才知道这是 64 位代码段。内核态和用户态仍然有不同的代码段选择子，CS 低 2 位仍然给出 CPL。

可以把长模式 GDT 粗略想成这样：

```
   x86-64 里仍然存在的 GDT（示意）

   GDT[0]  空描述符
   GDT[1]  内核 64 位代码段     L=1, DPL=0
   GDT[2]  内核数据段           DPL=0
   GDT[3]  用户数据段           DPL=3
   GDT[4]  用户 64 位代码段     L=1, DPL=3
   GDT[5+] TSS 描述符           下一篇讲 GS / swapgs 与现代 TSS 时展开
```

这里说的 **64 位代码段描述符**，不是一套全新的 64 位 base 描述符。它仍然是第二篇讲过的那种普通 8 字节代码/数据段描述符，只是 flags 里的 **L 位**打开了。

注意，这里说的是普通代码/数据段描述符。长模式下的某些**系统段描述符**确实变长了，比如 TSS 描述符和 LDT 描述符是 16 字节，因为它们还需要保存更宽的 base。下一篇主题是内核里的 GS / `swapgs` 与现代 TSS，所以会重点展开 TSS；LDT 只作为同类系统描述符顺带对比，不会作为主线。

普通代码段描述符大致还是这个老结构：

```
   代码段描述符（8 字节，示意）

   +---------------------------------------------------------------+
   | base[31:24] | G | D/B | L | AVL | limit[19:16] | access byte |
   +---------------------------------------------------------------+
   | base[23:16] | base[15:0] | limit[15:0]                       |
   +---------------------------------------------------------------+

   access byte 里仍然有：
     P    = present，描述符是否有效
     DPL  = 0/3 等特权级
     S    = 1，普通代码/数据段
     E    = 1，代码段
     R    = 代码段是否可读
```

和 32 位代码段相比，关键差别在 flags：

```
   32 位代码段：
     L   = 0
     D/B = 1       表示默认 32 位操作数/地址大小
     base/limit    参与普通分段寻址和范围检查

   64 位代码段：
     L   = 1       表示这是 64 位代码段
     D/B = 0       L=1 时 D/B 必须为 0
     base/limit    在 64 位代码的普通寻址里基本被忽略
```

所以一个 64 位代码段描述符真正重要的是：

```
   它是代码段：S=1, E=1
   它存在：P=1
   它的权限级：DPL=0 或 DPL=3
   它是 64 位代码：L=1
```

而不是：

```
   base 指向哪里
   limit 覆盖多大范围
```

但这里的代码段/数据段和 32 位时代的意义已经不一样了：

```
   还重要：
     L 位
     DPL
     P/type
     选择子低 2 位带来的 CPL/RPL

   不再重要于普通寻址：
     base
     limit
```

这也是为什么第二篇讲的描述符位结构没有白学。只是到了 x86-64，一部分字段退场，另一部分字段继续承担模式和权限的职责。

## 十一、收尾：段作为内存模型退场，作为快速上下文指针留下

这一篇的结论可以压成几句话：

- **x86-64 仍然有段寄存器和 GDT**。CS 还决定 CPL，代码段描述符的 L 位还决定 64 位代码，DPL/type/P 等检查仍然存在。
- **普通分段寻址基本退场**。在 64 位模式下，CS/DS/ES/SS 的 base 被当成 0，limit 不再限制普通地址范围；内存隔离主要交给分页。
- **FS/GS 是例外**。它们的 base 被保留下来，而且可以通过 `IA32_FS_BASE`、`IA32_GS_BASE`、`IA32_KERNEL_GS_BASE` 这些 MSR 或 FSGSBASE 指令管理。
- **用户态常用 FS 做 TLS**。`__thread`、`errno`、线程控制块、栈保护数据等，都可以通过 `%fs:偏移` 快速访问当前线程的数据。
- **内核常用 GS 做 per-CPU**。同一条 `gs:offset` 指令，在不同 CPU 上访问不同的 per-CPU 数据区。

所以，段机制并不是一下子消失了。它的主线更像这样：

```
   8086：
     段 = 凑 20 位物理地址的补丁

   80386：
     段 = base/limit/权限/GDT/TSS/门组成的保护模型

   32 位现代 OS：
     段 = 平坦模型，主要保护交给分页

   x86-64：
     普通段 = 模式和权限外壳
     FS/GS = TLS / per-CPU 的快速基址寄存器
```

下一篇会接着讲 GS 的内核用法：为什么内核入口要有 `swapgs`，`IA32_KERNEL_GS_BASE` 到底怎么和 GS base 对调，以及第三篇提到的 TSS 在长模式下又退化成了什么——不再保存整套任务现场，只剩 RSP0/RSP1/RSP2、IST 和 I/O 位图这些现代系统仍然离不开的字段。

## 附：复现实验命令

本文的用户态实验需要在 **x86-64 Linux** 上跑。它不依赖 root 权限。

**1）读取 FS base 并访问 `%fs:0`**

保存 `fsbase.c`：

```c
#define _GNU_SOURCE
#include <asm/prctl.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

static unsigned long get_fs_base(void)
{
    unsigned long fs;
    if (syscall(SYS_arch_prctl, ARCH_GET_FS, &fs) != 0) {
        return 0;
    }
    return fs;
}

int main(void)
{
    unsigned long fs = get_fs_base();
    unsigned long fs0;

    asm volatile("movq %%fs:0, %0" : "=r"(fs0));

    printf("FS.base      = 0x%016lx\n", fs);
    printf("*(FS.base+0) = 0x%016lx\n", fs0);
    return 0;
}
```

编译运行：

```bash
gcc -O2 fsbase.c -o fsbase
./fsbase
```

**2）看 `__thread` 生成的 FS 访问**

保存 `tls_asm.c`：

```c
#include <stdio.h>

__thread int x;

int f(void)
{
    return x + 1;
}

int main(void)
{
    x = 41;
    printf("f() = %d\n", f());
    return 0;
}
```

生成汇编并搜索 `fs:`：

```bash
gcc -O2 -S -fno-pic tls_asm.c -o tls_asm.s
grep -n "fs:" tls_asm.s
```

在常见 x86-64 Linux 工具链上，可以看到类似：

```asm
movl    %fs:x@tpoff, %eax
```

如果你的发行版默认 PIE、或者编译器选择了不同 TLS 模型，汇编可能变成 `__tls_get_addr` 路径。这不是机制变了，而是动态链接场景下 TLS 地址的计算方式更复杂；最终仍然要落到"当前线程 TLS base + 偏移"这个模型上。

---

上一篇：[特权级与门，以及 TSS 的本来用途](<三、特权级与门，以及TSS的本来用途.md>)

下一篇：[内核里的 GS / swapgs，与现代 TSS](<五、内核里的GS和swapgs，与现代TSS.md>)
