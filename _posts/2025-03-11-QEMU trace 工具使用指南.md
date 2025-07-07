---
title: QEMU trace 工具
date: 2025-03-11 10:23:00 +0800
categories:
  - QEMU
tags:
  - QEMU
  - trace
  - 调试
---
# QEMU trace工具指南

QEMU 提供了一套强大的追踪机制，通过在代码中插入轻量级的 `trace events` 来记录运行时行为。这些事件可在运行时动态启用/禁用，适用于调试、性能分析和行为观察。

---

## 一、基本概念

### 1.1 主要功能
- **调试**：跟踪特定函数或模块的行为（如磁盘 I/O、事件循环）。
- **性能分析**：测量事件频率或执行时间（如 I/O 操作的延迟）。
- **执行观察**：记录虚拟机的运行轨迹（如指令执行、设备交互）。
- **灵活性**：支持运行时动态启用/禁用事件，无需重新编译 QEMU。

### 1.2 支持的追踪backend

| backend  | 描述                                    |
| -------- | ------------------------------------- |
| `log`    | 输出到标准错误，适合简单调试，开销较大                   |
| `simple` | 写入二进制文件，适合离线分析，低开销                    |
| `ftrace` | Linux 系统专属，写入内核 ftrace 缓冲区            |
| `syslog` | 输出到系统日志（POSIX 系统），便于集成                |
| `ust`    | 使用 LTTng 用户态追踪器，高性能，需安装 `lttng-tools` |
| `dtrace` | 使用 DTrace/SystemTap 探针，生成 `.stp` 文件   |
| `nop`    | 空操作，用于关闭追踪以提升性能                       |

> 默认为 `log`，可通过 `./configure --enable-trace-backend=<backend>` 更改。

---

## 二、快速上手

### 2.1 添加 Trace 点

#### 步骤：
1. 在 C 文件中添加 trace 调用（包含头文件 `trace.h`）：

```c
#include "trace.h"

trace_bdrv_co_pwritev_part(child->bs, offset, bytes, flags);
```

2. 在对应目录的 `trace-events` 文件中定义事件格式：

```c
bdrv_co_pwritev_part(void *bs, int64_t offset, int64_t bytes, int flags) "bs=%p offset=%" PRId64 " bytes=%" PRId64 " flags=0x%x"
```

3. 重新配置并编译 QEMU：

```bash
./configure --enable-trace-backend=simple
make
```

### 2.2 启动 QEMU 并启用 Trace

```bash
qemu-system-x86_64 \
    -m 2048 \
    -hda disk.qcow2 \
    -trace events=/tmp/events \
    -trace-file trace-output
```

- `/tmp/events` 内容示例（可添加多个）：
```text
bdrv_co_pwritev_part
```

### 2.3 查看 Trace 日志

使用 `scripts/simpletrace.py` 解析生成的日志：

```bash
./scripts/simpletrace.py /usr/share/qemu/trace-events-all trace-output > trace.txt
```

#### 示例输出：

```text
bdrv_co_pwritev_part 1.684 pid=24910 bs=0x634ec59de9b0 offset=0x114136000 bytes=0x1000 flags=0x0  
bdrv_co_pwritev_part 3.216 pid=24910 bs=0x634ec59f8130 offset=0xb59fa000 bytes=0x2000 flags=0x8
```

#### 日志参数解释 ：

| 字段名                  | 含义                                    |
| -------------------- | ------------------------------------- |
| bdrv_co_pwritev_part | 跟踪的函数名，表示一次块设备写操作的入口                  |
| 1.684                | 事件发生的时间（单位通常是秒，具体取决于 trace 工具的输出格式）   |
| pid=24910            | 进程ID，表示当前 QEMU 进程的编号                  |
| bs=0x634ec59de9b0    | BlockDriverState 结构体的指针地址，唯一标识一个块设备节点 |
| offset=0x114136000   | 写操作的起始偏移（字节），十六进制                     |
| bytes=0x1000         | 本次写操作的字节数，十六进制（0x1000=4096字节=4KB）     |
| flags=0x0            | 本次写操作的标志位，十六进制（如 0x8 可能代表 FUA 等）      |

---

## 三、关键代码分析

### 3.1 事件定义（`trace-events`）

```c
// block/trace-events
thread_pool_submit(void *pool, void *work, void *opaque) "pool %p work %p opaque %p"
thread_pool_complete(void *pool, void *work, int ret) "pool %p work %p ret %d"
```

### 3.2 自动生成的探针函数（`trace-block.c`）

```c
void trace_thread_pool_submit(void *pool, void *work, void *opaque) {
    if (trace_event_get_state(TRACE_THREAD_POOL_SUBMIT)) {
        trace_event_vprintf(TRACE_THREAD_POOL_SUBMIT, "pool %p work %p opaque %p", pool, work, opaque);
    }
}
```

### 3.3 核心接口（`trace/control.h`）

- `trace_event_get_state(event)`：检查事件是否启用
- `trace_event_enable(event)` / `trace_event_disable(event)`：启用/禁用事件
- `trace_event_iter_init()` / `trace_event_iter_next()`：迭代所有事件

### 3.4 示例：`thread_pool_submit_aio()`

```c
ThreadPool *thread_pool_submit_aio(ThreadPool *pool, ThreadPoolFunc *func,
                                   void *opaque, BlockCompletionFunc *cb, void *cb_opaque) {
    ThreadPoolWork *work = thread_pool_work_new(pool, func, opaque, cb, cb_opaque);
    trace_thread_pool_submit(pool, work, opaque); // 插入 trace 点
    ...
}
```

---

## 四、具体使用命令与场景

### 4.1 准备工作

```bash
echo "thread_pool_submit" > /tmp/events
echo "thread_pool_complete" >> /tmp/events
echo "bdrv_aio_writev" >> /tmp/events
```

### 4.2 启动 QEMU 并启用 Trace

```bash
qemu-system-x86_64 \
    -m 2048 \
    -hda disk.qcow2 \
    -object iothread,id=iothread0 \
    -device virtio-blk,drive=hd0,iothread=iothread0 \
    -trace events=/tmp/events \
    -trace-file trace-output
```

### 4.3 解析 Trace 日志

```bash
./scripts/simpletrace.py /usr/share/qemu/trace-events-all trace-output
```

### 4.4 动态控制 Trace（QMP）

启动 QEMU：

```bash
qemu-system-x86_64 -qmp stdio ...
```

发送命令：

```json
{ "execute": "trace-event-set-state", "arguments": { "name": "thread_pool_submit", "enable": true } }
```

查看状态：

```json
{ "execute": "info trace-events" }
```

---

## 五、结合调试场景的追踪示例

### 场景目标：跟踪 `thread_pool_submit_aio()` 的调用栈和 I/O 流程

#### 步骤：

1. **确认事件定义**
   - `block/trace-events` 中确保有 `bdrv_aio_writev`, `thread_pool_submit`, `thread_pool_complete`

2. **运行 QEMU 并启用 IOThread**

```bash
qemu-system-x86_64 \
    -m 2048 \
    -hda disk.qcow2 \
    -object iothread,id=iothread0 \
    -device virtio-blk,drive=hd0,iothread=iothread0 \
    -trace events=/tmp/events \
    -trace-file trace-output
```

3. **解析日志**

```bash
./scripts/simpletrace.py /usr/share/qemu/trace-events-all trace-output
```

#### 示例输出：

```
bdrv_aio_pwritev 0.001 pid=12345 bs=0x5556abc12345 sector_num=2048 nb_sectors=16 opaque=0x5556def67890
thread_pool_submit 0.002 pid=12345 pool=0x5556ghi45678 work=0x5556jkl90123 opaque=0x5556def67890
thread_pool_complete 0.010 pid=12345 pool=0x5556ghi45678 work=0x5556jkl90123 ret=0
```

#### 分析内容：

- `bdrv_aio_writev`：块设备写请求的发起
- `thread_pool_submit`：任务提交至线程池
- `thread_pool_complete`：任务完成及返回结果

---

## 六、高级用法与注意事项

### 6.1 动态调整追踪（QMP + 通配符）

```json
{ "execute": "trace-event-set-state", "arguments": { "name": "bdrv_aio_*", "enable": true } }
```

### 6.2 性能优化建议

- 高频事件（如 `exec_tb`）可设为默认不启用：

```c
disable exec_tb(void *tb, uintptr_t pc) "tb:%p pc=0x%"PRIxPTR
```

- 使用 `simple` 或 `ftrace` 后端以降低开销

### 6.3 自定义分析脚本（Python）

```python
def process_event(event, args):
    if event.name == "thread_pool_submit":
        print(f"[Submit] Pool={args[0]}, Work={args[1]}, Opaque={args[2]}")
```

### 6.4 注意事项

- 确保 `trace-events-all` 文件与 QEMU 版本一致
- `ftrace` 和 `syslog` 后端受限于平台环境（Linux/POSIX）
- 高频事件可能导致日志过大，建议谨慎选择

---

## 七、总结

QEMU 的追踪系统提供了一种灵活、高效的手段来观察和分析其内部行为。无论是调试复杂问题还是进行性能调优，都可以借助 `trace_events` 和相应的后端工具实现。掌握以下几点将有助于高效使用该系统：

- **事件定义与插桩**：准确理解如何在代码中插入 trace 点
- **后端选择**：根据需求选择合适的后端（如 `simple` 适合离线分析）
- **动态控制**：利用 QMP 实现运行时开关 trace
- **日志解析与可视化**：熟练使用 `simpletrace.py` 或自定义脚本处理数据

--- 

📌 **附录参考文档**：  
🔗 [QEMU tracing documentation](https://github.com/virtualopensystems/qemu-arm/blob/master/docs/tracing.txt)
