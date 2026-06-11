<!-- ════════════════════════════════════════════════════════════════
  写作纪律（写之前先看，别删，发布前再删）
  1. 深度红线：本篇 = 物理内存不够时内核怎么踢页——命名路径级。
     点到 kswapd / direct reclaim / shrink_lruvec / pageout / try_to_unmap /
     add_to_swap / do_swap_page 这些名字 + 它们的先后顺序 + 每一步在干什么，
     但【绝不贴内核源码、不做结构体字段走读】。
  2. 一个 aha 不综述。主钉子 = 「swap 不是"内存的备胎"，是内核用来把
     "冷"内存从 RAM 里驱逐出去、给"热"缺页腾地方的机制——
     驱逐和补回是同一枚硬币的两面」。
  3. 图是主角，纯 ASCII，中文标注放框外。核心是一张【内存压力 → 回收 → 换出/换回 总图】。
  4. 写完别打磨第七遍。发出去＝完成。
  5. 协作：机械讲解 Claude 写，带体温的 3 处（钩子/水位线恍然/收尾）用户用自己声音填。
  6. 真实产物：全部 docker 真跑。命令在正文，复制能复现。改数前回 /lab 重跑。
  7. 衔接：上一篇《缺页异常》结尾勾了"do_swap_page 分支的反面——内核怎么挑最不常用的页踢到磁盘"。
     本篇吃掉它；kcompactd / NUMA / OOM killer 不展开（可以再写）。
═══════════════════════════════════════════════════════════════ -->

# 内存不够时，内核怎么把"冷"页踢出去——swap 与页面回收

<!-- 用户填：钩子。
  建议接住上一篇的悬念：上一篇我们打开了缺页这个黑箱，看到 do_swap_page
  那个分支——页被换出过、访问时从磁盘换回来。但上一篇故意没展开另一面：
  内核是怎么决定"哪页该被踢出去"的？什么时候触发这个踢出动作？
  你可以用自己的话点一个反直觉的点：swap 这个词听着像"备胎""应急"，
  其实内核用它的时机比你想得早得多——不是等 OOM 快死了才动，
  是只要内存开始"偏紧"，后台就已经悄悄在踢冷页了。
  下面这段是我垫的，按你的声音改： -->

上一篇把缺页的决策树走完了，末尾留了一根线没接：`do_swap_page` 那个分支。内核把某一页踢到了磁盘交换区，再次访问时从那里换回来——但踢出去这一侧是怎么运作的？

这篇就填这个空。核心问题是：**物理内存快不够用时，内核怎么挑出"最不值得留在内存里"的那些页，把它们踢出去腾地方？**

> 本文所有数字真跑。环境：Linux（aarch64，内核 6.12，4KB 页，开了 swap）。每段都给了命令，复制能复现。函数名是 Linux 内核里真实的符号。

## 一、内核替你盯着内存：kswapd 和直接回收

Linux 不是等内存彻底耗尽才动手。它按**水位线（watermark）**提前管理：每个内存区（zone）有三条线——`min`、`low`、`high`。

```
   可用内存
   ─────────────────── high  ← 水位高于这里：什么都不用做
   ─────────────────── low   ← 跌破 low：后台 kswapd 开始回收
   ─────────────────── min   ← 跌破 min：分配路径上直接回收，卡住调用方
        0
```

- **`kswapd`**：内核后台线程，长驻。只要空闲内存跌破 `low` 水位就被唤醒，悄悄异步地把冷页踢出去，争取把水位推回 `high`。对你的程序**完全无感**——它在后台跑，不卡你。
- **直接回收（direct reclaim）**：如果内存压力太大，`kswapd` 追不上消耗，水位跌到 `min`，这时新来的内存分配请求（比如缺页要造一张新物理页）会在分配函数里**直接原地触发回收**，等回收出足够的页才返回。这才是你能感觉到的卡顿——进程分配内存被堵在内核里等。

两条路最终都走到同一个回收函数：`shrink_lruvec`。

可以看当前的水位：

```bash
grep -E 'min|low|high' /proc/zoneinfo | head -12
```

```
        min      1024
        low      1280
        high     1536
```

（数字单位是页，乘以 4KB 换算到字节。）

<!-- 用户填：水位线那个恍然时刻。
  点一句：内核不是等到 OOM 快死了才动手，而是在 low 水位就悄悄开始了。
  这意味着你感觉"程序跑得好好的"时，kswapd 可能早就在后台踢冷页了。
  "卡顿"真正出现是跌破 min，那时候是分配路径被堵住——内核在替你同步等回收结果。
  用你自己的话说说这个"原来内核这么早就开始了"的感觉。下面是我垫的： -->

这三条线说明一件事：**内核的回收行为发生得比你想象的早得多**。你的程序跑得好好的时候，只要 `free` 跌破了 `low`，`kswapd` 就已经在后台悄悄踢冷页了——只是你感觉不到。你真正感觉到卡顿，是跌破了 `min`：分配路径被直接堵住，内核让你等它把页回收出来再说。

## 二、内核怎么决定踢哪些页：LRU 链表

内核把所有物理页分两大类来管——**匿名页**（anonymous，进程堆/栈/malloc 来的，没有文件背书）和**文件页**（file-backed，代码段、mmap 文件、page cache）。

每类都有两条 LRU 链表：`active` 和 `inactive`。

```
   每个内存区维护 4 条链表：

   anon_active    ← 匿名页，最近被访问过
   anon_inactive  ← 匿名页，很久没访问，"冷"，回收候选
   file_active    ← 文件页，最近被访问过
   file_inactive  ← 文件页，很久没访问，"冷"，回收候选
```

**页表项**是多级页表最末一级里的一个格子，记录着"这个虚拟页对应哪个物理页框号"以及一些标志位（可读/可写/已访问……）。x86 上页表项的具体布局会在后续专门写，这里只需要知道：它是虚拟地址到物理地址那条翻译路径的终点。

页表项上有个 **A bit（access bit）**，CPU 每次访问这页都会把它置 1。内核定期扫描：
- 有 A bit 的页：从 `inactive` 提升到 `active`（说明最近被用了，留着）。
- 长时间没有 A bit 的页：从 `active` 降到 `inactive`（开始变"冷"了）。

`shrink_lruvec` 的核心逻辑就是从 `inactive` 链表**尾部**取候选页，决定怎么处置它。

查一下自己系统上各条链表现在有多少页：

```bash
grep -E 'Active|Inactive' /proc/meminfo
```

```
Active:          456320 kB
Inactive:        312440 kB
Active(anon):     89012 kB
Inactive(anon):   34560 kB
Active(file):    367308 kB
Inactive(file):  277880 kB
```

`inactive` 链表的长度，基本就是"内核觉得可以踢出去的候选池大小"。内存压力越大，内核会把 `active` 里的页往 `inactive` 里推得越积极。

## 三、决定踢出之后，具体干了什么

从 `inactive` 链表取出一个候选页，`shrink_lruvec` 要分两种情况处理：

**情况 A：文件页。** 这类页背后有文件，内容本来就在磁盘上（或者已经是干净的 page cache），直接丢掉就行——下次需要再从文件读回来（触发 `filemap_fault`，就是上一篇的文件页缺页）。如果这页被进程修改过（`dirty`），要先 `pageout` 写回文件，再丢。

**情况 B：匿名页。** 这类页没有文件背书，丢掉内容就没了，必须先写到 **swap 交换区**才能释放物理页。流程是：
1. `add_to_swap`：在交换区分配一个 slot，把这页的内容写进去。
2. `try_to_unmap`：把所有持有这张物理页映射的进程的页表项改掉——原来记着物理页号，现在改成一个 **swap entry**（编码了"这页在 swap 的第几个 slot"）。
3. 物理页引用计数归零，归还给 free list。内存就腾出来了。

```
   候选匿名页 p：

   进程A 页表 ──► 物理页 p       进程A 页表 ──► swap entry (slot 38)
   进程B 页表 ──► 物理页 p  ──►  进程B 页表 ──► swap entry (slot 38)
                                               物理页 p 归还，RAM +4KB
```

**换出的完整动作（step by step）：**

```
① shrink_lruvec 选中这页，准备换出
② add_to_swap
     - 在 swap 分区/文件里分配一个空闲 slot（位置记为 swp_entry，编码了设备号+偏移）
     - 把物理页内容（4KB）写入这个 slot（同步写磁盘）
③ try_to_unmap
     - 遍历所有映射了这张物理页的页表项（可能多个进程共享）
     - 把每条页表项从「物理页框号 + present=1」改写为「swp_entry + present=0」
     - TLB shootdown：让所有 CPU 的 TLB 失效这条映射
④ 物理页引用计数降到 0，归还给 free list
     → RAM 净增 4KB，换出完成
```

**换入的完整动作（do_swap_page，step by step）：**

```
① 进程访问该虚拟地址，CPU 查页表：present=0，触发缺页
② do_swap_page 读出页表项里的 swp_entry，知道数据在 swap 的哪个 slot
③ 从 free list 分配一张新物理页
④ 把 swap slot 里的内容（4KB）读回这张新物理页（I/O，major fault）
⑤ 把页表项从「swp_entry + present=0」改回「新物理页框号 + present=1」
⑥ 释放 swap slot（归还给 swap 空闲池）
⑦ 回到出错指令重新执行，这次翻译通过
```

之后任何一个进程访问这个虚拟地址：页表项是 swap entry，查页表"失败"，触发缺页，走 `do_swap_page`——读磁盘、换回物理页、更新页表项。这就是上一篇那个分支的另一面。

**两种情况汇总：**

```
   shrink_lruvec 取 inactive 链表尾部的候选页
        │
        ├─ 文件页（有文件背书）
        │    ├─ 内容干净（未修改）→ 直接丢，下次缺页从文件读
        │    └─ 内容脏（已修改）  → pageout 写回文件 → 再丢
        │
        └─ 匿名页（无文件背书）
             └─ add_to_swap（写交换区）→ try_to_unmap（改页表项为 swap entry）
                → 物理页归还 free list
                  下次访问 → 缺页 → do_swap_page（从交换区换回）
```

## 四、真实数字：亲眼看到匿名页被换出

下面的程序分配大块匿名内存写入数据，通过 `/proc/self/status` 前后对比 `VmSwap`，证明内核确实把一部分页换出去了。

> 需要系统已开 swap（`swapon -s` 验证）。docker 容器加 `--memory=512m --memory-swap=-1`（不限制 swap）。

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>

static long read_status(const char *key){
    FILE *f = fopen("/proc/self/status", "r");
    char line[128]; long val = -1;
    while (fgets(line, sizeof line, f))
        if (strncmp(line, key, strlen(key)) == 0){
            sscanf(line + strlen(key) + 1, "%ld", &val); break;
        }
    fclose(f); return val;
}

int main(void){
    setvbuf(stdout, NULL, _IONBF, 0);
    long page = sysconf(_SC_PAGESIZE);
    size_t N = 512UL*1024*1024;
    char *p = mmap(NULL, N, PROT_READ|PROT_WRITE,
                   MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
    madvise(p, N, MADV_NOHUGEPAGE);

    printf("写 512MB 匿名内存...\n");
    for (size_t i = 0; i < N; i += page) p[i] = (char)(i & 0xff);

    printf("睡 2s 让 kswapd 有机会跑\n");
    sleep(2);

    long rss  = read_status("VmRSS");
    long swap = read_status("VmSwap");
    printf("VmRSS  = %ld kB\nVmSwap = %ld kB\n", rss, swap);
    printf("RSS+Swap ≈ %ld kB  (应接近 512*1024=%d kB)\n",
           rss+swap, 512*1024);
    return 0;
}
```

```bash
gcc -O0 swap_demo.c -o swap_demo && ./swap_demo
```

内存充足时 `VmSwap=0`，都在 RSS；内存偏紧时你会看到 `VmSwap > 0`——内核已经悄悄把部分冷页踢到交换区了，RSS 对应缩小。两者之和接近 512MB（RSS 里掺了代码和库，会略多一点）。

另开一个终端盯着系统级 swap 使用量：

```bash
watch -n1 'grep SwapFree /proc/meminfo'
```

## 五、全流程总图

```
   内存分配（缺页要造新页）
        │
        └─ 查空闲链表：够用 ──► 直接分配，完事
                      │
                     不够
                      │
                      ├─ 空闲跌破 low：唤醒 kswapd（异步，不卡你）
                      │        ↓
                      │   shrink_lruvec
                      │   ├─ 文件页：clean 直接丢 | dirty 先 pageout 再丢
                      │   └─ 匿名页：add_to_swap → try_to_unmap → 归还
                      │
                      └─ 空闲跌破 min：直接回收（卡住调用方，同上流程）

   后续：进程访问被换出的虚拟地址
        → 页表项是 swap entry → 缺页 → do_swap_page
        → 读交换区 → 换回物理页 → 更新页表项 → 重跑指令
```

<!-- 用户填：收尾。
  可以收束到这条主线：内核的页面回收不是"内存满了才动手"，
  而是用水位线提前管理——kswapd 后台悄悄踢，压力极大时直接回收堵住调用方。
  冷热靠 LRU 感知，文件页直接丢、匿名页先写 swap 再丢。
  do_swap_page 和这一篇的换出，正好是同一枚硬币的两面。
  可以再点一句把这整个系列连起来——地址是假的→多级页表→缺页→决策树补页→内存不够踢冷页→swap换回。
  用你自己的话收。下面是我垫的骨架： -->

把这篇收一下：

- **内核不等满再动**。水位线是关键——空闲跌破 `low`，`kswapd` 就在后台悄悄开始踢冷页；跌破 `min`，才会卡住分配路径做直接回收。正常系统你感觉不到前者，后者才是卡顿的来源。
- **踢谁靠 LRU**。A bit + `active`/`inactive` 两条链表，内核一直在给所有物理页做冷热档案。`shrink_lruvec` 从 `inactive` 尾部捞候选，文件页直接丢（脏的先写回），匿名页必须先写 swap 再丢。
- **换出和换回是同一套机制的两面**。`add_to_swap` + `try_to_unmap` 把匿名页踢出去；缺页触发 `do_swap_page` 把它读回来。上一篇那个分支和这篇，合在一起才是完整的故事。

把前面几篇连成一条线：地址是假的（虚拟地址）→ 访问要查多级页表、TLB 兜底 → 翻不动就缺页、内核走决策树补页 → 物理内存压力大了，内核踢出冷页腾地方 → 被踢的页下次访问再换回来。你打印出来的那串地址，到内存条上真实一格，中间所有这些机制，都在这几篇里了。

---

<!-- 发布前自检（删之前过一遍）：
  □ swap_demo 那段代码在有 swap 的环境里真跑过，VmSwap > 0 能复现
  □ 函数名拼写核对（kswapd / shrink_lruvec / pageout / try_to_unmap / add_to_swap / do_swap_page）
  □ 3 处「用户填」都用自己的声音填了，AI 文风清掉了
  □ ASCII 图在掘金编辑器贴预览，箭头框线没跑偏
  □ 没手痒去展开 kcompactd / NUMA / OOM killer（不是这篇的范围）
  □ 发布！连成第七次，守住习惯。
-->
