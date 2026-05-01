---
layout: post
title: "我用 asyncio 重写了公司数据采集系统，CTO 以为我偷偷加了服务器"
date: 2026-05-01 00:00:00 +0800
---

事情是这样的。上周产品经理丢过来一个需求：我们需要从 200 个数据源实时拉取行情数据，每 10 秒刷新一次，延迟必须控制在 2 秒以内。我一看老代码——同步 requests 逐条请求，跑完一轮要 14 秒，CPU 使用率不到 5%，但时间全耗在等网络 IO 上了。当时就想，市面上一个 Python 经典方案是 asyncio，于是花了一个下午把采集核心重写了。上线后 QPS 从 20 直接拉到 500，CTO 盯着监控面板看了五分钟，转头问我是不是偷偷申请了加机器。我说没，就改了几十行 Python。

下面把这套写法的核心技术、完整代码，以及我踩过的两个深坑一块分享出来。

---

## 你写的 Python，大部分时间都在等

先看一个典型场景：从 10 个 URL 抓数据，同步写法大概是：

```python
import time
import requests

def fetch_sync(url: str) -> str:
    print(f"[{time.strftime('%X')}] 请求 {url}")
    resp = requests.get(url, timeout=5)
    return resp.text[:50]  # 截取前50字符示意

def main_sync():
    urls = [f"https://httpbin.org/delay/1?t={i}" for i in range(10)]
    start = time.perf_counter()
    results = [fetch_sync(url) for url in urls]
    elapsed = time.perf_counter() - start
    print(f"同步总耗时: {elapsed:.2f}s，数据条数: {len(results)}")

if __name__ == "__main__":
    main_sync()
```

这段代码跑下来耗时 12 秒以上，因为每个请求依次阻塞，10 个请求串行执行。而 CPU 基本在睡觉——阻塞 IO 不占计算资源，线程却在原地干等。

## 换成 asyncio，10 个请求一个最慢的说了算

asyncio 的核心思路：单线程 + 事件循环。当一个协程发出网络请求开始等待，它把执行权交还给事件循环，事件循环马上去调度下一个协程。最终总耗时 ≈ 最慢的那个请求，而不是所有请求之和。

完整对比代码如下（可直接跑）：

```python
import asyncio
import time
import httpx   # httpx 提供原生 async 支持

# --- 异步版本 ---
async def fetch_async(client: httpx.AsyncClient, url: str) -> str:
    print(f"[{time.strftime('%X')}] 请求 {url}")
    resp = await client.get(url, timeout=5)
    return resp.text[:50]

async def main_async():
    urls = [f"https://httpbin.org/delay/1?t={i}" for i in range(10)]
    start = time.perf_counter()
    async with httpx.AsyncClient() as client:
        tasks = [fetch_async(client, url) for url in urls]
        results = await asyncio.gather(*tasks)
    elapsed = time.perf_counter() - start
    print(f"异步总耗时: {elapsed:.2f}s，数据条数: {len(results)}")

if __name__ == "__main__":
    asyncio.run(main_async())
```

运行结果：异步版本耗时仅 1.1 秒左右，几乎等于单个请求的延迟。这就是 `asyncio.gather` 的威力——它把所有协程同时提交给事件循环，后者的调度器让它们“并发”等待 IO。

## 进阶技巧：限制并发，别把服务器打爆

现实中你不可能无限制并发——上游接口有速率限制，或者机器本身端口资源有限。这时候用 `asyncio.Semaphore` 做并发控制，比如同时只允许 20 个请求：

```python
import asyncio
import httpx

CONCURRENCY = 20

async def fetch_with_limit(sem: asyncio.Semaphore, client: httpx.AsyncClient, url: str):
    async with sem:  # 超过 20 个协程时，其他会在这里等待
        resp = await client.get(url, timeout=5)
        return resp.status_code

async def main_limited():
    urls = [f"https://httpbin.org/delay/1?t={i}" for i in range(100)]
    sem = asyncio.Semaphore(CONCURRENCY)
    async with httpx.AsyncClient() as client:
        tasks = [fetch_with_limit(sem, client, url) for url in urls]
        results = await asyncio.gather(*tasks)
    print(f"完成 {len(results)} 个请求")

asyncio.run(main_limited())
```

信号量的原理很简单：每次 `async with sem` 会尝试获取一个许可，如果已有 20 个协程在执行，新协程就被挂起，直到某个协程完成释放许可。这是 asyncio 原生的流控方式，比线程池优雅得多。

## 踩坑实录：这两个坑我各踩了 1 小时

### 坑一：在协程里调了同步的 `requests.get`

最初迁移时偷懒，在 `async def` 里直接写了 `requests.get(url)`，结果整个事件循环被阻塞，并发变成了串行。排查半天才反应过来——**协程里绝对不能调用同步阻塞函数**。要么用异步库（如 `httpx`、`aiohttp`），要么用 `loop.run_in_executor()` 把阻塞函数扔进线程池：

```python
# 临时方案：把同步调用丢进线程池
loop = asyncio.get_running_loop()
resp = await loop.run_in_executor(None, requests.get, url)  # 不推荐长期用
```

但这只是权宜之计，线程池规模有限，真正的 IO 密集场景务必选择原生异步库。

### 坑二：`gather` 遇到异常直接炸，其他协程白跑了

默认 `asyncio.gather(*tasks)` 如果有一个任务抛出异常，它会立即把异常传播出来，其他还在跑的任务会被取消。数据采集场景里这很致命——一个接口超时，其他 199 个正常返回的数据全丢了。解决方法是加 `return_exceptions=True`：

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
for i, res in enumerate(results):
    if isinstance(res, Exception):
        print(f"任务{i}异常: {res}")
    else:
        process(res)
```

这样异常不会被抛出，而是作为结果的一部分返回，你拿到的是混合列表，需要手动区分。这条参数不显眼，但不用它线上必炸。

另外，`gather` 不处理任务之间的依赖，需要按顺序执行就用 `asyncio.wait` 或 `asyncio.as_completed`，这些区别一定要根据业务场景选。

## 总结

Python 并发不在于开多少线程，而在于能不能用好事件循环和协程。asyncio 把 IO 等待变成了调度资源，让单线程单进程也能扛住高并发数据采集。

---

#Python #异步编程 #asyncio #性能优化 #技术博客