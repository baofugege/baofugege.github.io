---
layout: post
title: "asyncio 踩坑实录：一个并发请求卡了我3小时，直到我发现这个 bug"
date: 2026-04-30 00:00:00 +0800
---

事情是这样的：上周我写了个小工具，要批量抓取 200 个 API 的数据。想着 IO 密集嘛，用 `asyncio` + `aiohttp` 梭哈，代码 15 分钟写完，跑起来前几个请求飞快，我还沾沾自喜。结果后半程就开始随机超时，有些协程干脆永不返回，日志里一片死寂，连异常都不抛。3 小时后我终于找到原因——一个看似人畜无害的 `time.sleep(0.1)` 混进了协程链，直接让整个事件循环瘫痪。这篇就把我踩过的坑和最终沉淀的最佳实践一次性讲透。

## 坑一：同步代码混进协程，事件循环直接“中风”

很多从多线程转过来的兄弟会下意识在 `async def` 里用 `time.sleep()`、`requests.get()`，或者调用一个没标 `async` 的数据库 driver。因为 asyncio 是单线程协作式并发，一旦一个协程里跑了同步阻塞调用，整个线程就卡在那里，事件循环无法调度任何其他任务——表面上是“并发”，实际退化成了串行。

我当时就是在解析响应时，调了一个老的工具函数，里面居然有一行 `time.sleep(0.1)` 用来“等待日志刷盘”。就这么个 0.1 秒，在高并发下把吞吐直接拉崩。

**正确做法**：凡是可能阻塞的操作，要么用 `await asyncio.sleep()`，要么用 `loop.run_in_executor()` 丢到线程池。如果你的第三方库只支持同步，务必用 `asyncio.to_thread()`（3.9+）或 `run_in_executor` 包一层：

```python
import asyncio
import time
import aiohttp

async def fetch_data(session, url):
    """正确的异步请求：所有阻塞点都是 awaitable"""
    async with session.get(url) as resp:
        data = await resp.json()
    # 假设有段 CPU 密集计算，用 to_thread 避免长时间独占事件循环
    processed = await asyncio.to_thread(heavy_cpu, data)
    return processed

def heavy_cpu(data):
    # CPU 密集型任务放线程池，不阻塞事件循环
    # 如果内有 time.sleep，也只会阻塞线程池中的一个线程
    time.sleep(0.1)  # 模拟耗时计算
    return data
```

## 坑二：`gather` 不设超时，协程“闷声发大财”

我第一次写的批量抓取大概长这样：

```python
async def main():
    urls = [f"https://api.example.com/item/{i}" for i in range(200)]
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_data(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    return results
```

看起来没毛病对吧？但某些 API 响应极慢，或者对方压根不回包，TCP 连接就 hang 住了。`gather` 默认会等所有任务结束，一个任务卡住，全盘皆输。更致命的是，`aiohttp` 默认没有超时，一个连接可能挂几十分钟，既不抛异常也不释放，内存里全是僵尸协程。

**解决**：为每个任务包裹 `asyncio.wait_for()`，并给 `aiohttp` 设置 `ClientTimeout`。

```python
import asyncio
import aiohttp
from aiohttp import ClientTimeout

async def fetch_with_timeout(session, url, timeout=10):
    """每个请求单独设置超时，防止个别任务拖垮全局"""
    try:
        async with session.get(url, timeout=ClientTimeout(total=timeout)) as resp:
            return await resp.json()
    except asyncio.TimeoutError:
        return {"error": "timeout", "url": url}

async def main():
    urls = [...]  # 200 个 URL
    timeout = aiohttp.ClientTimeout(total=10)  # 全局默认超时
    async with aiohttp.ClientSession(timeout=timeout) as session:
        tasks = [fetch_with_timeout(session, url) for url in urls]
        # return_exceptions=True，让部分失败不阻断其他结果
        results = await asyncio.gather(*tasks, return_exceptions=True)
    # 分类处理成功与失败
    success = [r for r in results if not isinstance(r, Exception)]
    errors = [r for r in results if isinstance(r, Exception)]
    print(f"成功 {len(success)}，失败 {len(errors)}")
    return success, errors
```

把 `gather` 的 `return_exceptions=True` 和任务级超时配合，你就能拿到“部分成功”的结果，而不是一个任务炸了就全丢。

## 坑三：无限制并发，触发目标服务器的“自卫反击”

200 个 URL 一起扔进 `gather`，等于是同时发起 200 个 TCP 连接。本地资源可能扛得住，但对方服务器会把你当成 DDoS，直接封 IP 或者返回 429。更要命的是，大量堆叠的协程在没有信号量保护的情况下，内存中的 socket 缓冲区会暴涨，吞吐量反而下降。

**最佳实践**：用 `asyncio.Semaphore` 控制最大并发数。

```python
import asyncio
import aiohttp

async def bounded_fetch(sem, session, url):
    async with sem:
        return await fetch_with_timeout(session, url)

async def main():
    urls = [...]  # 200 个 URL
    sem = asyncio.Semaphore(20)  # 同时最多 20 个请求
    async with aiohttp.ClientSession() as session:
        tasks = [bounded_fetch(sem, session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

信号量就像“并发大门”，每次只放 20 个进去，其他在门外排队。实测下来，比起无限制爆发式请求，带限流的版本总耗时反而更短，因为少了对端的反压等待和重试。

## 踩坑 & 注意事项

1. **Windows 上默认事件循环不同**。 `asyncio.run()` 在 Windows 使用 `ProactorEventLoop`，如果用到了未适配的子进程或管道，可能会报错。解决： `asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())`，或者干脆用 WSL。

2. **`create_task` 后的引用丢失**。如果你把 `asyncio.create_task(coro)` 的返回值丢了，Python 垃圾回收可能会在任务完成前就干掉它，看起来像任务“神秘消失”。对策是把所有 task 存到列表里，直到 `await` 或 `gather`。

3. **aiohttp 的 session 必须复用**。千万别在每次请求里 `async with aiohttp.ClientSession() as s` 新建 session，这会导致连接池失效，每次都 TCP 握手+TLS 握手，性能还不如同步的 `requests`。全局创建一个 session，传入所有协程复用。

4. **日志中协程无异常却卡死**。大概率是 `await` 了某个永远不会完成的 Future。排查用 `asyncio.all_tasks()` + `logging`，或者引入 `asyncio.Task` 的回调打印堆栈（`task.add_done_callback`）。还有一个利器：`PYTHONASYNCIODEBUG=1` 环境变量，能打印慢回调、未完成的 Future 等。

5. **小心 `asyncio.run()` 的多层调用**。在已经运行的事件循环里再调用 `asyncio.run()` 会报 `RuntimeError: asyncio.run() cannot be called from a running event loop`。在 Jupyter notebook 中往往已有 loop，直接用 `await main()` 即可。

## 总结

asyncio 用得好是利器，用不好就是隐形炸弹——同步阻塞、缺失超时、无限制并发，三座大山搬掉了，你才算真正驾驭了协程。

#Python #异步编程 #asyncio #踩坑复盘 #后端开发