---
tags: [python, concurrency, gil, asyncio, backend]
created: 2026-08-04
updated: 2026-08-04
status: 待学习
---

# Python 并发与 asyncio

## 一句话理解

Python 的并发不是“线程越多越快”：GIL 让 CPU 密集型的多线程无法并行，I/O 密集型才适合多线程/协程；选型看任务是吃 CPU 还是等 I/O。

## 核心原理

### GIL（全局解释器锁）

CPython 中同一进程同一时刻只有一个线程在执行 Python 字节码。GIL 在以下时机释放：

- 阻塞 I/O（读文件、网络请求、数据库等待）
- `time.sleep()`、显式释放、`sys.setswitchinterval` 时间片到期
- C 扩展可主动释放 GIL 后并行执行

因此：多线程适合 I/O 密集（等待期间其他线程能跑）；CPU 密集的多线程反而有调度开销，应该用多进程。

### 三种并发手段

| 手段 | 原理 | 适合 | 代价 |
|---|---|---|---|
| 多线程 | 共享内存 + GIL 切换 | I/O 密集、共享状态多 | 线程安全、锁、GIL |
| 多进程 | 每进程独立解释器与 GIL | CPU 密集 | IPC 与序列化开销、状态不共享 |
| asyncio | 单线程事件循环，协程在 await 处让出 | 大量 I/O 等待、连接数高 | 不能跑 CPU 密集；阻塞调用会卡住整个循环 |

### async/await 与事件循环

- 协程是在 `await` 点可挂起/恢复的函数，由事件循环调度成 Task。
- 事件循环在 I/O 就绪时恢复对应协程，单线程内实现高并发，无锁竞争。
- `asyncio.gather()` 并发执行、`asyncio.Semaphore` 限制并发数、`run_in_executor()` 把同步/CPU 任务丢进线程池。

`await` 的执行（2026-08-05 首学）：

```python
import asyncio

async def hello():
    print("start")
    await asyncio.sleep(1)
    print("end")

print("before")        # ①
asyncio.run(hello())   # ②
print("after")         # ③

# 输出顺序：before -> start -> end -> after
```

- `asyncio.run()` 负责创建事件循环、运行协程、结束后关闭。
- `await asyncio.sleep(1)` 的意思是：把控制权交回事件循环，当前协程挂起 1 秒；期间事件循环可以调度别的协程，1 秒后恢复继续。
- `await` 后面的代码必须等 await 完成才执行——这就是「让出控制权」。

`gather` 并发调度（2026-08-05 首学）：

```python
import asyncio

async def task_a():
    print("A1")
    await asyncio.sleep(1)
    print("A2")

async def main():
    await asyncio.gather(task_a(), task_a())

asyncio.run(main())

# 输出：A1 -> A1 -> A2 -> A2
```

`gather` 把两个协程同时放到事件循环排队：任务 1 打印 A1 后在 `await` 让出，任务 2 接着打印 A1 后也让出；1 秒后依次恢复打印 A2。两个 A1 连续出现，说明两个任务在等待期间并发推进；串行执行会是 A1 A2 A1 A2。

## 底层实现

- 原生协程基于生成器协议演进（PEP 342 -> PEP 380 -> PEP 492），本质是 `send()`/`yield` 的挂起恢复机制。
- `uvloop` 用 Cython 重写事件循环，比默认 asyncio 快。
- Task 内部包装 Future，事件循环通过 selector（epoll/kqueue）监听 fd 就绪后回调。
- 关键陷阱：在协程里调用同步阻塞库（`requests`、`time.sleep`），整个事件循环被卡住，其他所有请求排队。

## 常见应用场景

- FastAPI 网关：I/O 密集（上游 HTTP、数据库、认证服务）用 async；加解密/签名等 CPU 密集操作放线程池（`run_in_executor`），避免阻塞事件循环。
- [[Kerberos域接入与用户映射管控]]：管理面用 eventlet 异步 Taskflow 编排加域/退域任务，属于控制面异步任务模型，不是严格分布式事务。
- 脚本与工具：并发请求收口、批量处理限流。

## 容易踩坑的地方

- async 函数里用同步库阻塞：最典型的“接口假死”，要优先排查所有 await 路径上的同步调用。
- 协程不 await 不执行：忘记 `await` 只会产生 RuntimeWarning，任务根本没跑。
- GIL 下 CPU 密集多线程：无加速甚至更慢，面试常用来考选型。
- asyncio 与 eventlet/gevent 混用：两套事件模型互相阻塞，必须整体统一。
- 数据库驱动要用 async 版本（asyncpg/aiomysql），用同步驱动包一层照样阻塞循环。

## 面试高频问题

1. GIL 是什么？为什么存在？如何绕过？
2. CPU 密集和 I/O 密集分别怎么选线程/进程/协程？
3. asyncio 事件循环怎么工作的？一个协程被阻塞会怎样？
4. FastAPI 中 `async def` 与普通 `def` 有什么区别？（FastAPI 会把普通 def 放进线程池）
5. 如何排查事件循环被阻塞？有哪些工具/手段？

## 我的薄弱点

- 2026-08-05 首学：GIL 概念与存在原因、CPU/I-O 密集选型、`await` 让出机制此前均未接触；已首学，待确认练习巩固。
- 2026-08-05 `gather` 并发调度首次确认练习未答出；已用「单线程柜台 + 让出」模型讲解：`gather` 同时排队、`await` 让出、等待结束后恢复，输出 A1 A1 A2 A2。函数名回忆再次失败，已用记忆钩子巩固：**gather = 召集，把多个协程召集到一起并发跑**。

## 关联知识

- [[FastAPI与ASGI后端框架]]
- [[Kerberos域接入与用户映射管控]]
- [[SangBridge AI统一接入与管理控制台]]
