---
layout: post
title: "asyncio 踩坑实录：一个 bug 让我一度怀疑 Python 的协程是假的"
date: 2026-05-01 00:00:00 +0800
---

事情是这样的：上周五下午，我正准备收尾一个数据同步脚本，想着用 asyncio 并发拉取 30 个 API 分页，写完一跑——请求居然还是串行的。3 小时的排查，才发现自己掉进了事件循环的“隐形陷阱”。这篇复盘，希望能帮你少走弯路。

## 背景：一个看似简单的并发需求

我们要从某个开放平台拉取全量订单，对方限制了单页 100 条，共有 30 页。同步逻辑很清晰：先获取总页数，然后同时发出 30 个请求，最后合并结果。我第一反应就是上 `asyncio` + `aiohttp`，心里默念“这不就是 IO 密集型的标准剧本嘛”。

初版代码长这样：

```python
import asyncio
import aiohttp

async def fetch_page(session, page: int):
    url = f"https://api.example.com/orders?page={page}&size=100"
    async with session.get(url) as resp:
        data = await resp.json()
        return data["items"]

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_page(session, p) for p in range(1, 31)]
        results = await asyncio.gather(*tasks)
    # 合并所有返回的订单列表
    all_orders = [item for page in results for item in page]
    print(f"共获取 {len(all_orders)} 条订单")

asyncio.run(main())
```

逻辑看着没毛病：创建 30 个协程任务，用 `gather` 并发执行，拿到结果后展开。可实际运行时，日志里请求是一个接一个返回的，总耗时约 30 秒，和同步 `requests` 没区别。**协程去哪了？**

## 排查过程：逐层剥开事件循环的伪装

### 第 1 层：怀疑 aiohttp 的 session 限制

我第一个念头是 `ClientSession` 内部有连接池限制，可能卡在 TCP 连接上了。于是给 `session.get` 加了 `timeout`，又显式设置了连接数：

```python
connector = aiohttp.TCPConnector(limit=50)
async with aiohttp.ClientSession(connector=connector) as session:
    ...
```

没变化。排除连接池。

### 第 2 层：怀疑“假协程”占据了任务

我打印了每个 `fetch_page` 的开始和结束时间，发现所有任务确实同时启动了，但都在等第一个请求返回后才继续。这不科学——`await resp.json()` 应该交出控制权才对。于是我用 `asyncio.ensure_future` 替换了 `gather`，手动逐一轮询，也没改观。

### 第 3 层：罪魁祸首浮出水面

最后我把抓包打开，发现实际上 **所有请求几乎是同时发出的**，但服务端返回了 429（限流），而我的代码里根本没处理状态码。aiohttp 在遇到 429 时默认会重试，但重试策略是阻塞式的？不对，aiohttp 不会自动重试。真正的问题在下面。

原来，服务端反爬策略要求在请求头里带上 **签名**，而签名算法依赖时间戳，时间戳精确到**秒**。我在构造请求头时，用的是 `time.time()` 并截断到秒，结果 30 个任务在同一秒内使用了**相同的签名**，服务端只接受了第一个，其余全部返回 403。我的代码在拿到 403 后没有抛异常，而是 `return data["items"]` 直接 KeyError，但被 `gather` 默认吞掉了异常（当时没设 `return_exceptions=True`），导致事件循环一直在等超时重试——这就是串行假象。

## 真正的解决方案 & 并发优化

知道问题后，修起来就清晰了：
1. 签名中加入**随机 nonce** 或**递增序号**，确保同一秒内签名不同；
2. 给 `gather` 加 `return_exceptions=True`，让错误显式化；
3. 加上 `asyncio.Semaphore` 控制并发数，防止打爆对方 API。

优化后的完整代码（可直接运行）：

```python
import asyncio
import aiohttp
import hashlib
import time
import os
from typing import List, Dict

API_SECRET = os.getenv("API_SECRET", "dev-secret")
MAX_CONCURRENT = 10  # 控制同时进行中的请求数

async def fetch_page(session, sem: asyncio.Semaphore, page: int):
    # 1. 构造带 nonce 的签名，避免同一秒内签名重复
    ts = int(time.time())
    nonce = f"{page}-{ts}-{os.urandom(4).hex()}"
    raw = f"{ts}{nonce}{API_SECRET}"
    sign = hashlib.sha256(raw.encode()).hexdigest()
    headers = {
        "X-Timestamp": str(ts),
        "X-Nonce": nonce,
        "X-Sign": sign,
    }
    url = f"https://api.example.com/orders?page={page}&size=100"

    async with sem:  # 信号量控制并发上限
        async with session.get(url, headers=headers) as resp:
            if resp.status == 429:
                # 如果仍然超限，简单重试一次（实际可加指数退避）
                await asyncio.sleep(2)
                async with session.get(url, headers=headers) as retry_resp:
                    retry_resp.raise_for_status()
                    data = await retry_resp.json()
            else:
                resp.raise_for_status()  # 非 2xx 直接抛异常
                data = await resp.json()
            return data["items"]

async def main():
    sem = asyncio.Semaphore(MAX_CONCURRENT)
    connector = aiohttp.TCPConnector(limit=20)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [fetch_page(session, sem, p) for p in range(1, 31)]
        # return_exceptions=True 可以捕获单个任务的异常而不中断 gather
        results = await asyncio.gather(*tasks, return_exceptions=True)

    all_orders = []
    for idx, res in enumerate(results, start=1):
        if isinstance(res, Exception):
            print(f"页 {idx} 请求失败: {res}")
        else:
            all_orders.extend(res)
    print(f"成功获取 {len(all_orders)} 条订单")

if __name__ == "__main__":
    asyncio.run(main())
```

实测优化后，10 个并发请求在 3 秒内完成 30 页拉取，比最初版本快了近 10 倍。

## 这些坑，你可能也会踩

1. **asyncio.gather 默认会抛异常**：如果某个任务崩了，其他任务的结果会被丢弃，只有把 `return_exceptions=True` 打开才能收集所有结果并分别处理。很多人以为 gather 像 Promise.all 一样“部分成功”，其实它更严格。

2. **事件循环不会魔法**：协程只有在遇到 `await` 时才会让出控制权。如果你在协程里写了 `time.sleep()` 或者调了一个同步库（如 `requests`），整个事件循环就卡死了。排查时可以用 `asyncio.current_task()` 和日志查看任务切换点。

3. **服务端限流是并发的隐形杀手**：一味提升客户端并发数，如果服务端返回 429/403，重试策略不得当，线程池反而会陷入“重试风暴”。用信号量控制并发数，并实现指数退避重试，是生产环境的必备修养。

4. **aiohttp 的 session 要复用**：千万别在每个请求里新建 `ClientSession`，那样不但无法复用连接池，还可能导致文件描述符泄露。用 `async with` 包裹整个会话生命周期。

5. **别忘了结构化异常**：HTTP 状态码检查不要依赖返回数据，先用 `resp.raise_for_status()` 强制校验，再用 `try/except` 捕获 `aiohttp.ClientResponseError`，否则一个 KeyError 就会把所有线索埋没。

## 一句话总结

asyncio 的并发能力值得信任，但并发效果容易被“签名重复”、“隐藏异常”、“低效重试”这些现实细节吃掉，排查时不妨关掉隐式容错，让每个错误都浮出水面。

#Python #asyncio #后端开发 #踩坑日记 #高性能编程