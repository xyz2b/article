# 多线程 malloc 为什么会变慢——glibc 的 arena 到 bins 全景

---

> **🎯 交互式可视化**：[点击这里体验 arena → bins 全景动画](https://xyz2b.github.io/article/CodeAdventure/front/arena-bins-visualizer.html)
> 你可以切换线程数，实时看 arena 怎么在多线程间分配和复用、五类 bin 怎么接力处理空闲块。
>
> **🧭 静态结构全景**：[点击这里查看 arena 内部完整结构](https://xyz2b.github.io/article/CodeAdventure/front/malloc-pool-static.html)
> 一个预填充的 arena 全景：点 malloc_state 字段跳到对应区域，点任意 chunk 弹出它的完整内存布局，看清 fastbin / unsorted / small / large / top 每一类 bin 的内部细节。

---

上一篇《[free 完再 malloc 为什么拿回同一块](https://github.com/xyz2b/article/blob/main/CodeAdventure/free完再malloc为什么拿回同一块.md)》末尾讲 fastbin 时留了个钩子：

> "fastbin **不是线程私有**的，所以要上锁……**谁和谁共用、那把锁到底锁的是什么**，牵出的是 glibc 的 arena（分配区）机制——本篇用不到，留到下一篇专门讲。"

这篇就来兑现。

入口是一个可以立刻复现的现象：两行代码，一个在主线程、一个在子线程——两次 `malloc(16)`，拿到的地址差了**将近 128 GB**。

<a id="code-intro"></a>

```c
// intro.c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

static void *worker(void *arg) {
    void *p = malloc(16);
    printf("子线程 malloc(16) = %p\n", p);
    return NULL;
}

int main(void) {
    void *p = malloc(16);
    printf("主线程 malloc(16) = %p\n", p);
    pthread_t t;
    pthread_create(&t, NULL, worker, NULL);
    pthread_join(t, NULL);
    return 0;
}
```

```bash
gcc -O0 -pthread -o intro intro.c && ./intro
```

真实输出：

```
主线程 malloc(16) = 0x4052a0
子线程 malloc(16) = 0x7ffff8000b70
```

两块地址差了约 `0x7ffff7fc0000` 字节——将近 128 GB。这不是偶然，而是 glibc 刻意安排：它给线程各开了一个**独立的内存池**。这个池子，就叫 **arena（分配区）**。

> 说明同系列前三篇：x86-64 + glibc 2.36（Debian 12 里的 `gcc:13` 容器，glibc 2.36-9+deb12u14）。文中所有代码和输出都是真编真跑的。结论看关系，不看具体的地址数字。GDB dump 部分在 `--platform linux/arm64 --cap-add=SYS_PTRACE` 的原生 arm64 容器里跑（amd64 经 QEMU 跑 GDB 会报 `Couldn't get CS register`）；malloc_state 结构体布局在所有 64 位平台完全一致，arm64 dump 出的 sizeof、字段偏移、槽位数量和 x86-64 完全相同，只有地址数值不通用。

## 一、现象：线程一多，malloc 就"分家"——还会卡上限

主/子线程用的不是同一个池子，那多加线程，池子数量怎么变？实测：

关键是怎么"数 arena"。glibc 没有公开的计数接口，但 `malloc_info()` 会把每个 arena 输出成一个 `<heap>` XML 元素——数 `<heap` 出现几次就有几个 arena。再用一个屏障（barrier）让所有线程都分配完、同时存在时再统计，避免有人提前退出导致漏数：

<a id="code-a_arena"></a>

```c
// a_arena.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <malloc.h>

static int NTHREAD;
static pthread_barrier_t barr;

static void *worker(void *arg) {
    void *p = malloc(64);
    memset(p, 0xff, 64);            // 写一下，确保 arena 真正建立
    pthread_barrier_wait(&barr);    // ① 都分配完再统计，保证 arena 同时存在
    pthread_barrier_wait(&barr);    // ② 等主线程统计完再退出
    return NULL;
}

static int count_arenas(void) {
    char *buf = NULL; size_t len = 0;
    FILE *f = open_memstream(&buf, &len);
    malloc_info(0, f);              // 把内部状态吐成 XML
    fclose(f);
    int n = 0;
    for (char *s = buf; (s = strstr(s, "<heap ")); s += 6) n++;  // 数 <heap 元素
    free(buf);
    return n;
}

int main(int argc, char **argv) {
    NTHREAD = atoi(argv[1]);
    pthread_barrier_init(&barr, NULL, NTHREAD + 1);
    malloc(64);                     // 主线程在 main_arena 也分配一块

    pthread_t *t = calloc(NTHREAD, sizeof(pthread_t));
    for (long i = 0; i < NTHREAD; i++) pthread_create(&t[i], NULL, worker, (void*)i);

    pthread_barrier_wait(&barr);    // 等所有子线程都分配完
    printf("线程数=%-4d  arena 数=%d\n", NTHREAD, count_arenas());
    pthread_barrier_wait(&barr);    // 放子线程走

    for (int i = 0; i < NTHREAD; i++) pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -O0 -pthread -o a_arena a_arena.c
for n in 1 2 4 8 16 32 64 79 80 81 100 128 200; do ./a_arena $n; done
```

真实输出（容器 nproc=10）：

```
线程数=1     arena 数=2
线程数=2     arena 数=3
线程数=4     arena 数=5
线程数=8     arena 数=9
线程数=16    arena 数=17
线程数=32    arena 数=33
线程数=64    arena 数=65
线程数=79    arena 数=80   ← 封顶（79子线程+主线程=80）
线程数=80    arena 数=80
线程数=81    arena 数=80   ← 再加线程也不涨了
线程数=100   arena 数=80
线程数=128   arena 数=80
线程数=200   arena 数=80
```

规律清晰：每多一个线程就多一个 arena，但到**80 = 8 × nproc**（这台机器 nproc=10）就冻结不再涨。继续加线程，不再分家，多出来的线程必须**共用现有的 arena**。

共用意味着要争——争同一个内存池的操作权。争的方式是加锁：拿到锁才能分配，没拿到就等。这就是"多线程 malloc 为什么会变慢"里那把锁的来历。

用数据感受一下：相同工作量，8 个线程，有没有锁竞争，吞吐差多少？每个线程就反复 malloc/free 一块小内存，计总耗时：

<a id="code-a_lock"></a>

```c
// a_lock.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>

#define ROUNDS 2000000
static int NTHREAD;

static void *worker(void *arg) {
    for (long i = 0; i < ROUNDS; i++) {
        void *p = malloc(32);
        *(volatile char*)p = 1;     // 写一下，防编译器优化掉
        free(p);
    }
    return NULL;
}

int main(int argc, char **argv) {
    NTHREAD = atoi(argv[1]);
    pthread_t *t = calloc(NTHREAD, sizeof(pthread_t));

    struct timespec a, b;
    clock_gettime(CLOCK_MONOTONIC, &a);
    for (long i = 0; i < NTHREAD; i++) pthread_create(&t[i], NULL, worker, (void*)i);
    for (int i = 0; i < NTHREAD; i++) pthread_join(t[i], NULL);
    clock_gettime(CLOCK_MONOTONIC, &b);

    double ms = (b.tv_sec - a.tv_sec) * 1000.0 + (b.tv_nsec - a.tv_nsec) / 1e6;
    long total = (long)NTHREAD * ROUNDS;
    printf("线程数=%d  总计%ld次  耗时%.0f ms  (%.1f M ops/s)\n",
           NTHREAD, total, ms, total / ms / 1000.0);
    return 0;
}
```

编译后，单线程和 8 线程各跑两种配置，对照有锁竞争和没锁竞争：

```bash
gcc -O2 -pthread -o a_lock a_lock.c
# 默认：每线程可有独立 arena，互不干扰
./a_lock 1
./a_lock 8
# MALLOC_ARENA_MAX=1：强制所有线程共用一个 arena，人为制造锁竞争
MALLOC_ARENA_MAX=1 ./a_lock 1
MALLOC_ARENA_MAX=1 ./a_lock 8
```

真实输出（每线程 200 万次 malloc/free）：

```
默认:
  线程数=1  总计 2000000次  耗时 19 ms  (105.5 M ops/s)
  线程数=8  总计16000000次  耗时 37 ms  (427.3 M ops/s)
MALLOC_ARENA_MAX=1:
  线程数=1  总计 2000000次  耗时 19 ms  (107.2 M ops/s)
  线程数=8  总计16000000次  耗时123 ms  (130.3 M ops/s)
```

先看**单线程**：两种配置都是 19 ms，没差——因为单线程根本不存在"争"，有几个 arena 都一样。再看 **8 线程**：默认配置 37 ms，强制共用一个 arena 后飙到 123 ms，慢了 **3.3 倍**。单线程做平、多线程拉开——差距完全来自锁竞争，不是线程多了本身就慢。

好，现象钉死了。接下来就是回答：这个"池子"到底是什么结构？它内部的空闲块怎么组织？那把锁锁的是什么？

## 二、整体：一个 arena = 一个 malloc_state + 一片堆

### 2.1 malloc_state：arena 的"账本"

glibc 里，每个 arena 的管理信息都压在一个叫 **`malloc_state`** 的结构体里。`main_arena` 是 libc 里的一个全局符号，所以不用自己定义、不用拿地址，GDB 里直接按名字访问它就行。

为了同时看到"多 arena 环链"和"fastbin 里挂着块"，写一个会触发第二个 arena、并在 fastbin 里留一块的小程序当观察对象：

<a id="code-a_gdb"></a>

```c
// a_gdb.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

static void *worker(void *arg) {
    void *p = malloc(32);       // 子线程分配 → 触发创建第二个 arena
    *(volatile char*)p = 1;
    sleep(100);                 // 挂住，给 GDB 留出 attach/断点的时间
    return NULL;
}

int main(void) {
    void *a = malloc(32);
    *(volatile char*)a = 1;
    void *blk[8];
    for (int i = 0; i < 8; i++) blk[i] = malloc(32);
    for (int i = 0; i < 8; i++) free(blk[i]);   // 7 进 tcache，第 8 块溢出进 fastbin
    pthread_t t;
    pthread_create(&t, NULL, worker, NULL);
    pthread_join(t, NULL);
    return 0;
}
```

带 `-g` 编译后，用 GDB 把进程停在 `return 0` 那行（此时 8 块已 free、子线程的 arena 也已建好），就能 dump `main_arena`：

```bash
gcc -O0 -g -pthread -o a_gdb a_gdb.c
gdb -q ./a_gdb
(gdb) break a_gdb.c:23        # 停在 return 0
(gdb) run
```

> 注意：这一段必须在原生 arm64 容器里跑（`--platform linux/arm64 --cap-add=SYS_PTRACE`）。amd64 镜像经 QEMU 模拟时，GDB 读寄存器会报 `Couldn't get CS register`，断不下来。malloc_state 的结构体布局所有 64 位平台一致，下面的 sizeof、槽位数都通用，只有具体地址数值不通用。

停下后，先看结构体多大：

```
(gdb) print sizeof(main_arena)
$1 = 2200
```

2200 字节的账本，管着进程的全部堆内存分配。里面最关键的几个字段：

```
(gdb) print main_arena.mutex           → 0   （锁，0=未持有）
(gdb) sizeof(main_arena.fastbinsY)/sizeof(fastbinsY[0])  → 10  （fastbin 槽数）
(gdb) sizeof(main_arena.bins)/sizeof(bins[0])            → 254 （其他 bin 槽数）
(gdb) print main_arena.top             → 0x...（top chunk 指针）
(gdb) print main_arena.next            → 0x7f...（指向下一个 arena）
```

```
   malloc_state（2200 字节的账本）：

   ┌────────────────────────────────────────────┐
   │ mutex          4B   ← 那把锁               │
   │ flags          4B                          │
   │ have_fastchunks 4B                         │
   │ fastbinsY[10] 80B   ← 10 个 fastbin 槽     │
   │ top            8B   ← top chunk 指针       │
   │ bins[254]    2032B  ← unsorted/small/large │
   │ next           8B   ← 下一个 arena         │
   │ next_free      8B                          │
   │ ...                                        │
   └────────────────────────────────────────────┘
        ↑ 这个结构体就是一个 arena 的全部索引
```

`mutex` 就是那把锁——`free`/`malloc` 操作 fastbin 或 bins 时，必须先持有它。**一把锁管着整个账本里所有的空闲链表**，这就是为什么多个线程争同一个 arena 时会互相等待。

### 2.2 main_arena 和线程 arena

glibc 里有两种 arena：

**main_arena**：静态全局变量，随进程启动就存在，管着 `brk` 扩出来的主堆（地址在 `0x4xxxxx` 的低地址堆区）。主线程默认用它。

**线程 arena**：子线程第一次 `malloc` 时，glibc 并不会去碰 `main_arena`——只要 arena 总数还没到上限，它就直接用 `mmap` 新开一块地，在那块地的起始位置存一个 `malloc_state`，这就是第二个 arena。注意这跟 `main_arena` 当时有没有被锁无关：开篇 intro.c 里主线程那次 `malloc` 早就结束、锁也释放了（它正卡在 `pthread_join` 干等），子线程照样新开一块——给新线程开独立池子是 glibc 主动的策略，不是抢不到锁的退路。只有 arena 数量到了上限，新线程才改为去争抢现成的 arena（详见 2.3）。它管的堆是那块 mmap 区域（地址在 `0x7fxxxxxx` 的高地址）——这就是开篇主线程和子线程地址差了 128 GB 的原因：一个在低地址 brk 堆，一个在高地址 mmap 区。

所有 arena 通过 `malloc_state.next` 串成一个**环形链表**。GDB 坐实：

```
(gdb) print main_arena.next
$2 = (struct malloc_state *) 0x7f...0030   ← 第二个 arena

(gdb) print main_arena.next->next
$3 = (struct malloc_state *) 0x7f...af0 <main_arena>   ← 绕回来了
```

`next→next` 回到 `&main_arena`，完整的环形链表。

### 2.3 线程怎么挑 arena，上限为什么是 8×nproc

线程每次 `malloc` 时，glibc 先去找一个**没被锁住**的 arena：

1. 先试上次用过的 arena，`trylock` 一下——锁到了，直接用；
2. 锁没到，顺着 `next` 环往后转，逐个 `trylock`；
3. 转一圈都没空闲的，**且还没到上限**，就新建一个；
4. 已到上限（`MALLOC_ARENA_MAX = 8 × nproc`），就**等**或随机挑一个，等它解锁。

上限 `8 × nproc` 是经验值：太少会有锁竞争，太多会碎片化、TLB 压力大。公式里 `nproc` 取的是机器逻辑 CPU 数（这台是 10），所以上限是 80。实测曲线正好在第 79 个子线程（加上主线程共 80 个 arena）时触顶，之后冻结。

> **版本说明**：`8 × nproc` 上限自 **glibc 2.10**（2009 年，per-thread arena 引入时）起保持不变，glibc 2.36 与更早版本在这个数字上无差异。注意这是 **64 位平台**的系数；32 位平台系数为 `2`（即上限 `2 × nproc`）。通过 `MALLOC_ARENA_MAX` 环境变量或 `mallopt(M_ARENA_MAX, n)` 可以手动覆盖。

```
arena 数量随线程增长（nproc=10）：
   ──────────────────────────────────────────────────── 80 (= 8×nproc)
   65                                               ▓▓▓▓▓▓▓▓▓▓▓
                                         ▓▓▓▓▓▓▓▓▓▓
                               ▓▓▓▓▓▓▓▓▓
                     ▓▓▓▓▓▓▓▓▓
           ▓▓▓▓▓▓▓▓▓
    ▓▓▓▓▓▓▓
    1  2  4  8 16 32 64 79 80 81 100 128 200  线程数
```

---

## 三、局部：一个 arena 里的 5 类 bin 全景

有了整体，现在下钻到 arena 内部。**空闲块不是堆在一块——是按大小和特点，分进 5 类不同结构的链表**，合称 "bins"。这一章把每类 bin 的静态结构（在 `malloc_state` 的哪个字段）和动态行为（`free` 怎么放进去、`malloc` 怎么取出来）交织着讲完。

先做一件基础工作：搞清分桶用的是**实际 chunk 大小**，不是你 `malloc` 时要的字节数。办法很直接——`malloc` 一块，再从用户指针往前 8 字节读出 chunk 头里的 `size`，清掉低 3 位标志位（下一节细说这 3 位）就是实际 chunk 大小：

<a id="code-bucket"></a>

```c
// bucket.c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

static size_t chunksize(void *p) {
    size_t sz = *((size_t*)p - 1);   // chunk 头在用户指针前 8 字节
    return sz & ~(size_t)0x7;        // 低 3 位是标志位，清掉才是真大小
}

int main(void) {
    int reqs[] = {0, 8, 16, 24, 25, 40, 41, 56, 120, 121, 1032, 1033};
    printf("请求大小 -> 实际 chunk 大小:\n");
    for (int i = 0; i < (int)(sizeof(reqs)/sizeof(reqs[0])); i++) {
        void *p = malloc(reqs[i]);
        printf("  malloc(%5d)  chunk = %zu\n", reqs[i], chunksize(p));
    }
    return 0;
}
```

```bash
gcc -O0 -o bucket bucket.c && ./bucket
```

真实输出：

```
请求大小 -> 实际 chunk 大小:
  malloc(    0)  chunk = 32
  malloc(    8)  chunk = 32
  malloc(   16)  chunk = 32
  malloc(   24)  chunk = 32
  malloc(   25)  chunk = 48
  malloc(   40)  chunk = 48
  malloc(   41)  chunk = 64
  malloc(   56)  chunk = 64
  malloc(  120)  chunk = 128
  malloc(  121)  chunk = 144
  malloc( 1032)  chunk = 1040
  malloc( 1033)  chunk = 1056
```

规则：请求字节 + 8（chunk 头）→ 向上对齐 16 → 下限 32。`malloc(0/8/16/24)` 都是 chunk 32，`malloc(25)` 才跳到 chunk 48。所有分桶逻辑看的都是这个 chunk 大小。

### 3.0 先看清一块 chunk 长什么样

后面所有 bin 都是把"空闲 chunk"串成链表，所以得先把 chunk 这个最小单位的内存布局钉死。把前面 [`a_gdb.c`](#code-a_gdb) 里那块**正在用的** `a`（`malloc(32)`）和一块**已空闲的** chunk 都从 GDB 里 dump 出来对照：

```
正在用的 chunk a（malloc(32)，刚写过 *a=1）：
(gdb) print a
$1 = (void *) 0x...2a0
(gdb) x/4gx ((char*)a - 16)
0x...290:  0x0000000000000000  0x0000000000000031   ← prev_size | size
0x...2a0:  0x0000000000000001  0x0000000000000000   ← 用户数据(那个 1) 起
           ↑ malloc 返回给你的就是这个地址(0x...2a0)

已空闲的 chunk（fastbin 里那块）：
(gdb) x/4gx (char*)freed
0x...410:  0x0000000000000000  0x0000000000000031   ← prev_size | size
0x...420:  0x000000000003e756  0x0000000000000000   ← fd | bk
           ↑ 原本是用户数据区，现在头 8 字节被借去当链表指针 fd
```

两块的头部一模一样：`prev_size` + `size`，画起来是 16 字节，但每块实际只摊到 **8 字节**开销——前面那句"请求 + 8 字节头"就是这么来的。窍门在 `prev_size`：它只在**前一块空闲时**才存前块大小；前一块在用时，这 8 字节会被前块当用户数据借走。所以 `malloc(24)` 能塞进 chunk32（24 + 8 = 32），多出的 24 字节里就含了后一块借给它的 `prev_size` 空间。

`size` 字段是 `0x31` = 49，它把"大小"和 3 个标志位压在一个字段里。`0x31` = 二进制 `110001`，**最低 3 位**是标志位：

```
   0x31 = 0b 1 1 0 0 0 1
             ───┬─── ┬ ┬┬
                │    │ │└ bit0 P (PREV_INUSE，前一块是否在用)  = 1
                │    │ └─ bit1 M (IS_MMAPPED，是否 mmap 来的)  = 0
                │    └─── bit2 A (NON_MAIN_ARENA，是否非主arena)= 0
                └──────── 高位 110000 = 0x30 = 48 = 真正的 chunk 大小
```

把低 3 位清零（`0x31 & ~0x7 = 0x30 = 48`）才是真正的 chunk 大小——这也是为什么 chunk 大小永远是 8 的倍数：低 3 位得空出来当标志。前面 [`bucket.c`](#code-bucket) 读 chunk 大小时那句 `sz & ~(size_t)0x7`，做的就是这件事。

关键对照在数据区：`a` 正在用，`0x...2a0` 放的是你的数据（那个 `1`）；而那块空闲 chunk，**同样的偏移**（`0x...420`）已经不是用户数据了，被 libc 借去存链表指针 `fd`（`0x3e756`，第三篇讲过的 safe-linking 加扰值）。**"空闲块用自己的数据区当链表节点"——所有 bin 链表都靠这个机关，不额外花一分钱内存。**

```
   一块 chunk 的完整布局：

   ┌─────────────────┐ ← chunk 起始(内部地址)
   │ prev_size  8B   │   前块空闲时存前块大小，前块在用时被前块借用
   ├─────────────────┤
   │ size       8B   │   本块大小 | A | M | P(低3位标志)
   ├─────────────────┤ ← malloc 返回给你的地址(用户指针)
   │                 │
   │   用户数据区     │   ← 正在用:你的数据
   │                 │      ← 空闲后:被借作 fd/bk 链表指针
   └─────────────────┘
```

### 3.1 tcache：线程私有的快车道（独立于 arena）

> **glibc 版本注脚**：tcache 是 **glibc 2.26（2017 年）** 才引入的。2.26 以前没有这一层线程私有缓存，小块 `free` 后直接进 fastbin / unsorted bin，`malloc` 时也从 fastbin 摘取。"free 完再 malloc 拿回同一块"的 LIFO 现象在旧版同样成立（fastbin 本身也是后进先出），但没有"每桶 7 块"上限，也不存在"溢出到 fastbin"这个说法。如果你的目标环境是 glibc < 2.26，可跳过本节，直接看 3.2 fastbin。

**tcache 不在 `malloc_state` 里**，也不属于 arena——它是每个线程独有的一块内存，随线程第一次 `malloc` 时创建，跟着线程走。结构同样是按 chunk 大小分桶的单向链表，但有几点和后面的 bin 截然不同：

| | tcache | arena 里的 bin |
|---|---|---|
| 归属 | 线程私有 | 属于某个 arena |
| 加锁 | **不需要** | 必须持有 arena.mutex |
| 桶数 | **64** 个（chunk 32~1040）| 因 bin 类型而异 |
| 每桶上限 | **7** 块 | 无上限（各类不同） |
| 覆盖请求 | ≲1032 字节 | 视类型 |

**那 64 个桶的链表头存在哪？** 不在 `malloc_state` 里（那是 arena 的账本，线程私有的 tcache 不能放那）。glibc 用一个单独的结构体管它，GDB 里能直接看到它的定义：

```
(gdb) ptype tcache_perthread_struct
type = struct tcache_perthread_struct {
    uint16_t      counts[64];     ← 每个桶当前囤了几块(用来判断是否满 7)
    tcache_entry *entries[64];    ← 64 个桶的链表头指针
}
(gdb) print sizeof(tcache_perthread_struct)
$1 = 640                          ← 64*2 + 64*8 = 640 字节
```

这个 640 字节的结构体**本身也是从堆上 malloc 出来的**——而且是每个线程在堆上分配的**第一块** chunk。在 [`a_gdb.c`](#code-a_gdb) 断点处，用第一个分配指针反推堆头，能看到它就趴在堆的最前面：

```
(gdb) x/2gx (堆基址)
0x...000:  0x0000000000000000  0x0000000000000291
                                        ↑ 0x291 = 656 = 640 + 16头部 | PREV_INUSE
           ← 堆上第一个 chunk，就是 tcache_perthread_struct

(gdb) print tcache_struct->counts[1]    # chunk48 那档
$2 = 7                                   ← 正好 7，桶已满(程序 free 了 8 块，1 块溢出去 fastbin)
(gdb) print tcache_struct->counts[0]
$3 = 0
```

`counts[1] = 7` 这个数字直接坐实了：程序连续 free 了 8 块 `malloc(32)`（chunk48），前 7 块把 tcache 桶 1 填满（`counts[1]=7`），第 8 块溢出——溢出去哪了？正是下一节 fastbin 里那块。

再有一个 thread-local 指针 `tcache` 指向这个结构，线程访问自己的桶就靠它。**所以"tcache 挂在哪"的完整答案是：一个 `tcache_perthread_struct` 结构体（堆上第一块 chunk），里面 `entries[64]` 是 64 个桶的链表头，`counts[64]` 记每桶块数；一个线程私有指针 `tcache` 指着它。**

```
   thread-local 指针         堆上第一个 chunk
   ┌────────┐              ┌──────────────────────────────┐
   │ tcache │ ───────────→ │ prev_size | size=0x291        │
   └────────┘              │ counts[64]   ← 各桶块数        │
   (每线程一份)             │ entries[0] → [块]→[块]→NULL    │ 桶0
                           │ entries[1] → [块]→...          │ 桶1
                           │   ...                          │
                           │ entries[63]→ NULL              │ 桶63
                           └──────────────────────────────┘
```

**动态行为**：

`free` 时，先看该 chunk 对应的 tcache 桶有没有满（`counts[i] < 7`）。没满就**头插**进 `entries[i]`、`counts[i]++`，完事，全程不碰 arena 也不加锁，是最快路径。

`malloc` 时，第一个检查的就是 tcache：算出桶下标，`entries[i]` 非空就**头摘**、`counts[i]--`，直接返回，同样不加锁。

桶下标：`(chunk_size - 32) / 16`——直接算，O(1)。比如 `malloc(16)`→chunk32→桶 0，`malloc(32)`→chunk48→桶 1，对应上面 `counts[1]=7` 那个填满的桶。

tcache 满了（`counts[i]` 攒够 7）之后，多出来的块才往下落，进 arena 的 bin。

### 3.2 fastbin：小块专属的第二快车道

**静态位置**：`malloc_state.fastbinsY[10]`——10 个槽，对应 10 档 chunk 大小。

为什么只有 10 个槽但实际用 7 个？`global_max_fast = 128` 字节，只接收 chunk ≤ 128 的块，对应 **chunk 32 / 48 / 64 / 80 / 96 / 112 / 128 共 7 档**（请求约 1~120 字节）。另外 3 个槽预留，实测请求 121（chunk 144）就已经不进 fastbin 了。

GDB 坐实：

```
(gdb) print main_arena.fastbinsY
$4 = {0x0, 0x...410, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}
         ↑           ↑
       槽0空        槽1有块（chunk48 那档，程序里溢出的那块）

(gdb) x/2gx main_arena.fastbinsY[1]
0x...410:  0x0000000000000000  0x0000000000000031
                                        ↑ 0x31 = 48 | PREV_INUSE，正是 chunk 48
```

**数据结构**：LIFO 单向链表，不做相邻空闲块合并（为了快，代价是容易碎片化）。

**为什么比 tcache 慢**：fastbinsY 属于 `malloc_state`，多线程共享。操作时必须持有 `arena.mutex`——这就是上一篇留的钩子：**那把锁锁的是整个 `malloc_state`，fastbin 在里面，所以要加锁**。

**动态行为**：

`free` 时，tcache 对应桶已满（7 块）→ 检查 chunk ≤ global_max_fast → 是，则头插进 fastbinsY 对应槽（需要先获取 mutex）。

`malloc` 时，tcache 对应桶空了 → 检查 fastbinsY 对应槽 → 非空则头摘一块，**同时把 fastbin 里其余块批量填回 tcache**（tcache 还有空位的话），下次再 malloc 同 size 就能直接走免锁的 tcache。

```
   fastbinsY[10]（在 malloc_state 里，要锁）：

   槽 0  [chunk 32]  ──→ chunk ──→ chunk ──→ NULL
   槽 1  [chunk 48]  ──→ chunk ──→ NULL
   槽 2  [chunk 64]  ──→ NULL
     ...
   槽 6  [chunk128]  ──→ chunk ──→ NULL      ← 7档最大（请求120字节）
   槽 7~9           NULL（预留，默认不用）
        单向链表，LIFO，不合并，只收 chunk≤128
```

### 3.3 unsorted bin：新 free 块的中转站

chunk 大小超出 fastbin 范围（chunk > 128），或者 fastbin 里的块在 `malloc` 时被合并了之后，空闲块先不直接分类进 small/large bin，而是**统一扔进一个中转站**——unsorted bin。

**静态位置**：`malloc_state.bins[1]`（下标 1，专用一个双向循环链表）。

**为什么要中转站**：分类进 small/large bin 有代价（要找到对应 bin 插入正确位置），而且刚 free 的块很可能马上又被 malloc 同 size——中转一步，让 `malloc` 先过来这里"顺手捡走"，省得白费力气归类又立刻取走。

**动态行为**：

`free` 一块 chunk > 128 的块：先尝试和**物理相邻**的空闲块**合并（consolidate）**，合并后的块**头插进 unsorted bin**，等待后续处理。

这里的"相邻"是个关键、又最容易误解的词——**它指的不是链表上的前后，而是内存地址上的紧挨着**。回忆 3.0 节：堆上的 chunk 是一块挨一块连续排布的，每块开头有 `prev_size | size`。所以从一块 chunk 出发，找它的物理邻居完全靠地址算：

- **后一块**：当前 chunk 起始地址 + 本块 `size`，就是紧跟在后面那块的起始地址；
- **前一块**：看本块 `size` 字段的 `PREV_INUSE` 位（3.0 讲的最低位 P）。P=0 说明前一块是空闲的，此时本块开头的 `prev_size` 记着前块多大，地址减去 `prev_size` 就定位到前块。

合并就是：如果后一块（或前一块）也是空闲的，把它从原来的 bin 链表里摘下来（unlink），两块的 `size` 相加，当成一整块更大的 chunk。一句话——**物理上连续的两块空闲内存，合成一块连续的大空闲块**，这样下次有人要大块时装得下，碎片就少了。

跑出来看。要一块单块装不下、只有相邻两块合并后才装得下的大小，就能证明合并真的发生了：

<a id="code-a_consolidate"></a>

```c
// a_consolidate.c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    char *a = malloc(2000);   // chunk 2016
    char *b = malloc(2000);   // 紧跟在 a 后面
    char *c = malloc(2000);   // 守卫块：挡在 b 和堆顶之间，防止 b 被并进 top
    (void)c;
    printf("a=%p  b=%p  间距 b-a=%ld\n", a, b, (long)(b - a));
    free(a);                  // a 进 unsorted
    free(b);                  // b 与 a 物理相邻、a 又空闲 → 合并
    char *big = malloc(4000); // chunk≈4016，单块 2016 装不下，只有 a+b 合并(≈4032)才够
    printf("malloc(4000) = %p  (==a? %s)\n", big, big == a ? "是" : "否");
    return 0;
}
```

```bash
gcc -O0 -o a_consolidate a_consolidate.c && ./a_consolidate
```

真实输出：

```
a=0x4052a0  b=0x405a80  间距 b-a=2016
malloc(4000) = 0x4052a0  (==a? 是)
```

`b-a` 间距正好等于 a 的 chunk 大小 2016——证明 a、b 在内存里就是紧挨着的。`free(a)` 再 `free(b)` 之后，要一块 4000 字节（chunk≈4016，比单块 2016 大得多），居然拿回了 `a` 的地址：只有 a、b 合并成了一块 ≈4032 的连续空闲块，才装得下。**这就是 consolidate——物理相邻才能合并，合并看的是地址不是链表。**

**那一次 free 到底合并几块？** 就 3 块封顶：**前邻 + 当前块 + 后邻**。free 一块时，glibc 只看它紧挨着的前一块和后一块——哪个空闲就并哪个，两个都空闲就把三块并成一块，一次合并到此为止，不会顺着再往外滚。

可前面那个实验明明把 a、b 并成了一块，怎么只算"前后各一"？关键在 glibc 维持的一条不变量：**任意两块物理相邻的（非 fastbin）空闲块，绝不允许同时存在**——因为每次 free 都当场合并掉了。所以当你**连续** free 多块相邻的块时，是一次次**接力**合并的：

```
   free(a)：a 空闲，邻居 b 还在用 → 不并，unsorted 里是 [a]
   free(b)：b 的前邻 a 空闲 → 并成 [ab]（这一步就是 3 块规则里的"前邻+当前"）
   free(c)：c 的前邻正是那块 [ab]（空闲）→ 再并成 [abc]
```

每次 free 仍然只碰前后各一块，但因为前一块本身已经是上一轮合并好的大块，**连续 free N 块相邻内存，就会逐次滚成一整块**。验证一下，连续 free 三块、再要一块"单块、两块都装不下，只有三块合并才够"的内存：

<a id="code-a_consolidate3"></a>

```c
// a_consolidate3.c
#include <stdio.h>
#include <stdlib.h>

static size_t chunksize(void *p){ return (*((size_t*)p - 1)) & ~(size_t)0x7; }

int main(void) {
    char *a = malloc(2000), *b = malloc(2000), *c = malloc(2000);
    char *d = malloc(2000); (void)d;   // 守卫块，挡在 c 和 top 之间
    printf("间距 b-a=%ld  c-b=%ld  单块chunk=%zu\n",
           (long)(b-a), (long)(c-b), chunksize(a));
    free(a); free(b); free(c);         // 逐次接力：[a]→[ab]→[abc]
    char *big = malloc(6000);          // chunk≈6016 > 两块4032，必须三块合并(≈6048)
    printf("malloc(6000) = %p  (==a? %s)  chunk=%zu\n",
           big, big==a?"是":"否", chunksize(big));
    return 0;
}
```

```bash
gcc -O0 -o a_consolidate3 a_consolidate3.c && ./a_consolidate3
```

真实输出：

```
间距 b-a=2016  c-b=2016  单块chunk=2016
malloc(6000) = 0x4052a0  (==a? 是)  chunk=6016
```

要 6000 字节（chunk≈6016，比两块合并的 4032 还大），拿回了 a——a、b、c 三块滚成了一整块。**所以"合并多少块"的完整答案是：单次 free 最多碰前后各一块（共 3 块）；但靠"相邻空闲块不并存"这条不变量，连续 free 多少块相邻内存，就接力合并成多大一块。**

> 反过来也解释了上一篇说的 fastbin "不合并所以快、但碎片化"：fastbin 里的块 free 时**故意不做 consolidate**（连 `PREV_INUSE` 都不清，让邻居以为它还在用），省掉合并开销，所以 fastbin 里允许一长串相邻空闲块并存；代价是这些小空闲块各自为政，凑不成大块。直到某次需要才由 `malloc_consolidate` 把这一长串**一口气全合并**、倒进 unsorted bin。

`malloc` 在 tcache 和 fastbin 都没找到合适块时，开始**遍历 unsorted bin**：

- 每取一个 chunk，先看大小**精确符合**请求吗？符合就立刻返回（remainder 放回 unsorted）；
- 不符合：判断 chunk 属于 small 还是 large 范围，**挪到对应的 small bin 或 large bin 归位**，继续遍历；
- 遍历完还没找到：去找 small bin / large bin。

unsorted bin 是整个 bins 里唯一"不分类存放"的链表，是一个 FIFO 双向链表（新块头插，遍历从尾部开始摘）。

```
   bins[1] = unsorted bin（双向循环链表）：

   head ↔ [chunk A 合并后] ↔ [chunk B 切剩余] ↔ [chunk C 大块 free] ↔ head
           ↑ 各种大小混在一起，malloc 来了再分类
```

### 3.4 small bin：精确分档的 FIFO 队列

chunk 大小 ≤ 1008 字节（包括 fastbin 范围内但不被 fastbin 接收的情况，实际上 small bin 从 chunk 16 起，但与 tcache/fastbin 共存下，实际活跃范围是 chunk 144~1008）进 small bin。

**静态位置**：`malloc_state.bins[2]` 到 `bins[63]`，共 **62 个桶**，每桶 16 字节一档（和 tcache 同样的等差排布）。

**数据结构**：FIFO 双向链表——`free` 时插头，`malloc` 时摘尾，先来先用。

**动态行为**：

从 unsorted bin 归类进来的 small chunk，按其 chunk 大小，下标 `(chunk_size - 32) / 16 - 1` 找到对应 small bin（偏移修正是因为 bins 下标从 1 开始且 bins[1]=unsorted），插入链表头。

`malloc` 请求落在 small 范围时：先算桶下标，对应 small bin 非空则 **FIFO 摘尾**——注意是摘尾不是摘头，先 free 的先被取走，和 tcache/fastbin 的 LIFO 相反。这样有利于将先 free 的（时间最长的）块优先复用，时间局部性稍差但碎片更少。

```
   small bins（bins[2..63]，每档 16B）：

   bins[2]  [chunk  48]  head ↔ B ↔ A ← malloc 取这端（FIFO，摘尾）
   bins[3]  [chunk  64]  head ↔ C
   bins[4]  [chunk  80]  head ↔ ...
     ...
   bins[63] [chunk1008]  head ↔ ...
        双向链表，FIFO，精确匹配，不拆分
```

### 3.5 large bin：范围分档 + 最佳适配

chunk ≥ 1024 字节（即请求 ≳ 1016 字节，超出 small bin 最大 chunk 1008）进 large bin。

**静态位置**：`malloc_state.bins[64]` 到 `bins[126]`，共 **63 个桶**，但不再是等差 16——按**对数区间**分档（越大的块档越粗）：

```
bins[64~95]:   32 字节一档（chunk 1024~2016）
bins[96~111]:  64 字节一档
bins[112~119]: 128 字节一档
bins[120~123]: 256 字节一档
...（越来越粗）
```

**数据结构**：双向链表（`fd`/`bk`），桶内**按大小从大到小排序**（链表头最大）。但和 small bin 不同，**一个 large bin 桶里装的是一段大小范围**（比如 bins[64] 装 chunk 1024~1056），所以同一个桶里有好多种不同大小的块混着。为了不必每次都从头扫整条链，glibc 给 large bin 加了第二层指针——就是 3.3 节 `malloc_chunk` 结构体里那对 `fd_nextsize` / `bk_nextsize`：

```
(gdb) ptype struct malloc_chunk
struct malloc_chunk {
    size_t mchunk_prev_size;
    size_t mchunk_size;
    struct malloc_chunk *fd, *bk;              ← 主链:桶内所有块，按大小降序
    struct malloc_chunk *fd_nextsize, *bk_nextsize;  ← 跳表:只串"每种大小的第一块"
}
```

**"同 size 聚成一簇"是什么意思**：桶内若有 3 块 1040、2 块 1024，它们在主链 `fd`/`bk` 上是这样排的（降序，同大小的紧挨成一簇）：

```
   head ↔ [1040#1] ↔ [1040#2] ↔ [1040#3] ↔ [1024#1] ↔ [1024#2] ↔ head
           └──────── 1040 这一簇 ───────┘   └──── 1024 这一簇 ────┘
              ▲ 只有每簇的第一块(1040#1、1024#1)额外参与 nextsize 跳表
              fd_nextsize: 1040#1 ──→ 1024#1 ──→ ...（跳过簇内其余块）
```

**怎么聚的**：插入一块新的 large chunk 时，先沿 `fd_nextsize` 跳表找它该落在哪个大小档：

- 桶里**还没有**这个大小 → 它是新的一簇，插进主链对应位置，并接进 `fd_nextsize`/`bk_nextsize` 跳表，成为这一簇的"代表"；
- 桶里**已有**同样大小 → 不动跳表，直接把新块插到那个已有代表的**后面**（簇内第二个位置）。这样簇内顺序是 FIFO（先来的代表排前），而代表在跳表上的位置不用改——这是省事的小机关。

`fd_nextsize` 跳表的意义：malloc 找某个大小时，顺着跳表一跳一跳地跨过整簇，O(不同大小数) 就能定位，不用在主链上一块块挪。

**动态行为**：

`free` 大块：先和**物理相邻**的空闲块 consolidate（机制同 3.3，看地址不看链表），合并后的块先进 unsorted bin，下一次 `malloc` 遍历 unsorted 时才按上面的规则归类、插进对应 large bin 的正确位置（保持桶内降序 + 维护跳表）。

`malloc` 请求落在 large 范围时：找到覆盖该请求的桶，**最佳适配（best-fit）**——顺着 `fd_nextsize` 跳表找到"≥请求的最小那一簇"，取一块。如果这块比请求大、且**多出来的尾巴够单独成一块**（remainder ≥ MINSIZE = 32），就**切两半**：

```
   找到的空闲块(比如 chunk 4032)，请求只要 chunk 1040：
   ┌─────────────────────────────────────────┐
   │ prev_size│size=4032│  ...... 4032 ......  │
   └─────────────────────────────────────────┘
                    切：前 1040 给你，后 2992 留下
   ┌──────────────┬──────────────────────────┐
   │ size=1040    │ size=2992(remainder)      │
   │ 返回给用户  →  │ 改写头部，丢回 unsorted bin │
   └──────────────┴──────────────────────────┘
```

切分就是改 chunk 头：把原块 `size` 改成请求的 1040、置好 `PREV_INUSE`，返回它的用户指针；在 1040 字节之后的位置**新造一个 chunk 头**，`size` 填剩余的 2992，把这个 remainder 头插进 unsorted bin（所以前面 unsorted 那张图里才有"切剩余"的块）。要是剩下的尾巴不够 32 字节，就不切了，整块给你（宁可让你多用一点，也不制造装不下任何东西的碎渣）。

best-fit + nextsize 跳表，比 small bin 的 FIFO 多了"找最接近的大小"这一步，但能减少碎片——大块通常生命周期长、大小各异，精确匹配比快速匹配更划算。

### 3.6 top chunk 和 mmap 直通路

遍历完 unsorted / small / large 还没找到，最后的兜底是 **top chunk**——也叫 wilderness，是堆顶紧靠 program break 的那块尚未分割的大块，`malloc_state.top` 字段指着它。从它低地址端切一块给请求，剩下的那截还是一整块连续空间，`top` 指针顺势上移指向它，它就成了新的 top chunk。要是 top chunk 本身都不够大、装不下这次请求，就调 `brk` 把 program break 往高地址推一截，把 top chunk 撑大后再切。

```
        低地址
   ┌──────────────────────┐
   │  chunk（已分配）       │
   ├──────────────────────┤
   │  chunk（空闲，在bin里）│
   ├──────────────────────┤
   │  chunk（已分配）       │
   ├──────────────────────┤ ◀─── malloc_state.top
   │                      │      （指向这块的起始）
   │   top chunk          │
   │   （未切割的荒地）     │   malloc 从这块的低地址端
   │                      │   切一块给你，top 起始往高地址挪、荒地变小
   │                      │
   ├──────────────────────┤ ◀─── program break
   │   （未映射，要扩堆     │      （堆的当前上沿，brk 系统调用推动）
   │     就往高地址推）     │
        高地址
```

两个指针指的位置要分清：`malloc_state.top` 指向 top chunk 的**起始**（堆里已用区之后那块荒地的开头）；`program break` 是整个堆的**高地址上沿**。top chunk 就夹在这两者之间。malloc 切走一块时，从 top 的低地址端切、`top` 指针往高地址挪、荒地变小；荒地不够了，就 `brk` 把 program break 往高地址推、把 top 撑大。

另一条路是 **mmap 直通**：请求 ≥ 128 KB（`M_MMAP_THRESHOLD`），直接绕过所有 arena 和 bin，`mmap` 一块独立区域返回给你。`free` 时对应地 `munmap`，完全绕过池子，不进任何 bin。

用数字感受一下这条分水岭：一块要 120000 字节（不到 128KB）、一块要 131072 字节（正好 128KB），用 `mallinfo2().hblks`（当前 mmap 区块数）和地址高位看它俩去了哪：

<a id="code-a_mmap"></a>

```c
// a_mmap.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <malloc.h>

int main(void) {
    void *small = malloc(120000);          // < 128KB，走堆
    struct mallinfo2 m1 = mallinfo2();
    void *big   = malloc(131072);          // = 128KB，转 mmap
    struct mallinfo2 m2 = mallinfo2();

    printf("malloc(120000) = %p   此时 mmap 区块数 hblks = %zu\n", small, m1.hblks);
    printf("malloc(131072) = %p   此时 mmap 区块数 hblks = %zu\n", big,   m2.hblks);
    printf("地址高位: 小块=0x%lx...  大块=0x%lx...\n",
           ((unsigned long)small) >> 28, ((unsigned long)big) >> 28);
    return 0;
}
```

```bash
gcc -O0 -o a_mmap a_mmap.c && ./a_mmap
```

真实输出：

```
malloc(120000) = 0x4052a0          此时 mmap 区块数 hblks = 0
malloc(131072) = 0x7fffff5bc010    此时 mmap 区块数 hblks = 1
地址高位: 小块=0x0...  大块=0x7ffff...
```

120000 字节（< 128 KB）在低地址堆区，`hblks=0`——走的 arena/bin；131072 字节（= 128 KB）跳到高地址 `0x7fffff...`，`hblks` 从 0 变成 1——单独 mmap 出来的，和整个 bin 体系毫无关系。

---

## 四、把它串成一条线：一次 malloc 走遍这些 bin

上面三章是静态结构。现在走一遍完整的动态查找链，把 `malloc_state` 的字段和查找顺序对应起来。

### 4.1 malloc 的查找链

下面以 `malloc(64)`（chunk 80）为主线，越往后的步骤覆盖越大的请求。顺序和 glibc 2.36 的 `_int_malloc` 一致：

```
① 算出 chunk_size = 80，桶下标 = (80-32)/16 = 3

② 先查 tcache[3]（线程私有，不加锁）
   ├─ 非空 → 头摘，直接返回          ← 最快路径，不碰 arena
   └─ 空   → 进 _int_malloc，获取 arena.mutex（加锁）

③ 查 fastbinsY[3]（chunk 80 ≤ 128，在 fastbin 范围）
   ├─ 非空 → 头摘一块，
   │         顺手把该档 fastbin 剩余块批量填进 tcache[3]（填满 7 个为止）
   │         返回
   └─ 空   → 往下

④ chunk 在 small 范围 → 查 small bin[3]
   ├─ 非空 → FIFO 摘尾，同样把同档剩余块批量填进 tcache，返回
   └─ 空   → 往下
   （若是 large 请求：跳过这步，先调 malloc_consolidate
     把所有 fastbin 合并、倒进 unsorted bin，再往下）

⑤ 遍历 unsorted bin（bins[1]），逐块处理：
   ├─ 正好精确匹配请求 → 返回它（或先囤进 tcache 再接着遍历）
   ├─ 是 last_remainder 且请求是 small、它够大 → 切一块返回，余料留原位
   └─ 都不是 → 按大小归类，挪进对应 small/large bin，继续遍历
   （遍历上限 10000 次，或 unsorted 排空为止）

⑥ large 请求：在请求自己的 large bin 里 best-fit（沿 nextsize 跳表找
   ≥请求的最小一簇），找到就切一块、余料回 unsorted，返回

⑦ 还没有 → 扫 binmap，找**下一个更大的非空 bin**，
   从中取最小的够用块，切一块给你、余料（remainder）头插回 unsorted bin
   （这就是"没有精确匹配就切大块"，见下方 a_split.c 实测）

⑧ 连更大的 bin 都空 → 从 top chunk 切一块
   ├─ top 够大 → 切割，余下缩小仍是 top，返回
   └─ top 不够 → 有 fastchunk 先 consolidate 重试；否则 sysmalloc：
                 小请求 brk() 往高地址扩堆补 top；≥128KB 请求直接 mmap
```

第 ⑦ 步"没有精确匹配就切一块更大的空闲块"，跑出来看：

<a id="code-a_split"></a>

```c
// a_split.c
#include <stdio.h>
#include <stdlib.h>

static size_t chunksize(void *p){ return (*((size_t*)p - 1)) & ~(size_t)0x7; }

int main(void) {
    char *big   = malloc(1200);   // chunk 1216
    char *guard = malloc(100); (void)guard;   // 守卫块，防止 big 并进 top
    printf("big = %p  chunk=%zu\n", big, chunksize(big));
    free(big);                    // big 进 unsorted bin
    char *s1 = malloc(500);       // 要 chunk 512，unsorted 里只有 1216，看是否被切
    char *s2 = malloc(500);
    printf("s1  = %p  chunk=%zu  (==big? %s)\n", s1, chunksize(s1), s1==big?"是":"否");
    printf("s2  = %p  (s2-s1=%ld)\n", s2, (long)(s2-s1));
    return 0;
}
```

```bash
gcc -O0 -o a_split a_split.c && ./a_split
```

真实输出：

```
big = 0x4052a0  chunk=1216
s1  = 0x4052a0  chunk=512  (==big? 是)
s2  = 0x4054a0  (s2-s1=512)
```

要 chunk 512、unsorted 里只有那块 1216，`s1` 拿回了 `big` 的地址、chunk 变成 512——1216 被切成 512（给 s1）+ 704 余料回 unsorted；`s2` 紧跟在 `s1` 后 512 字节处，正是从那块余料里再切的。**没有精确匹配，就切一块更大的发给你。**

### 4.2 free 的归位链

`free` 是查找链的镜像，顺序同 glibc 2.36 的 `_int_free`：

```
free(p)：

① 算 chunk_size（从 p-8 读 size 字段，清掉低 3 位标志）

② tcache 对应桶未满（< 7 块）
   → 先查 key 字段防 double-free（同一块已在本桶则报错）
   → 头插进 tcache，done（不加锁，最快）

③ chunk ≤ global_max_fast（128）且 tcache 已满
   → 获取 mutex
   → 查该 fastbin 头是否就是 p（防 double-free）
   → 头插进 fastbinsY 对应槽（**不与邻居合并**，留给以后批量处理）

④ chunk > 128（small/large），获取 mutex 后做 consolidate：
   → 看物理前一块（PREV_INUSE=0 即空闲）→ 空闲就 unlink、向前合并
   → 看物理后一块：
       ├─ 后一块是 top → 整块**并进 top**（堆顶荒地长大）
       └─ 后一块不是 top → 空闲就 unlink 向后合并，
                            合并后的块**头插进 unsorted bin**
   → 若合并后这块 ≥ 64KB（FASTBIN_CONSOLIDATION_THRESHOLD）：
       触发 malloc_consolidate 清空所有 fastbin；若 top 攒得够大，
       还会 systrim/heap_trim 把堆顶内存**真正还给内核**（RSS 下降）

⑤ mmap 出来的大块 → munmap，直接整块还给内核
```

> 第 ④ 步最后那条解释了上一篇的悬念"free 之后内存（RSS）通常不降"：平时 free 只是把块还进 bin、攥在 libc 池子里；只有 free 的块够大、把堆顶 top 撑过收缩阈值时，才会 `systrim` 真还给内核。所以"free 不还内存"是常态，"还"是特例。

### 4.3 完整全景图

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │                              进程                                      │
   │                                                                        │
   │   线程 A             线程 B             线程 C    ...                   │
   │  ┌──────────┐       ┌──────────┐       ┌──────────┐                    │
   │  │  tcache  │       │  tcache  │       │  tcache  │ ← 线程私有，免锁    │
   │  │ entries  │       │ entries  │       │ entries  │   64 桶单链表       │
   │  │ counts   │       │ counts   │       │ counts   │   每桶 ≤ 7 块       │
   │  └────┬─────┘       └────┬─────┘       └────┬─────┘                    │
   │       │  （tcache 独立于 arena，挂在线程上；线程按负载分到不同 arena）  │
   │       ▼                  ▼                   ▼                         │
   │  ┌──────────────────────────────────────────────────────────────┐    │
   │  │  arena 0（main_arena）        管 brk 堆（低地址 0x40xxxx）     │    │
   │  │  mutex          ← 锁整个账本(fastbin+bins+top)，多线程瓶颈     │    │
   │  │  fastbinsY[10]  ← chunk≤128，10 档，LIFO，要锁                 │    │
   │  │  bins[1]        ← unsorted，新 free 块中转                     │    │
   │  │  bins[2..63]    ← small，精确 16B 一档，FIFO                   │    │
   │  │  bins[64..126]  ← large，对数分档，best-fit                    │    │
   │  │  top            ← 兜底，从堆顶 wilderness 切                   │    │
   │  │  next ──┐                                                     │    │
   │  └─────────│─────────────────────────────────────────────────────┘    │
   │            ▼                                                           │
   │  ┌──────────────────────────────────────────────────────────────┐    │
   │  │  arena 1（线程 arena）        管 mmap 堆（高地址 0x7fxxxx）    │    │
   │  │  mutex / fastbinsY[10] / bins[1..126] / top  （结构同 arena 0）│    │
   │  │  next ──┐                                                     │    │
   │  └─────────│─────────────────────────────────────────────────────┘    │
   │            ▼                                                           │
   │        ...最多 8×nproc 个 arena...                                     │
   │            │                                                          │
   │            └──────── next 最终绕回 main_arena（环形链表）             │
   │                                                                        │
   │   >128KB 的请求：直接 mmap，绕过所有 arena 和 bin ────────→ 内核       │
   └──────────────────────────────────────────────────────────────────────┘
```

---

## 五、回头看开篇两个问题

**Q1：主/子线程 malloc 地址差了 128 GB，为什么？**

主线程走 main_arena，main_arena 管的是 `brk` 堆区（低地址 `0x40xxxx`）；子线程触发新建线程 arena，线程 arena 的堆是 `mmap` 出来的（高地址 `0x7fffff...`）。两个 arena 各有各的堆，地址天然不连续。

**Q2：多线程 malloc 为什么慢、怎么会慢 3.3 倍？**

tcache 操作线程私有、免锁——再多线程也不慢。一旦 tcache 不命中，就要去 arena 的 fastbin 或更高层，必须持有 `arena.mutex`。当多个线程被迫共用同一个 arena（线程数 > 8×nproc，或者强制 `MALLOC_ARENA_MAX=1`），它们串行争那一把锁——8 线程实测从 427 M ops/s 降到 130 M ops/s，慢 3.3 倍。

arena 扩张上限（8×nproc）就是为了平衡这两种压力：arena 越多、锁竞争越少，但内存碎片越多、TLB 越紧张；8×nproc 是 glibc 选的折中点。

**第三篇那个钩子兑现**：那把锁是 `malloc_state.mutex`，锁的是整个 arena 账本（fastbin + unsorted/small/large bin + top）；fastbin 不是线程私有，操作它要先持有这把锁；超过 arena 上限后线程共享 arena，这把锁就成了多线程 malloc 的瓶颈。

---

把三、四章串成一句话：**tcache 线程私有免锁最快、fastbin 加锁 LIFO 专攻小块、unsorted 中转过渡、small bin 精确 FIFO、large bin 对数分档 best-fit、top 兜底、超阈值直接 mmap**——一次 `malloc` 就是这 7 个层级按顺序查找，命中越早越快，而多线程竞争的代价，从第二层 fastbin 开始就开始计价了。

---

> 这是"一条代码的冒险之旅"系列的第四篇。上一篇讲 `free` 完再 `malloc` 为什么拿回同一块、以及 use-after-free 的物理根源：《[free 完再 malloc 为什么拿回同一块](https://github.com/xyz2b/article/blob/main/CodeAdventure/free完再malloc为什么拿回同一块.md)》。下一篇把这把 `malloc_state.mutex`（一个普通的 `pthread_mutex_t`）单独拎出来，从用户态 CAS 一路追到内核调度器：《[一个 pthread_mutex_lock 到底锁了什么](https://github.com/xyz2b/article/blob/main/CodeAdventure/一个pthread_mutex_lock到底锁了什么.md)》。
