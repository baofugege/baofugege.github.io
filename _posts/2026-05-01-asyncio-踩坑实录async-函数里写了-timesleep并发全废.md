---
layout: post
title: "asyncio 踩坑实录：async 函数里写了 time.sleep，并发全废了，这坑我爬了3小时"
date: 2026-05-01 00:00:00 +0800
---

事情是这样的，我们有个数据采集服务，需要从十几个上游 API 同时拉数据，同步代码跑下来一趟要 30 分钟。我心想这不就是 asyncio 的拿手好戏吗？周末花了一下午把 requests 全换成了 aiohttp，函数全部加上 `async/await`，整套代码跑起来——结果还是 30 分钟，一根毛都没少，我当场傻眼。

最后定位到一个藏在嵌套函数里的 `time.sleep(0.5)`。就是因为这半秒，整个事件循环在这个协程里被死死卡住，所谓的“异步并发”直接退化成了串行。

所以说，asyncio 那些很反直觉的坑，不自己踩一遍真的很难意识到。下面我把这次踩坑的排查过程、核心原因、以及正确的避坑姿势整理出来。

## 为什么一个 sleep 就能废掉并发？

先复习一下 asyncio 的核心机制：**事件循环（Event Loop）**。它本质是一个单线程调度器，里面维护着一个任务队列。每一个 `async def` 声明的协程在 `await` 的时候会把控制权交还给事件循环，事件循环就可以趁机切到其他就绪的协程上去干活。

但这里的“交还”必须是你 **主动让出** 才行。`await asyncio.sleep(n)` 会向事件循环注册一个定时器，然后立即交出控制权，这样其他任务就能跑。而 `time.sleep(n)` 是一个**同步阻塞调用**，它让整个线程睡死在原地，事件循环完全拿不到控制权——这期间你不管写了多少协程，全部都得排队干等。

简化理解就是：
- `await asyncio.sleep()`：我给事件循环留了个定时提醒，然后自己先让开。
- `time.sleep()`：我睡大觉，天王老子来了也别想把控制权拿走。

## 错误示范 vs 正确写法

**错误代码**（看起来像是异步，实际上单线程堵死）：

```python
import asyncio
import time

async def fetch_data(url: str):
    # 模拟请求前处理
    print(f"开始请求 {url}")
    time.sleep(0.5)           # ❌ 同步阻塞，整个事件循环停滞
    # 这里还会去发起 aiohttp 请求等等
    print(f"完成请求 {url}")

async def main():
    urls = [f"https://api.example.com/data/{i}" for i in range(10)]
    # 看似并发启动
    tasks = [asyncio.create_task(fetch_data(url)) for url in urls]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

运行时你会发现打印是一条一条顺序出来的，10 个任务总耗时 5 秒以上，并发形同虚设。

**正确写法**（使用 `asyncio.sleep` 让渡控制权）：

```python
import asyncio
import aiohttp

async def fetch_data(session: aiohttp.ClientSession, url: str) -> dict:
    """
    真正的异步请求函数：IO 全部交给事件循环调度
    """
    print(f"开始请求 {url}")
    # 模拟速率限制等待，使用 asyncio.sleep，不阻塞其他协程
    await asyncio.sleep(0.5)
    
    async with session.get(url, timeout=10) as resp:
        data = await resp.json()
        print(f"完成请求 {url}, 状态码 {resp.status}")
        return data

async def main():
    urls = [f"https://api.example.com/data/{i}" for i in range(10)]
    
    # 使用连接池复用 TCP 连接，减少开销
    connector = aiohttp.TCPConnector(limit=20)   # 最大并发连接数
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [fetch_data(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # 简单错误处理
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"请求 {urls[i]} 失败: {result}")

asyncio.run(main())
```

这样一改，10 个任务的 `asyncio.sleep(0.5)` 是同时进行的，总耗时被压缩到 0.5 秒左右（加上实际请求时间）。再配合 aiohttp 的连接池，效率是质变级的。

## 踩坑经验：这些点不注意，迟早还会翻车

1. **第三方库是否原生异步**  
   不是所有带 `async/await` 的项目就行了。`time.sleep`、`requests`、`pymongo` 等同步库一旦出现在协程内部，立刻拉垮。一定要用它们的异步替代：`aiohttp`、`httpx`、`motor`（异步 MongoDB）、`aiomysql` 等。如果某个库实在没有异步版本，用 `await asyncio.to_thread(同步函数, *args)` 扔到线程池里跑，虽然不完美但至少不会阻塞事件循环。

2. **asyncio.gather 的异常处理陷阱**  
   默认情况下 `gather` 遇到一个任务抛出异常，会立即把异常传播出去，导致其他还在运行的任务被取消。如果你希望所有任务都执行完再统一处理结果，一定要加 `return_exceptions=True`，然后手动检查每个返回值是否为 `Exception` 实例。

3. **不要用 `asyncio.create_task` 后不管它的异常**  
   创建了 Task 但没有 await 或持有引用，当这个 Task 内部抛异常时，它会被垃圾回收静默吞掉，你连错误日志都看不到。每个 Task 要么被 `gather`，要么定期检查异常。

4. **不要无限制地创建协程**  
   采集几百个 URL 时创建几百个并发连接，很可能触发目标 API 的限流或者耗尽本地文件描述符。用 `asyncio.Semaphore` 控制最大并发数是基本操作：

```python
sem = asyncio.Semaphore(20)  # 最多同时 20 个请求

async def limited_fetch(session, url):
    async with sem:
        return await fetch_data(session, url)
```

5. **为每个请求设置超时**  
   异步 IO 如果没有超时，某个请求 hang 住，会把整个任务组拖死。永远给你的 `session.get` 加上 `timeout` 参数。

## 总结

**asyncio 的“异步”不是魔法——你写了同步阻塞，它就变回单线程串行；你彻底交出控制权，它才给你真正的高并发。**

#Python #异步编程 #踩坑 #性能优化 #爬虫