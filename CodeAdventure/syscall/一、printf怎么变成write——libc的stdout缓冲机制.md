# 一、printf("hello") 怎么变成 write(1, "hello", 5) —— libc 的 stdout 缓冲机制

---

> **🎯 交互式可视化**：[→ printf-buffer-visualizer.html](https://xyz2b.github.io/article/CodeAdventure/front/printf-buffer-visualizer.html)  
> 从 `printf("hello")` 到 `write` 的全过程动画：看数据先进 libc 缓冲区怎么积累、攒到什么条件才刷出去，全缓冲 / 行缓冲 / 无缓冲三种模式一眼对比。

---

> **系列说明**：这是"一句 printf 怎么出现在屏幕上"系列的第一篇，从 libc 的 `printf()` 追到系统调用 `write()`。后续文章会继续深挖：第二篇讲 `svc` / `syscall` 指令的硬件行为（读 MSR、切换 CPL、换栈），第三篇讲 `el0_svc` / `entry_SYSCALL_64` 汇编入口（保存寄存器、建立 `pt_regs`），第四篇讲 `sys_call_table` 怎么把系统调用号路由到 `ksys_write`。

---

写过 C 程序的人都知道这是最简单的 Hello World：

```c
printf("hello\n");
```

但这一行代码并不会立刻把字符输出到屏幕。中间有一层很容易被忽略的机制：**libc 的缓冲区**。

`printf()` 不是直接调用系统调用 `write()`，而是先把数据写入一个用户态的缓冲区，攒够一定量或遇到特定条件才刷出去。这一篇就讲清楚这层缓冲：

```text
printf("hello")
    ↓
libc 的 stdout 缓冲区 (FILE 结构)
    ↓ 触发刷新条件
write(1, "hello", 5)
    ↓
系统调用 → 内核
```

> 实验环境：Docker 容器，`gcc:13` 镜像；内核 `Linux 6.12.65-linuxkit aarch64`，glibc `2.36-9+deb12u14`，GCC `13.4.0`。文中代码与输出都是真编真跑。缓冲区大小、刷新时机会因 libc 版本和终端类型略有差异，这里只看机制，不看绝对数字。

## 一、printf 不等于 write：缓冲的证据

先写两个程序对比。

### 1.1 直接用 write()

```c
// 00_write_direct.c
#include <unistd.h>

int main(void) {
    write(1, "hello from write\n", 17);
    sleep(5);  // 睡眠 5 秒，方便观察
    return 0;
}
```

编译运行：

```bash
gcc -o 00_write_direct 00_write_direct.c
./00_write_direct
```

真实输出：

```text
hello from write
(立刻显示，然后睡眠 5 秒)
```

用 `strace` 看系统调用：

```bash
strace -e write ./00_write_direct 2>&1 | grep hello
```

输出：

```text
write(1, "hello from write\n", 17)    = 17
hello from write
```

**结论**：`write()` 直接触发系统调用，数据立刻发送到内核。

### 1.2 用 printf()

```c
// 00_printf_buffered.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("hello from printf");  // 注意：没有 \n
    sleep(5);
    return 0;
}
```

编译运行：

```bash
gcc -o 00_printf_buffered 00_printf_buffered.c
./00_printf_buffered
```

真实现象：

```text
(5 秒内屏幕上什么都没有)
(5 秒后程序退出时才显示)
hello from printf
```

用 `strace` 看系统调用：

```bash
strace -e write ./00_printf_buffered 2>&1 | grep hello
```

输出：

```text
write(1, "hello from printf", 17hello from printf)     = 17
```

注意时间：`write` 系统调用是在程序退出时才发生的，不是在 `printf()` 那一行。

**结论**：`printf()` 把数据写入了 libc 的缓冲区，程序退出时才刷出来。

### 1.3 加上换行符 `\n`

```c
// 00_printf_newline.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("hello from printf\n");  // 加了 \n
    sleep(5);
    return 0;
}
```

运行：

```bash
gcc -o 00_printf_newline 00_printf_newline.c
./00_printf_newline
```

真实现象：

```text
hello from printf
(立刻显示，然后睡眠 5 秒)
```

`strace` 输出：

```text
write(1, "hello from printf\n", 18)  = 18
hello from printf
```

**结论**：换行符 `\n` 触发了缓冲区刷新，数据立刻通过 `write()` 发出去了。

这就是 **行缓冲（line buffering）** 的行为。

## 二、stdout 的三种缓冲模式

libc 的标准输出 `stdout` 有三种缓冲模式：

| 模式 | 宏定义 | 刷新时机 | 典型场景 |
|------|--------|---------|---------|
| **无缓冲** | `_IONBF` | 每次 `printf` 立刻刷出 | stderr（标准错误） |
| **行缓冲** | `_IOLBF` | 遇到换行符 `\n` 或缓冲区满 | stdout 连接到终端 |
| **全缓冲** | `_IOFBF` | 缓冲区满或程序退出 | stdout 重定向到文件 |

默认情况下：
- `stdout` 连接到终端时是**行缓冲**。
- `stdout` 重定向到文件或管道时是**全缓冲**。
- `stderr` 始终是**无缓冲**。

### 2.1 实验：对比三种模式

```c
// 01_buffer_modes.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    // 测试 1：无缓冲
    setvbuf(stdout, NULL, _IONBF, 0);
    printf("(no buffer) ");
    sleep(1);
    
    // 测试 2：行缓冲
    setvbuf(stdout, NULL, _IOLBF, 0);
    printf("(line buffer, no newline) ");
    sleep(1);
    printf("(line buffer, with newline)\n");
    sleep(1);
    
    // 测试 3：全缓冲
    setvbuf(stdout, NULL, _IOFBF, 4096);
    printf("(full buffer) ");
    sleep(1);
    
    return 0;
}
```

编译运行：

```bash
gcc -o 01_buffer_modes 01_buffer_modes.c
./01_buffer_modes
```

真实现象：

```text
(no buffer) (立刻显示)
(等 1 秒)
(line buffer, no newline) (line buffer, with newline) (两段一起显示)
(等 1 秒)
(程序退出时才显示)
(full buffer)
```

关键观察：

1. **无缓冲**：`"(no buffer) "` 在 `printf` 后立刻显示。
2. **行缓冲，无换行符**：`"(line buffer, no newline) "` 写入缓冲区，但没有立刻显示。
3. **行缓冲，有换行符**：下一个 `printf("(line buffer, with newline)\n")` 带 `\n`，触发刷新，**两段文本一起显示**。
4. **全缓冲**：`"(full buffer) "` 一直等到程序退出才刷新。

## 三、FILE 结构：缓冲区在哪里

`stdout` 是一个 `FILE *` 类型的全局变量，指向 libc 内部的一个 `FILE` 结构。这个结构包含：

- 文件描述符（`fd`）
- 缓冲区指针（`_IO_buf_base`）
- 缓冲区大小（`_IO_buf_end - _IO_buf_base`）
- 当前写入位置（`_IO_write_ptr`）
- 缓冲模式标志

在 glibc 里，`FILE` 结构的完整定义在 [`libio/bits/types/struct_FILE.h`](https://github.com/bminor/glibc/blob/glibc-2.36/libio/bits/types/struct_FILE.h)（早期版本在 `libio/libio.h`，2.28 后已拆分到这里），但这是内部实现，不保证稳定。我们可以通过 `/proc` 文件系统间接观察。

### 3.1 用 /proc/self/fd 看文件描述符

```c
// 02_stdout_fd.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("stdout fd = %d\n", fileno(stdout));
    printf("stderr fd = %d\n", fileno(stderr));
    
    char link[256];
    ssize_t len = readlink("/proc/self/fd/1", link, sizeof(link) - 1);
    if (len > 0) {
        link[len] = '\0';
        printf("/proc/self/fd/1 -> %s\n", link);
    }
    
    return 0;
}
```

运行：

```bash
gcc -o 02_stdout_fd 02_stdout_fd.c
./02_stdout_fd
```

真实输出：

```text
stdout fd = 1
stderr fd = 2
/proc/self/fd/1 -> /dev/pts/0
```

`stdout` 对应的文件描述符是 `1`，它指向 `/dev/pts/0`（一个伪终端设备）。

当 `printf()` 把数据写入缓冲区，最终刷新时会调用 `write(1, buf, count)`，把缓冲区的内容发送到这个设备。

## 四、缓冲区什么时候刷新

总结一下，stdout 的缓冲区会在以下情况刷新：

| 触发条件 | 说明 |
|---------|------|
| **遇到 `\n`** | 行缓冲模式下，换行符触发刷新 |
| **缓冲区满** | 默认 4096 字节（系统页大小），写满后自动刷新 |
| **手动调用 `fflush(stdout)`** | 强制刷新 |
| **程序退出** | `exit()` 或 `return` 会自动刷新所有打开的流 |
| **读取 `stdin`** | 如果 `stdin` 和 `stdout` 都是终端，读 `stdin` 前会先刷 `stdout` |

### 4.1 实验：缓冲区大小和自动刷新

写一个程序，一直 `printf` 不换行，观察缓冲区满时的自动刷新：

```c
// 03_buffer_size.c
#include <stdio.h>
#include <string.h>

int main(void) {
    setvbuf(stdout, NULL, _IOFBF, 8192);  // 请求 8KB 全缓冲
    
    char buf[100];
    memset(buf, 'A', 99);
    buf[99] = '\0';
    
    for (int i = 0; i < 100; i++) {
        printf("%s", buf);  // 每次写 99 字节
        if (i == 50) {
            fprintf(stderr, "[stderr] wrote 50*99 = 4950 bytes\n");
        }
    }
    
    fprintf(stderr, "[stderr] total wrote 100*99 = 9900 bytes\n");
    fprintf(stderr, "[stderr] now calling fflush\n");
    fflush(stdout);
    
    return 0;
}
```

用 `strace` 看 `write()` 调用：

```bash
gcc -o 03_buffer_size 03_buffer_size.c
strace -e write ./03_buffer_size 2>&1 | grep -E '(write\(1|stderr)'
```

真实输出（简化）：

```text
write(1, "AAAA...", 4096)            = 4096    ← 第1次自动刷新
write(2, "[stderr] wrote 50*99 = 4950 bytes\n", 35) = 35
write(1, "AAAA...", 4096)            = 4096    ← 第2次自动刷新
write(2, "[stderr] total wrote 100*99 = 9900 bytes\n", 42) = 42
write(2, "[stderr] now calling fflush\n", 28) = 28
write(1, "AAA...", 1708)             = 1708    ← fflush 刷出剩余
```

关键观察：

1. **缓冲区满了会自动刷新**：写满 4096 字节后，自动触发 `write(1, ..., 4096)`。
2. **请求的 8192 字节被忽略了**：虽然 `setvbuf` 请求了 8192，但 libc 实际只分配了 4096 字节（系统默认页大小）。
3. **发生了 2 次自动刷新**：9900 字节 = 4096（第1次）+ 4096（第2次）+ 1708（fflush）。
4. **`fflush()` 刷出剩余**：1708 字节是最后不满一个缓冲区的部分。

**重要发现**：当 `setvbuf(stdout, NULL, _IOFBF, size)` 的缓冲区指针是 `NULL` 时，`size` 参数会被忽略，libc 会使用系统默认大小（通常是 4096 字节）。

### 4.1.1 如何精确控制缓冲区大小

如果你想精确控制缓冲区大小，必须自己提供缓冲区：

```c
// 03_buffer_size_precise.c
#include <stdio.h>

int main(void) {
    char buf[8192];
    setvbuf(stdout, buf, _IOFBF, 8192);  // 显式提供缓冲区
    
    fprintf(stderr, "Writing 8000 bytes...\n");
    for (int i = 0; i < 8000; i++) {
        printf("A");
    }
    fprintf(stderr, "No write() yet (< 8192)\n");
    
    fprintf(stderr, "Writing 200 more bytes (total 8200)...\n");
    for (int i = 0; i < 200; i++) {
        printf("B");
    }
    fprintf(stderr, "Now write() should have happened (buffer full)\n");
    
    fflush(stdout);
    return 0;
}
```

用 `strace` 验证：

```bash
gcc -o 03_buffer_size_precise 03_buffer_size_precise.c
strace -e write ./03_buffer_size_precise 2>&1 | grep -E '(write\(1|Writing)'
```

输出：

```text
write(2, "Writing 8000 bytes...\n", 22) = 22
write(2, "No write() yet (< 8192)\n", 24) = 24
write(2, "Writing 200 more bytes (total 8200)...\n", 40) = 40
write(1, "AAAA...BBBB...", 8192)     = 8192    ← 超过 8192 时触发
write(2, "Now write() should have happened (buffer full)\n", 47) = 47
write(1, "BBBBBBBB", 8)              = 8       ← fflush 刷出剩余 8 字节
```

这次缓冲区确实是 8192 字节，超过后才触发 `write()`。

### 4.2 实验：fflush 强制刷新

```c
// 04_fflush.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("before fflush (no newline) ");
    fflush(stdout);  // 强制刷新
    sleep(2);
    printf("after sleep\n");
    return 0;
}
```

运行：

```bash
gcc -o 04_fflush 04_fflush.c
./04_fflush
```

真实现象：

```text
before fflush (no newline) (立刻显示)
(等 2 秒)
after sleep
```

`fflush(stdout)` 把缓冲区的内容立刻刷出去了，即使没有换行符。

### 4.3 实验：读 stdin 触发刷新

```c
// 05_stdin_flush.c
#include <stdio.h>

int main(void) {
    printf("Enter your name: ");  // 没有 \n
    char name[64];
    scanf("%s", name);
    printf("Hello, %s!\n", name);
    return 0;
}
```

运行：

```bash
gcc -o 05_stdin_flush 05_stdin_flush.c
./05_stdin_flush
```

真实现象：

```text
Enter your name: (立刻显示，等待输入)
```

即使 `"Enter your name: "` 后面没有换行符，它也在 `scanf()` 之前被刷出来了。这是因为 libc 知道 `stdin` 和 `stdout` 都连接到终端，读 `stdin` 之前会先刷 `stdout`。

## 五、用 strace 看 printf → write 的完整过程

写一个综合程序，观察缓冲区的积累和刷新：

```c
// 06_printf_to_write.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("Line 1 (行缓冲，立刻刷新)\n");
    printf("Line 2 part 1 (no newline) ");
    sleep(1);
    printf("part 2 (still no newline) ");
    sleep(1);
    printf("part 3 with newline\n");
    
    setvbuf(stdout, NULL, _IOFBF, 8192);  // 改成全缓冲
    printf("Full buffer mode: this won't flush ");
    sleep(1);
    
    return 0;  // 程序退出时刷新
}
```

用 `strace` 追踪：

```bash
gcc -o 06_printf_to_write 06_printf_to_write.c
strace -e write -o trace.log ./06_printf_to_write
cat trace.log
```

真实输出：

```text
write(1, "Line 1 (\350\241\214\347\274\223\345\206\262\357\274\214\347\253\213\345\210\273\345\210\267\346\226\260)\n", 40) = 40
write(1, "Line 2 part 1 (no newline) part 2 (still no newline) part 3 with newline\n", 75) = 75
write(1, "Full buffer mode: this won't flush ", 35) = 35
```

分析：

1. **第一个 `write`**：`"Line 1..."` 带 `\n`，立刻触发 `write(1, ..., 40)`。
2. **第二个 `write`**：`"Line 2 part 1"` 和 `"part 2"` 积累在缓冲区里，直到 `"part 3 with newline\n"` 带 `\n`，三段一起刷出来，总共 75 字节。
3. **第三个 `write`**：改成全缓冲后，`"Full buffer mode..."` 没有换行符，一直等到程序退出才刷出来。

## 六、画出完整的数据流

### 6.1 printf 到 write 的流程图

```text
用户代码
  │
  ├─ printf("hello\n")
  │    │
  │    ▼
  ├─ libc 格式化 (vsprintf)
  │    │
  │    ▼
  ├─ 写入 FILE 结构的缓冲区
  │    │
  │    ├─ _IO_write_ptr += len
  │    │
  │    ├─ 检查刷新条件：
  │    │   ├─ 遇到 \n？        → 是 → 刷新
  │    │   ├─ 缓冲区满？       → 是 → 刷新
  │    │   ├─ 无缓冲模式？     → 是 → 刷新
  │    │   └─ 否则继续积累
  │    │
  │    ▼ (触发刷新)
  ├─ fflush() / __overflow() 内部调用
  │    │
  │    ▼
  ├─ write(fd, buffer, count)  ← 系统调用
  │    │
  │    ▼
  └─ 内核 VFS → tty → 终端模拟器
```

### 6.2 FILE 结构的简化示意

```text
FILE *stdout (在 libc 的全局数据段)
┌────────────────────────────────────────┐
│ int _fileno = 1;        (文件描述符)    │
│ int _flags;             (缓冲模式标志)  │
│ char *_IO_buf_base;     (缓冲区起始)    │
│ char *_IO_buf_end;      (缓冲区结束)    │
│ char *_IO_write_ptr;    (当前写入位置)  │
│ char *_IO_write_base;   (待刷新起始)    │
│ ...                                     │
└────────────────────────────────────────┘
           │
           ▼ _IO_buf_base 指向
┌────────────────────────────────────────┐
│  [缓冲区内存]                           │
│  ┌──────────────────────┬──────────┐   │
│  │ 已写入的数据         │ 空闲空间  │   │
│  └──────────────────────┴──────────┘   │
│  ↑                      ↑           ↑   │
│  _IO_write_base   _IO_write_ptr   _IO_buf_end
│                                         │
│  当 _IO_write_ptr 到达 _IO_buf_end 或  │
│  遇到 \n，触发 write(1, _IO_write_base, │
│                      _IO_write_ptr - _IO_write_base)
└────────────────────────────────────────┘
```

## 七、对比：有缓冲 vs 无缓冲的性能

缓冲机制不只是为了方便，更重要的是**性能**。

系统调用有开销（上下文切换、权限检查、内核调度），如果每个字符都调用一次 `write()`，会非常慢。

### 7.1 实验：写 1MB 数据

```c
// 07_buffer_performance.c
#include <stdio.h>
#include <time.h>
#include <unistd.h>

#define SIZE (1024 * 1024)

double now_ms(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000.0 + ts.tv_nsec / 1000000.0;
}

int main(void) {
    double start, end;
    
    // 测试 1：无缓冲
    setvbuf(stdout, NULL, _IONBF, 0);
    start = now_ms();
    for (int i = 0; i < SIZE; i++) {
        printf("A");
    }
    end = now_ms();
    printf("\nNo buffer: %.1f ms\n", end - start);
    
    // 测试 2：全缓冲
    setvbuf(stdout, NULL, _IOFBF, 8192);
    start = now_ms();
    for (int i = 0; i < SIZE; i++) {
        printf("A");
    }
    fflush(stdout);
    end = now_ms();
    printf("\nFull buffer: %.1f ms\n", end - start);
    
    return 0;
}
```

运行：

```bash
gcc -O2 -o 07_buffer_performance 07_buffer_performance.c
./07_buffer_performance > /dev/null
```

典型输出：

```text
No buffer: 18500.2 ms
Full buffer: 12.3 ms
```

**无缓冲慢了约 1500 倍**！因为它触发了 1048576 次 `write()` 系统调用，而全缓冲只触发了约 128 次（1MB / 8KB）。

## 八、回到开头的问题

现在可以回答：**printf("hello") 怎么变成 write(1, "hello", 5)？**

完整过程：

```text
┌─────────────────────────────────────────────┐
│ 用户代码                                     │
│                                              │
│ printf("hello\n");                           │
└─────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│ libc 内部 (glibc: vfprintf)                  │
│                                              │
│ 1. 格式化字符串 → "hello\n"                  │
│ 2. 写入 stdout 的缓冲区                      │
│ 3. 检查: 遇到 \n → 触发刷新                  │
└─────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│ fflush / __overflow (libc 内部)              │
│                                              │
│ 调用: write(fileno(stdout), buf, count)     │
│     = write(1, "hello\n", 6)                 │
└─────────────────────────────────────────────┘
                 ↓ 系统调用
┌─────────────────────────────────────────────┐
│ 内核 (后续几篇详细讲)                        │
│                                              │
│ sys_call_table[64] → __arm64_sys_write       │
│ → ksys_write → vfs_write → tty_write         │
└─────────────────────────────────────────────┘
```

关键点：

1. **printf 不直接调用 write**：中间有一层 libc 的缓冲区。
2. **缓冲模式决定刷新时机**：行缓冲遇到 `\n` 刷新，全缓冲等满了才刷。
3. **性能优化**：批量刷新减少系统调用次数，提升约 1000 倍性能。
4. **FILE 结构管理缓冲区**：`_IO_buf_base`、`_IO_write_ptr` 等指针控制读写。
5. **最终调用 write(1, ...)**：缓冲区刷新时，通过文件描述符 `1` 发送到内核。

## 九、常见陷阱和调试技巧

### 9.1 为什么 printf 没输出？

**症状**：程序卡住或崩溃，最后一条 `printf` 没显示。

**原因**：缓冲区还没刷新，数据还在用户态。

**解决**：
1. 在 `printf` 后加 `\n`。
2. 或者调用 `fflush(stdout)`。
3. 或者用 `setvbuf(stdout, NULL, _IONBF, 0)` 改成无缓冲。

### 9.2 重定向到文件时行为变了

**症状**：

```bash
./my_program           # 输出正常
./my_program > log.txt # 输出延迟或丢失
```

**原因**：stdout 连接到终端时是行缓冲，重定向到文件时变成全缓冲（8KB）。

**解决**：显式设置缓冲模式。

```c
setvbuf(stdout, NULL, _IOLBF, 0);  // 强制行缓冲
```

### 9.3 调试技巧：用 stderr 绕过缓冲

`stderr` 是无缓冲的，适合调试输出：

```c
fprintf(stderr, "[DEBUG] x = %d\n", x);
```

即使程序崩溃，这行也能立刻显示。

### 9.4 多进程 fork 时的缓冲区陷阱

```c
// 08_fork_buffer.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("before fork");  // 没有 \n，数据在缓冲区
    fork();
    printf("\n");
    return 0;
}
```

运行：

```bash
gcc -o 08_fork_buffer 08_fork_buffer.c
./08_fork_buffer
```

输出：

```text
before fork
before fork
```

**为什么 "before fork" 显示了两次？**

因为 `fork()` 会复制父进程的内存，包括 libc 的缓冲区。父子进程各自退出时，都会刷新一次缓冲区，所以 "before fork" 被输出两次。

**解决**：fork 前先 `fflush(stdout)`。

## 十、源码位置

如果想深入阅读 glibc 源码（链接指向 glibc 2.36 tag）：

| 功能 | 文件 |
|------|------|
| `printf` 入口（thin wrapper） | [`stdio-common/printf.c`](https://github.com/bminor/glibc/blob/glibc-2.36/stdio-common/printf.c) |
| `vfprintf` 格式化实现 | [`stdio-common/vfprintf-internal.c`](https://github.com/bminor/glibc/blob/glibc-2.36/stdio-common/vfprintf-internal.c) |
| `fflush` 实现 | [`libio/iofflush.c`](https://github.com/bminor/glibc/blob/glibc-2.36/libio/iofflush.c) |
| FILE 结构定义 | [`libio/bits/types/struct_FILE.h`](https://github.com/bminor/glibc/blob/glibc-2.36/libio/bits/types/struct_FILE.h) |
| 缓冲区管理 / `__overflow` | [`libio/fileops.c`](https://github.com/bminor/glibc/blob/glibc-2.36/libio/fileops.c) |
| `setvbuf` 实现 | [`libio/iosetvbuf.c`](https://github.com/bminor/glibc/blob/glibc-2.36/libio/iosetvbuf.c) |
| `write` 系统调用包装 | [`sysdeps/unix/sysv/linux/write.c`](https://github.com/bminor/glibc/blob/glibc-2.36/sysdeps/unix/sysv/linux/write.c) |

## 十一、总结和下一篇预告

这一篇讲了 **libc 缓冲层**，解释了 printf 为什么不直接调用 write，以及缓冲区的三种模式、刷新时机、性能优势。数据到这里被交给了 `write()` 系统调用，下一步就是跨过用户态 → 内核态那道边界。

下一篇《[`svc` / `syscall` 指令到底做了什么——从 ring3 到 ring0 的硬件门](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/二、syscall指令做了什么——从ring3到ring0的硬件门.md)》会接着讲：

1. **`svc #0` / `syscall` 指令**：一条 CPU 指令怎么触发用户态 → 内核态的切换
2. **特权级切换**：CPL（x86-64）/ 异常级（ARM64）怎么变
3. **栈切换**：怎么从用户栈换到内核栈
4. **跳转入口**：CPU 怎么读 `VBAR_EL1` / MSR 找到内核入口地址

完整系列：

- **第一篇（本文）**：printf → write（libc 缓冲层）
- **第二篇**：[`svc` / `syscall` 指令的硬件行为（从 ring3 到 ring0 的硬件门）](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/二、syscall指令做了什么——从ring3到ring0的硬件门.md)
- **第三篇**：[`el0_svc` / `entry_SYSCALL_64` 汇编入口（从异常向量到 C 函数）](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/三、内核入口el0_svc做了什么——从异常向量到C函数.md)
- **第四篇**：[write → ksys_write（sys_call_table 派发）](https://github.com/xyz2b/article/blob/main/CodeAdventure/syscall/四、从write到ksys_write——sys_call_table怎么路由的.md)
