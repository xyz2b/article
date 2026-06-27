# 一个 pthread_mutex_lock() 到底锁了什么——从用户态 CAS 到内核调度

---

> **🎯 交互式可视化**：[→ mutex-visualizer.html](https://xyz2b.github.io/article/CodeAdventure/front/mutex-visualizer.html)  
> 两个线程的状态机（running / spinning / sleeping）、锁的 0/1/2 三态、内核等待队列的进出，一步步看 lock 一次到底发生了什么。

---

上一篇《[多线程 malloc 为什么会变慢——arena 到 bins 全景](https://github.com/xyz2b/article/blob/main/CodeAdventure/malloc/多线程malloc为什么会变慢——arena到bins全景.md)》末尾把多线程 malloc 的瓶颈钉死在一把锁上：

> "那把锁是 `malloc_state.mutex`，锁的是整个 arena 账本……超过 arena 上限后线程共享 arena，这把锁就成了多线程 malloc 的瓶颈。"

那把锁就是一个普普通通的 `pthread_mutex_t`。这一篇把它单独拎出来，回答一个问题：`pthread_mutex_lock(&m)` 这一行，到底锁了什么、做了什么？

入口是一个能立刻复现的反差——**同一个 `pthread_mutex_lock` 调用，快的时候 5 纳秒、慢的时候要陷内核睡一觉再被唤醒，差出几个数量级。** 先把这个反差量出来。

<a id="code-intro"></a>

```c
// intro.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>

#define ROUNDS 10000000
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static long shared = 0;

static double now_ms(void) {
    struct timespec t; clock_gettime(CLOCK_MONOTONIC, &t);
    return t.tv_sec * 1000.0 + t.tv_nsec / 1e6;
}

static void *worker(void *arg) {
    for (long i = 0; i < ROUNDS; i++) {
        pthread_mutex_lock(&m);
        shared++;                 // 临界区极短，让锁本身成为主要开销
        pthread_mutex_unlock(&m);
    }
    return NULL;
}

int main(int argc, char **argv) {
    int n = argc > 1 ? atoi(argv[1]) : 1;
    pthread_t t[64];
    double a = now_ms();
    for (int i = 0; i < n; i++) pthread_create(&t[i], NULL, worker, NULL);
    for (int i = 0; i < n; i++) pthread_join(t[i], NULL);
    double b = now_ms();
    long total = (long)n * ROUNDS;
    printf("线程数=%d  总计 %ld 次  耗时 %.0f ms  (%.1f M lock/s)  每次约 %.1f ns\n",
           n, total, b - a, total / (b - a) / 1000.0, (b - a) * 1e6 / total);
    return 0;
}
```

```bash
gcc -O2 -pthread -o intro intro.c
./intro 1
./intro 2
./intro 8
```

真实输出：

```
线程数=1  总计 10000000 次  耗时 46 ms  (219.2 M lock/s)  每次约 4.6 ns
线程数=2  总计 20000000 次  耗时 518 ms  (38.6 M lock/s)  每次约 25.9 ns
线程数=8  总计 80000000 次  耗时 1934 ms  (41.4 M lock/s)  每次约 24.2 ns
```

> 说明同系列前四篇：x86-64 语义 + glibc 2.36（Debian 12 里的 `gcc:13` 容器，glibc 2.36-9+deb12u14），文中代码与输出都是真编真跑。看关系不看绝对数字。涉及内核态观测（GDB、`/proc`、strace）的部分在 `--platform linux/arm64 --cap-add=SYS_PTRACE --security-opt seccomp=unconfined` 的原生 arm64 容器里跑；`pthread_mutex_t` 的字段布局、futex 协议、调度行为在 64 位平台上一致，只有具体地址数值不通用。本机内核没开 `CONFIG_SCHED_DEBUG`，所以 `/proc/<tid>/sched` 里的 EEVDF 单字段（vruntime 等）看不到，调度这段改用 `schedstat`、上下文切换计数、`wchan` 这些能实测的指标坐实。

单线程 **4.6 ns** 一次 lock/unlock——这点时间，连一次函数调用的开销都未必够，更别说陷内核了。可两个线程一争，立刻涨到 **25.9 ns**，而且这还只是平均值；真正抢不到锁的那些次，要陷内核睡到被唤醒，单次能到微秒级。

同一行 `pthread_mutex_lock`，凭什么差这么多？因为它内部根本是两条完全不同的路径：**没人跟你争的时候，它连内核都不进，就是一个原子操作（快路径）；一旦争不过别人，它才陷进内核、把自己挂起来睡觉（慢路径）。** 这一篇就把这两条路径拆开看。

## 一、快路径：没人争的时候，lock 就是一个原子操作

先证明一件反直觉的事：**无竞争时，`pthread_mutex_lock` 根本不进内核。** 怎么证？一个程序只要进内核（执行系统调用），`strace` 就能抓到。我们写一个不创建任何线程、只反复 lock/unlock 一百万次的程序，用 `strace` 数它一共调了几次和锁相关的系统调用（`futex`，下一节细说它是什么）：

<a id="code-solo"></a>

```c
// solo.c
#include <pthread.h>
#include <stdio.h>
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
int main(void) {
    long s = 0;
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&m);
        s++;
        pthread_mutex_unlock(&m);
    }
    printf("s=%ld\n", s);
    return 0;
}
```

```bash
gcc -O0 -pthread -o solo solo.c
strace -e trace=futex ./solo
```

真实输出：

```
s=1000000
+++ exited with 0 +++
```

`strace -e trace=futex` 会把每一次 `futex` 系统调用打印成一行。这里**一行都没有**——100 万次 lock/unlock，0 次进内核。整个过程纯粹在用户态完成。

那它在用户态做了什么？`pthread_mutex_t` 里有个整数字段 `__lock`，lock 就是把它从 0 改成 1，unlock 就是改回 0。关键是这个"改"必须是**原子**的：用一条 CPU 指令（x86 上是 `lock cmpxchg`，ARM 上是 `ldaxr/stlxr` 一对），一次性完成"看它是不是 0、是 0 就改成 1"这个**比较并交换**（compare-and-swap，CAS）动作，中间不可能被别的线程插进来。

- **lock**：CAS 把 `__lock` 从 `0` 换成 `1`。换成功 → 拿到锁，往下跑。
- **unlock**：把 `__lock` 写回 `0`。

没有竞争时，每次 CAS 都一击即中，全程不碰内核——这就是那 4.6 ns 的来历：本质上就是一条原子指令的开销。

用 GDB 把这个字段挖出来看。`pthread_mutex_t` 是个联合体，真正的状态在 `__data` 里：

```
(gdb) print m.__data.__lock     → 0    （没人持有）
(gdb) print m.__data.__owner    → 0    （没有属主）
(gdb) print m.__data.__count    → 0    （重入计数，非递归锁恒为0）
(gdb) print m.__data.__kind     → 0    （锁类型：0=默认的 TIMED_NP，非递归、不自旋）
```

初始全是 0。下一节我们让它动起来，看这个 `__lock` 从 0 变 1、再变 2 的全过程。

## 二、慢路径：争不过，就陷内核睡觉

快路径成立有个前提：CAS 一次就成功。可如果另一个线程已经把 `__lock` 改成 1 了呢？这时你的 CAS 失败——锁被人占着。怎么办？

有两个朴素的选择，glibc 都没有简单采用：

1. **空转重试（busy-wait / 自旋）**：写个 `while (CAS 失败) ;` 死循环不停试。问题是持锁线程可能要占用锁很久，你这一圈圈空转纯属浪费 CPU——而且你在烧 CPU，持锁线程反而可能抢不到 CPU 去干完活、释放锁，越等越久。
2. **直接陷内核等**：每次拿不到锁就做个系统调用让内核挂起自己。问题是系统调用本身就有开销，要是锁马上就能拿到（持锁线程下一纳秒就放），为这点等待陷一趟内核太亏。

glibc 默认的 mutex 走的是一条更聪明的路，靠的是一个叫 **futex**（fast userspace mutex，快速用户态互斥）的内核机制。futex 的设计哲学一句话概括：**无竞争时别来烦我（内核），有竞争了再喊我。** 它比方案 2"直接陷内核"聪明在两点：**unlock 只在有人等时才叫醒（避免无谓的 WAKE）**、**lock 陷内核后、挂起前还会再看一眼锁是不是真的还被占（抓住"锁刚被放"的窗口、避免无谓的睡眠）**。

具体说，`__lock` 这个整数同时被用户态和内核认得（它是一个"futex 字"）：

- **用户态先用 CAS 试**——这就是第一节的快路径，成功就完事，内核根本不知情。
- **CAS 失败了**，才发起 `futex(FUTEX_WAIT)` 系统调用，把"我要在 `__lock` 这个地址上睡觉，除非它的值变了"这件事告诉内核——注意这里带了个预期值（比如 2），内核进来后会**再检查一次**，值变了就不睡、直接返回让你重试（下面 2.1 节细讲）。
- 持锁线程 unlock 时，**只有发现 `__lock` 之前是 2（"有人在等"标记）**，才发起 `futex(FUTEX_WAKE)`，让内核把睡着的线程叫醒——没人等就不进内核。

先把"有竞争就进内核"这件事量出来。还是 `strace`，这次让两个线程真争同一把锁，临界区里故意磨蹭一会，逼出真实竞争：

<a id="code-syscall"></a>

```c
// syscall.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static long shared = 0;
static int ROUNDS;
static volatile int hold;     // 让持锁线程在临界区里磨蹭，逼出真实竞争

static void *worker(void *arg) {
    for (long i = 0; i < ROUNDS; i++) {
        pthread_mutex_lock(&m);
        for (volatile int k = 0; k < hold; k++) {}   // 占住临界区
        shared++;
        pthread_mutex_unlock(&m);
    }
    return NULL;
}

int main(int argc, char **argv) {
    int n  = atoi(argv[1]);
    ROUNDS = atoi(argv[2]);
    hold   = atoi(argv[3]);
    pthread_t t[64];
    for (int i = 0; i < n; i++) pthread_create(&t[i], NULL, worker, NULL);
    for (int i = 0; i < n; i++) pthread_join(t[i], NULL);
    printf("线程数=%d 每线程%d次 hold=%d  shared=%ld\n", n, ROUNDS, hold, shared);
    return 0;
}
```

```bash
gcc -O0 -pthread -o syscall syscall.c
# 单线程无竞争，10万次
strace -f -e trace=futex -c ./syscall 1 100000 0
# 两线程争锁，每线程5000次，临界区占住
strace -f -e trace=futex -c ./syscall 2 5000 2000
```

`-c` 让 strace 汇总每种系统调用的次数。真实输出（节选）：

```
=== 单线程无竞争 ===
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         1           futex      ← 仅 1 次，还是 join 触发的

=== 两线程争锁 ===
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.054707          12      4404      2166 futex      ← 4404 次！
```

无竞争时那唯一的 1 次 `futex`，是 `pthread_join` 等子线程退出用的，不是 lock/unlock（上一节 solo.c 不 join，连这 1 次都没有）。而两线程一争，`futex` 飙到 **4404 次**——这就是慢路径的代价：每一次抢不到锁，都要陷一趟内核。

那 `errors` 列的 2166 次"错误"是什么？不是 bug——是 futex 的正常工作方式。线程发起 `FUTEX_WAIT` 时会带上"我以为 `__lock` 现在是 2"这个预期值，内核临挂起前再核对一次：要是这一刻值已经变了（锁在你陷内核的路上刚好被放了），内核就不挂起、直接返回 `EAGAIN` 让它重试。**这就是 futex 比"拿不到锁就直接睡"更聪明的地方：抓住"锁刚被放"的时间窗口，避免无谓的睡眠。** 这些 `EAGAIN` 就记成了 errors——不是失败，恰恰是这个"挂起前最后校验"机制在起作用。

### 2.1 `__lock` 的三个状态：0 → 1 → 2

`__lock` 不只是简单的 0/1 开关——glibc 用三个值编码了"有没有人在等"这个信息，这样 unlock 才知道要不要叫醒别人：

- **0**：空闲，没人持有。
- **1**：有人持有，但**没人在等**（无竞争）。unlock 时不用进内核，直接写回 0 就完事。
- **2**：有人持有，**且有人在等**（有竞争）。unlock 时必须调 `FUTEX_WAKE` 叫醒等待者。

lock 的完整逻辑：

```
① CAS(__lock, 0 → 1)：尝试从 0 换成 1
   成功 → 拿到锁，返回
   失败 → 往下

② 自旋几次 CAS(__lock, 0 → 1)（只在配置了 adaptive 自旋的锁类型才有这步，默认非 adaptive）

③ CAS(__lock, 1 → 2)：把"有人在等"这个标记立起来
   （如果已经是 2，这步是空操作）

④ futex(__lock, FUTEX_WAIT, 2)：陷内核，告诉内核"我要在这个地址上睡，除非它的值不是 2"
   内核进来后会再看一眼 __lock 是不是真的还等于 2：
     - 等于 2 → 挂起，把线程状态改成 TASK_INTERRUPTIBLE，放到等待队列上睡觉
     - 不等于 2（锁在你陷内核的路上刚被放了）→ 不挂起，立刻返回 EAGAIN
   如果被挂起，醒来后重新从 ① 开始争
```

unlock 的逻辑镜像对称：

```
① __lock 减 1（原子操作，x86 上是 lock xadd -1）
   结果是 0 → 说明没人等，done（这就是无竞争 unlock 为什么快）
   结果非 0 → 说明有人等（__lock 之前是 2，减完变 1），往下

② 把 __lock 写成 0（先释放持有权）

③ futex(__lock, FUTEX_WAKE, 1)：叫醒一个等待者
```

这套状态机让"无竞争"的 lock/unlock 完全不碰内核（0→1→0 纯 CAS），只有真的有人在等时才调 `FUTEX_WAKE`。

用 GDB 抓三个时刻的 `__lock`，坐实这个 0→1→2 的变化。写一个会产生竞争的程序：主线程先拿锁，子线程去抢、抢不到就卡住，我们用断点在这三个时刻暂停，dump `__lock` 的值：

<a id="code-mstate"></a>

```c
// mstate.c
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_barrier_t b;

void *contender(void *arg) {
    pthread_barrier_wait(&b);   // 和主线程对齐：此刻主线程已持锁
    pthread_mutex_lock(&m);     // 抢不到 → 陷入 futex 睡眠
    printf("contender 拿到锁了\n");
    pthread_mutex_unlock(&m);
    return NULL;
}

int main(void) {
    pthread_barrier_init(&b, NULL, 2);
    // 时刻 A：还没人 lock
    pthread_t t;
    pthread_mutex_lock(&m);     // 主线程拿到锁
    // 时刻 B：主线程持锁，无人竞争
    pthread_create(&t, NULL, contender, NULL);
    pthread_barrier_wait(&b);   // 放竞争者去抢
    sleep(2);                   // 给竞争者时间陷入睡眠
    // 时刻 C：竞争者正卡在 lock 里睡着
    pthread_mutex_unlock(&m);   // 释放 → 唤醒竞争者
    pthread_join(t, NULL);
    return 0;
}
```

用 GDB 在三个位置打断点、分别 print `m.__data.__lock` 和 `__owner`：

```bash
gcc -O0 -g -pthread -o mstate mstate.c
gdb -q ./mstate
(gdb) break mstate.c:24      # 时刻 A
(gdb) run
```

真实输出：

```
=== 时刻 A：还没人 lock ===
m.__data.__lock  = 0
m.__data.__owner = 0

=== 时刻 B：主线程刚 lock，无竞争 ===
m.__data.__lock  = 1
m.__data.__owner = 400       ← 主线程的内核 TID

=== 时刻 C：竞争者正卡在 lock 睡着 ===
m.__data.__lock  = 2         ← "有人在等"标记
m.__data.__owner = 400       ← 主线程仍持有

(gdb) info threads
  Id   Target Id                                Frame 
* 1    Thread 0xfffff7ff6f80 (LWP 400) "mstate" main () at mstate.c:34
  2    Thread 0xfffff7dff1a0 (LWP 402) "mstate" futex_wait (...) at ../sysdeps/nptl/futex-internal.h:146
```

Thread 2 停在 `futex_wait`——这就是慢路径的现场。`__lock` 从 0 变 1（无竞争持有）、再变 2（有竞争，竞争者已在内核睡着）。整个状态机清晰可见。

### 2.2 FUTEX_WAIT 和 FUTEX_WAKE 的配对

`strace` 能把这两个系统调用的配对关系抓出来，带上耗时（`-T`）：

```bash
strace -f -e trace=futex -T ./mstate
```

真实输出（节选关键的两对）：

```
[pid   417] futex(0x420070, FUTEX_WAIT_PRIVATE, 2, NULL <unfinished ...>
[pid   416] futex(0x420070, FUTEX_WAKE_PRIVATE, 1) = 1 <0.000370>
[pid   417] <... futex resumed>)        = 0 <2.002129>
```

- 竞争者线程（pid 417）发起 `FUTEX_WAIT_PRIVATE`，在 `0x420070`（就是 `m.__data.__lock` 的地址）上睡觉，等的预期值是 `2`。
- 它阻塞了 **2.002 秒**（`sleep(2)` 那段时间）。
- 主线程（pid 416）调 `FUTEX_WAKE_PRIVATE`，唤醒 1 个等待者（返回值 `= 1` 就是"叫醒了 1 个"）。
- 竞争者的 `FUTEX_WAIT` 返回 `0`（被唤醒），继续跑。

整个过程：竞争者进内核→挂在 futex 等待队列上→持锁者 unlock 时叫醒→被放回运行队列等调度器选中→醒来重新争锁。下一章细说内核这边发生了什么。

## 三、内核那边：等待队列、线程状态、调度器

### 3.1 睡着的线程在哪、状态是什么

线程调 `FUTEX_WAIT` 陷进内核后，内核做了什么？

1. **检查 `__lock` 的当前值是否仍等于传进来的预期值 2**——不等于就立刻返回 `EAGAIN`（这就是 2.2 里说的"挂起前最后校验"），避免明明锁已经释放了还傻乎乎地睡。
2. 等于 → 把这个线程的状态改成 **`TASK_INTERRUPTIBLE`**（可中断睡眠），从 CPU 的运行队列上摘下来。
3. 把它挂到 futex **等待队列**（wait queue）上——内核用一个哈希表管着所有 futex 地址的等待队列，`__lock` 的地址经过哈希函数映射到某个桶（bucket），线程就挂在那个桶的链表里。
4. 让出 CPU，调度器挑另一个能跑的线程去跑。

线程醒来的条件：有人调 `FUTEX_WAKE` 唤醒它，或者被信号打断（所以叫 INTERRUPTIBLE）。

我们能从 `/proc` 文件系统看到这些状态。写个程序让竞争者长时间卡在 lock，给我们 3 秒观测窗口：

<a id="code-snapshot"></a>

```c
// snapshot.c
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/syscall.h>

pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_barrier_t b;

void *waiter(void *arg) {
    pthread_barrier_wait(&b);
    printf("竞争者（tid=%ld）去抢锁...\n", syscall(SYS_gettid));
    pthread_mutex_lock(&m);     // 卡这儿
    printf("拿到了\n");
    pthread_mutex_unlock(&m);
    return NULL;
}

int main(void) {
    pthread_barrier_init(&b, NULL, 2);
    pthread_mutex_lock(&m);
    printf("主线程（tid=%ld）已持锁\n", syscall(SYS_gettid));
    pthread_t t;
    pthread_create(&t, NULL, waiter, NULL);
    pthread_barrier_wait(&b);
    sleep(3);                   // 给 3 秒观测窗口
    printf("释放锁\n");
    pthread_mutex_unlock(&m);
    pthread_join(t, NULL);
    return 0;
}
```

启动后在 3 秒内读 `/proc/<pid>/task/<tid>/status` 和 `wchan`：

```bash
gcc -O0 -pthread -o snapshot snapshot.c
./snapshot &
PID=$!
sleep 0.5
for tid in $(ls /proc/$PID/task/); do
    grep "^State:" /proc/$PID/task/$tid/status
    echo "wchan=$(cat /proc/$PID/task/$tid/wchan)"
done
```

真实输出：

```
主线程（tid=430）已持锁
竞争者（tid=432）去抢锁...

tid=430  State: S (sleeping)  wchan=hrtimer_nanosleep   ← 主线程在 sleep(3)
tid=432  State: S (sleeping)  wchan=futex_wait_queue    ← 竞争者在 futex 等待队列！
```

`wchan`（wait channel）是内核留的调试接口，显示线程当前阻塞在哪个内核函数里。`futex_wait_queue` 就是 futex 子系统里"把线程挂到等待队列"那个函数的名字——**这块内存级证据，坐实了竞争者真的被挂在 futex 的等待队列上睡着了。**


再看另一个角度：CPU 时间。一个线程拿着锁干活、另一个线程等锁，谁在烧 CPU？

<a id="code-cpu"></a>

```c
// cpu.c
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_barrier_t b;
volatile long sink;

void *waiter(void *arg) {
    pthread_barrier_wait(&b);
    pthread_mutex_lock(&m);    // 抢不到，睡 3 秒
    pthread_mutex_unlock(&m);
    return NULL;
}

int main(void) {
    pthread_barrier_init(&b, NULL, 2);
    pthread_mutex_lock(&m);
    pthread_t t;
    pthread_create(&t, NULL, waiter, NULL);
    pthread_barrier_wait(&b);
    // 持锁线程空转 3 秒（busy）
    for (long i = 0; i < 3000000000L; i++) sink += i;
    pthread_mutex_unlock(&m);
    pthread_join(t, NULL);
    return 0;
}
```

启动后读两个线程的 `/proc/<pid>/task/<tid>/stat` 第 14、15 字段（utime、stime，累计 CPU 时间）：

```bash
gcc -O0 -pthread -o cpu cpu.c
./cpu &
PID=$!
sleep 1.5
for tid in $(ls /proc/$PID/task/); do
    read -a F < /proc/$PID/task/$tid/stat
    echo "tid=$tid  state=${F[2]}  utime=${F[13]}  stime=${F[14]}"
done
```

真实输出：

```
tid=454  state=R  utime=150  stime=0    ← 持锁线程：状态 R（running），烧了 150 ticks
tid=456  state=S  utime=0    stime=0    ← 等锁线程：状态 S（sleeping），CPU 时间 0
```

等锁的线程 **utime=0**——它不是在 while 循环里空转等，而是真的被挂起、完全不占 CPU。这就是 futex 的 "fast" 体现在哪：无竞争时快（纯用户态），有竞争时也"快"（不浪费 CPU 空转，让别的线程尽快干完活放锁）。

### 3.2 unlock 唤醒：从内核等待队列到调度器运行队列

持锁线程 unlock 时，如果 `__lock` 之前是 2（有人等），它会调 `futex(FUTEX_WAKE)`。内核收到后：

1. 根据 `__lock` 地址哈希找到对应的 futex 等待队列。
2. 从队列里摘一个（或多个，看参数）睡着的线程。
3. 调用内核调度器的 **`try_to_wake_up`** 函数：把线程状态从 `TASK_INTERRUPTIBLE` 改成 `TASK_RUNNING`，放回 CPU 的**运行队列**（runqueue）。
4. 运行队列里可能还有别的能跑的线程在排队——调度器根据优先级、公平性策略（Linux 6.6 起默认是 **EEVDF** 调度器）挑一个，给它 CPU。

注意："被唤醒" ≠ "立刻跑"。它只是从"睡着"（不在任何运行队列）变成"可以跑"（在运行队列里排队），什么时候真正上 CPU 跑，得等调度器选它。

### 3.3 调度延迟：被唤醒后还要排队

我们能从 `/proc/<pid>/task/<tid>/schedstat` 看到每个线程**在运行队列里等 CPU 的累计时间**（第二个数字，单位纳秒）。让 4 个线程争一把锁，跑完后各自读自己的 `schedstat`：

<a id="code-sched"></a>

```c
// sched.c
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <sys/syscall.h>

pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
static int ROUNDS = 3000;

void *worker(void *arg) {
    long tid = syscall(SYS_gettid);
    for (int i = 0; i < ROUNDS; i++) {
        pthread_mutex_lock(&m);
        for (volatile int k = 0; k < 3000; k++) {}
        pthread_mutex_unlock(&m);
        for (volatile int k = 0; k < 500; k++) {}  // 临界区外也干点活，制造交替
    }
    char path[64];
    snprintf(path, 64, "/proc/self/task/%ld/schedstat", tid);
    FILE *f = fopen(path, "r");
    char buf[128];
    fgets(buf, 128, f);
    fclose(f);
    printf("线程 tid=%ld  schedstat = %s", tid, buf);
    return NULL;
}

int main(void) {
    pthread_t t[4];
    for (int i = 0; i < 4; i++) pthread_create(&t[i], NULL, worker, NULL);
    for (int i = 0; i < 4; i++) pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -O0 -pthread -o sched sched.c
./sched
```

真实输出（`schedstat` 三个数：在 CPU 上跑的纳秒 / 在运行队列里等的纳秒 / 被调度的次数）：

```
线程 tid=487  schedstat = 15771547 1522166 969
线程 tid=490  schedstat = 16728386 2175790 992
线程 tid=488  schedstat = 18028338 1975958 1045
线程 tid=489  schedstat = 17450168 2216665 1077
```

每个线程在 CPU 上跑了约 16-18 毫秒（第一个数 `run_ns`），同时在运行队列里**等**了一段（第二个数 `wait_ns`）——这段就是调度延迟：被 `FUTEX_WAKE` 叫醒后并不立刻上 CPU，要先进运行队列、排队、等调度器选中。`wait_ns` 的绝对值受机器负载和虚拟化层影响波动很大（这台经 QEMU 模拟的机器多次跑下来从几微秒到一两毫秒不等），但它**非零**这件事本身就说明了关键：唤醒不等于运行，中间隔着一次调度。

真正稳定复现的是第三个数——**被调度的次数高达近千次甚至两千次**。每抢锁失败一次，就是"睡下去（一次切换）→ 被唤醒 → 排队 → 选中（又一次切换）"一整轮。这就是"有竞争"的代价：不只是 futex 系统调用本身的开销，还有上下文切换、调度器决策、缓存失效……加起来让每次 lock 从纳秒级拖到了微秒甚至毫秒级。

### 3.4 上下文切换的数量级差异

最后用 `/usr/bin/time -v` 统计整个程序的上下文切换次数，对比无竞争和有竞争：

```bash
# 无竞争（单线程，纯 CAS）
/usr/bin/time -v ./syscall 1 200000 0 2>&1 | grep context
# 有竞争（两线程争锁，临界区占住）
/usr/bin/time -v ./syscall 2 5000 3000 2>&1 | grep context
```

真实输出：

```
无竞争:
  Voluntary context switches: 5
  Involuntary context switches: 0

有竞争:
  Voluntary context switches: 2353
  Involuntary context switches: 0
```

- **Voluntary（自愿）切换**：线程主动让出 CPU（比如 `futex_wait` 睡觉、等 I/O）。
- **Involuntary（非自愿）切换**：时间片用完被调度器强制换下。

无竞争时只有 5 次（主要是启动/退出那点开销），有竞争时飙到 **2353 次**——每次抢不到锁就 `FUTEX_WAIT` 主动让出，这就是 voluntary 的来源。而 involuntary=0，说明不是因为线程太多、时间片不够才慢的，就是锁竞争导致的主动休眠/唤醒。

---

## 四、回头看开篇问题：为什么差几个数量级

**同一个 `pthread_mutex_lock`，凭什么快的 5 纳秒、慢的上微秒？**

因为它内部是两条完全不同的路径：

**快路径（无竞争）**：
1. 一条原子指令 CAS，把 `__lock` 从 0 改成 1。
2. 成功 → 拿到锁，继续跑。
3. 全程用户态，不进内核，不切换上下文，不过调度器。
4. 开销：**~5 ns**，就是一条 `lock cmpxchg`（x86）或 `ldaxr/stlxr`（ARM）的延迟。

**慢路径（有竞争）**：
1. CAS 失败（锁被占）。
2. 把 `__lock` 改成 2（标记"有人在等"）。
3. **陷内核**：`futex(FUTEX_WAIT, 2)` 系统调用。
4. 内核检查 `__lock` 仍是 2 → 把线程状态改成 `TASK_INTERRUPTIBLE`，挂到 futex 等待队列。
5. 让出 CPU，调度器挑别人跑。
6. 持锁线程 unlock → 内核收到 `FUTEX_WAKE` → 把等待线程放回运行队列（状态变 `TASK_RUNNING`）。
7. 排队等调度器选中（`schedstat` 里那 1-2 毫秒的 `wait_ns`）。
8. 被选中，恢复执行，重新争锁。
9. 开销：**两次系统调用**（WAIT + WAKE）+ **两次上下文切换**（睡下去 + 醒过来）+ **调度延迟** ≈ 微秒到毫秒级。

开篇测出的 4.6 ns vs 25.9 ns，25.9 还只是**平均值**——真正走慢路径的那些次，单次能到几微秒。而 `strace` 抓到的那个 `<2.002129>` 秒阻塞时间，就是极端情况：锁被占了 2 秒，等待线程就真睡了 2 秒。

这就是为什么"避免锁竞争"是并发编程的核心优化点：不是因为"锁"本身慢（无竞争时它快得飞起），而是**竞争**把你从 5 纳秒的用户态 CAS 踹到了几微秒的内核态休眠/唤醒/调度循环里。

---

## 五、那把锁到底锁了什么

回到最初的问题：`pthread_mutex_lock(&m)` 到底锁了什么？

**表面上**：它锁的是 `m` 这个 `pthread_mutex_t` 变量——或者更准确地说，锁的是你**约定**的那片临界区代码（`lock` 和 `unlock` 之间那几行）。C 语言不知道你在保护什么，mutex 本身也不知道——它只是提供一个"同一时刻只有一个线程能通过这扇门"的机制，保护什么数据、怎么保护，全靠程序员自己遵守约定。

**实现上**：它锁的是 `pthread_mutex_t.__data.__lock` 这个整数：

- **无竞争时**：纯用户态的原子操作（CAS）把它从 0 改成 1、再改回 0，连内核都不进。
- **有竞争时**：它变成内核和用户态之间的"信号灯"——用户态 CAS 它，内核在它上面挂等待队列、做休眠/唤醒。这个整数的地址（在 futex 术语里叫"futex 字"）既被用户态认得（CAS 操作数），也被内核认得（哈希成等待队列的 key）。

**代价上**：无竞争时它快到可以忽略（5 纳秒），有竞争时它把你拖进内核调度器的泥潭（微秒到毫秒）——这就是为什么上一篇说"8 线程强制共用 1 个 arena，吞吐掉 3.3 倍"：不是锁本身慢，是**争**这个动作把快路径全毁了。

所以"锁了什么"的完整答案是：**它锁的是一个约定、靠的是一个整数、守的是一条快路径、付的是一笔进内核的代价。** 这把锁聪明就聪明在：**没人跟你争的时候，它装作不存在；一旦有人争，它就喊来内核当裁判。**

---

## 六、一张图串起来

```
   线程 A                               线程 B
     │                                    │
     ├─ pthread_mutex_lock(&m)            │
     │   └─ CAS(__lock, 0→1)              │
     │      成功！ __lock=1                │
     ├─ shared++ (临界区)                 │
     │                                    ├─ pthread_mutex_lock(&m)
     │                                    │   └─ CAS(__lock, 0→1)
     │                                    │      失败！(__lock 已是 1)
     │                                    │   └─ CAS(__lock, 1→2)  标记"有人等"
     │                                    │   └─ futex(FUTEX_WAIT, 2)  ← 进内核
     │                                    │       内核：状态→INTERRUPTIBLE
     │                                    │             挂到等待队列
     │                                    │             让出 CPU
     │                                    ▼ 💤 睡着了
     ├─ pthread_mutex_unlock(&m)          │
     │   └─ __lock 减 1 → 结果非 0        │
     │   └─ __lock 写成 0                 │
     │   └─ futex(FUTEX_WAKE, 1)          │
     │       内核：从等待队列摘 B          │
     │             状态→TASK_RUNNING       │
     │             放进运行队列             ▲ 被唤醒，进运行队列排队
     │                                    │ （schedstat 的 wait_ns 在涨）
     ├─ 继续跑别的                         │
     │                                    │ 调度器选中 B
     │                                    ├─ 醒来，重新 CAS
     │                                    │   └─ CAS(__lock, 0→1) 成功
     │                                    ├─ shared++ (临界区)
     │                                    ├─ pthread_mutex_unlock(&m)
     │                                    │   └─ __lock 写回 0
     │                                    │       （没人等，不调 FUTEX_WAKE）
```

---

> 这是"一条代码的冒险之旅"系列的第五篇。上一篇讲多线程 malloc 为什么慢、那把锁是什么：《[多线程 malloc 为什么会变慢——arena 到 bins 全景](https://github.com/xyz2b/article/blob/main/CodeAdventure/malloc/多线程malloc为什么会变慢——arena到bins全景.md)》。本篇把那把锁（`pthread_mutex_t`）单独拎出来，从用户态 CAS 快路径追到了内核调度器——**同一个 lock，无竞争时 5 纳秒纯用户态、有竞争时陷内核睡觉被唤醒排队过调度器，差出几个数量级。** 下一篇待定（可能继续挖并发的坑，也可能换条线索）。
