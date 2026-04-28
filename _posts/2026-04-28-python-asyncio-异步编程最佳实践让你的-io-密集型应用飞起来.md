---
layout: post
title: "Python asyncio 异步编程最佳实践：让你的 IO 密集型应用飞起来"
date: 2026-04-28 00:00:00 +0800
---

在现代后端开发中，网络请求、数据库查询、文件读写等 IO 操作往往是性能瓶颈所在。Python 的 `asyncio` 库通过**协程 + 事件循环**的机制，让单线程程序也能高效处理大量并发 IO，是构建高性能服务的利器。

---

## 核心概念速览

**事件循环（Event Loop）** 是 asyncio 的调度中枢。它运行在单线程中，不断轮询就绪的协程并执行。当某个协程遇到 `await` 时，控制权交回事件循环，由其调度其他就绪任务——这就避免了 CPU 在等待 IO 时的空转。

**协程（Coroutine）** 用 `async def` 定义，用 `await` 挂起：

```python
import asyncio

async def fetch_data(url: str) -> str:
    await asyncio.sleep(1)  # 挂起 1s，期间事件循环可运行其他任务
    return f"data from {url}"
```

---

## 最佳实践一：用 `gather()` 实现真正的并发

串行 `await` 是最常见的误区——它让你写了 `async` 代码，却退化成同步执行：

```python
# 错误：串行执行，总耗时 ~3s
async def bad_main():
    r1 = await fetch_data("url1")
    r2 = await fetch_data("url2")
    r3 = await fetch_data("url3")
```

正确做法是用 `asyncio.gather()` 并发启动所有任务，总耗时仅取决于最慢的那个：

```python
# 正确：并发执行，总耗时 ~1s
async def main():
    results = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3"),
    )
    print(results)

asyncio.run(main())
```

---

## 最佳实践二：用 `TaskGroup` 做结构化并发（Python 3.11+）

`asyncio.gather()` 有一个隐患：某个子任务抛出异常时，其他任务不会自动取消，可能造成资源泄漏。Python 3.11 引入的 `TaskGroup` 解决了这个问题：

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch_data("url1"))
        t2 = tg.create_task(fetch_data("url2"))
    # 离开 with 块时，所有任务已完成或被取消
    print(t1.result(), t2.result())
```

任意子任务失败时，`TaskGroup` 会自动取消其余任务并聚合异常——这是生产代码的推荐写法。

---

## 最佳实践三：别在协程里调用阻塞代码

`asyncio` 是协作式调度，任何同步阻塞调用（如 `time.sleep()`、`requests.get()`、同步文件读写）都会**卡住整个事件循环**，让其他协程无法运行。

```python
import asyncio
import time

# 危险：time.sleep 阻塞事件循环
async def blocking_bad():
    time.sleep(2)  # 整个循环被冻结 2s

# 正确：用 asyncio.sleep 或 run_in_executor
async def non_blocking_good():
    await asyncio.sleep(2)  # 只挂起当前协程

# 对于无法替换的同步库，用线程池隔离
async def use_sync_lib():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, time.sleep, 2)
```

IO 密集型用线程池（`ThreadPoolExecutor`），CPU 密集型用进程池（`ProcessPoolExecutor`）——这是绕过 GIL 的标准手法。

---

## 最佳实践四：合理控制并发量

无限并发会打垮下游服务或耗尽文件描述符。用 `asyncio.Semaphore` 限制同时运行的协程数：

```python
async def fetch_with_limit(url: str, sem: asyncio.Semaphore) -> str:
    async with sem:
        return await fetch_data(url)

async def main():
    sem = asyncio.Semaphore(10)  # 最多 10 个并发
    urls = [f"https://example.com/{i}" for i in range(100)]
    results = await asyncio.gather(*[fetch_with_limit(u, sem) for u in urls])
```

---

## 最佳实践五：始终用 `asyncio.run()` 作为入口

不要手动管理事件循环的创建与关闭，`asyncio.run()` 会处理好生命周期并在结束时清理资源：

```python
# 推荐
asyncio.run(main())

# 避免（旧写法，容易出错）
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

---

## 小结

| 场景 | 推荐方案 |
|------|----------|
| 多任务并发 | `asyncio.gather()` / `TaskGroup` |
| 限制并发数 | `asyncio.Semaphore` |
| 调用同步阻塞库 | `loop.run_in_executor()` |
| 程序入口 | `asyncio.run()` |

asyncio 的核心哲学是：**让等待变成机会**。只要你不在协程中引入阻塞操作，事件循环就能把每一个 IO 等待时间都榨干，用来推进其他任务。掌握这几条实践，就能写出真正高效的异步 Python 代码。

#Python #asyncio #AsyncProgramming #BackendDevelopment #ConcurrentProgramming