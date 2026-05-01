---
layout: post
title: "Python asyncio 踩坑实录：这个问题坑了我 3 小时！"
date: 2026-05-01 00:00:00 +0800
---

上周在给 DevOS 的“自主内容生成”模块加并发抓取能力时，我想当然地写了一段 asyncio 代码，结果上线后数据丢了一半、也没报错。溯源才发现，是错误用法让协程悄悄崩掉、事件循环还假装一切正常。这篇文章就复盘一下那次翻车，同时分享一些经过验证的 asyncio 最佳实践，希望能让你少走弯路。

## 背景：我要做一个“并发爬虫”

DevOS 里有一个功能是同时向多个信息源拉取摘要，再汇总生成日报。源有 REST API、有静态网页，属于典型的 IO 密集型场景——这正是 asyncio 的主场。我快速写了个原型：

```python
import asyncio
import aiohttp

async def fetch_source(session, url):
    async with session.get(url) as resp:
        data = await resp.json()
        return data["summary"]

async def main():
    urls = [
        "https://api.dev1.example.com/news",
        "https://api.dev2.example.com/news",
        "https://api.dev3.example.com/news",
    ]
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_source(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        print("汇总结果:", results)

asyncio.run(main())
```

测试了几个正常源，跑得飞起，耗时从串行的 4 秒降到 1.2 秒。我觉得稳了，直接就推到了线上。

三天后查日志，发现日报经常只有一两条源的内容，而且没有任何异常堆栈。我停掉其他任务，手动跑了好几遍，才发现：**如果某个源返回 500 或者超时，`gather` 默认行为会直接抛出异常，而当时我没处理，后续汇总逻辑根本没执行，但最坑的是——异常被静默吞掉了**。

这就引出了第一个大坑。

## 坑 1：`asyncio.gather` 的异常处理策略

看文档才知道，`gather()` 默认的策略是：只要有一个子任务抛异常，它马上就把这个异常抛出，其他正在跑的任务虽然不会被取消，但它们的返回值会被丢弃。更让人崩溃的是，如果外层用 `try/except` 捕获了异常，那些未完成的任务很可能还在后台跑，最终留下一个不知所踪的报错。

我的解决方法是改用 `asyncio.gather(..., return_exceptions=True)`。这样 gather 永远不抛异常，而是把异常对象作为返回值放进结果列表，你可以区别对待：

```python
async def safe_fetch(session, url):
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
            resp.raise_for_status()
            data = await resp.json()
            return data["summary"]
    except Exception as e:
        # 把异常返回给 gather，而不是在协程内部 crash
        return e

async def main_v2():
    urls = [...]
    async with aiohttp.ClientSession() as session:
        tasks = [safe_fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        summaries = []
        for url, result in zip(urls, results):
            if isinstance(result, Exception):
                print(f"源 {url} 抓取失败: {result}")
            else:
                summaries.append(result)

        print("可用摘要:", summaries)
```

这样做的好处：**一目了然拿到所有成功结果，同时记录所有失败原因**，不会因为一个源挂了就全盘丢弃。更重要的是外部 gather 完全不抛异常，不会中断后续的汇总逻辑。

## 坑 2：不小心把阻塞调用丢进了协程

排查过程中我又发现一个问题：原本我用 `requests.get()` 在异步函数里获取一个配置接口，只是偶尔跑一下，测试时没暴露。但并发一上来，那个阻塞请求直接把整个事件循环卡死了，导致其他所有协程都干等着。

很多刚接触 asyncio 的开发者会踩这个坑——**把所有 `def` 前面加个 `async` 就以为是异步了**，实际上只要内部有一个同步阻塞操作（比如 `requests.get`、`time.sleep`、文件全量读取），整个线程就被占住，事件循环再也切换不了任务。

改正方法是：网络请求都用 `aiohttp` 或 `httpx` 的异步客户端；文件操作如果必须用同步读，就丢给线程池：

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=4)

async def read_config_async(path: str) -> dict:
    # 把同步的 open/read 放进线程池，避免阻塞事件循环
    loop = asyncio.get_running_loop()
    def _sync_read():
        import json
        with open(path, "r") as f:
            return json.load(f)
    return await loop.run_in_executor(executor, _sync_read)
```

原则：**异步函数里绝对不能出现同步的、耗时不确定的阻塞操作**。不确定的就用 `run_in_executor` 包裹，或者用对应的异步库。

## 坑 3：忘记限制并发数，被对方限流

修完上面两个问题，以为太平了。没想到有一回十几个源同时返回大报文，我这边瞬时发了 20 个请求到同一个第三方 API，直接触发了人家的速率限制，拉黑了 15 分钟。根源是 `gather` 会一下子把任务全丢进事件循环，对目标站点毫无节制。

解决方案是加个 `asyncio.Semaphore`，将并发数控制在对方可接受的范围：

```python
sem = asyncio.Semaphore(5)   # 最多同时 5 个请求

async def rate_limited_fetch(session, url):
    async with sem:
        return await safe_fetch(session, url)
```

这个小改动让我再也没有被限流过。

## 总结与心得

- **异常处理**：用 `return_exceptions=True` 将异常当作值返回，避免单个失败拖累全局。
- **避免阻塞**：异步函数内绝不放同步阻塞代码，需要时走 `loop.run_in_executor`。
- **友好限流**：`asyncio.Semaphore` 是保护自己和对方服务的最后一道防线。
- **调试技巧**：开启 `asyncio` 的 debug 模式（`PYTHONASYNCIODEBUG=1` 或 `loop.set_debug(True)`）会让慢回调、未等待的协程等潜在问题尽早暴露。

那 3 小时虽然煎熬，但彻底帮我建立了一套 asyncio 的防御性编程习惯。现在 DevOS 的并发拉取模块已经稳定运行超过 2000 小时，再没丢过一条数据。希望这些教训也能让你的异步代码变得更“皮实”。

**一句话点睛**：异步编程的坑，大多藏在你“以为它异步了”的地方。

#Python #AsyncIO #并发编程 #后端开发 #踩坑实录