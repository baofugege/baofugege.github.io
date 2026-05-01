---
layout: post
title: "Python asyncio 踩坑实录：这个问题坑了我3小时"
date: 2026-05-01 00:00:00 +0800
---

上周接到一个需求：批量拉取30个三方API的数据，清洗后入库。我心想这不简单，requests+for循环一把梭。结果跑了40多秒，产品经理的眼神像刀子一样扎过来。“能不能快一点？”——我硬着头皮说能，转身打开了asyncio的大门。提速10倍确实爽，但中间踩的坑，每一个都可能让你在深夜怀疑人生。今天把这些真实到肉疼的经验复盘出来，希望能帮你少走弯路。

## 为什么是asyncio？先看一组数据

我用一个简化版的场景做实测：并发请求20个URL，每个接口响应时间约200ms。同步版请求耗时4秒，异步版只用了0.3秒，效率差了13倍。核心原因在于，asyncio用 **事件循环** + **协程** 把IO等待的时间都利用起来了——线程不必傻等网络回包，而是立刻切换去发下一个请求。Python的GIL在这里根本构不成瓶颈，因为瓶颈在IO，不在CPU。

不过数字好看，坑也多。下面我会用两个可以直接跑的代码示例，把原理和实战串起来，顺便挖出那些文档里不怎么写的暗坑。

## 核心原理：事件循环到底在忙什么

事件循环是单线程的。它维护了一个任务队列，哪个协程在 `await` 一个IO操作时，就会把执行权还给循环，循环立刻调度另一个就绪的协程。你可以这么理解：餐厅里你是唯一的服务员（单线程），客人点完菜（发起请求）你不需要站着等厨房出餐（等待响应），而是先去服务下一位客人。菜好了厨房会喊你（IO就绪回调），你再端上去。这样一个人服务20桌都不在话下。

用代码感受一下：

```python
import asyncio
import time

async def fetch_one(url: str) -> str:
    """
    模拟一个IO操作：请求某个URL并返回数据
    用asyncio.sleep模拟网络延迟，不会阻塞事件循环
    """
    print(f"开始请求 {url}")
    await asyncio.sleep(0.2)   # 模拟200ms的网络IO
    print(f"完成请求 {url}")
    return f"{url} 的数据"

async def main_sync_like():
    """
    错误示范：把async当同步写，一个接一个await
    耗时 = N * 单任务耗时，协程的优势全没了
    """
    t0 = time.time()
    result1 = await fetch_one("url_1")
    result2 = await fetch_one("url_2")
    result3 = await fetch_one("url_3")
    print(f"耗时: {time.time() - t0:.2f}s")  # 约0.6s

async def main_async():
    """
    正确打开方式：用gather并发执行
    耗时 ≈ 最慢那一个任务的耗时
    """
    t0 = time.time()
    results = await asyncio.gather(
        fetch_one("url_1"),
        fetch_one("url_2"),
        fetch_one("url_3")
    )
    print(f"耗时: {time.time() - t0:.2f}s")  # 约0.2s
    print(results)

# Python 3.7+ 运行方式
# asyncio.run(main_async())
```

`asyncio.gather()` 是并发的关键：它同时注册多个协程，当所有协程都完成（或出错）时才返回。注意这里所有请求“同时”发起，总耗时只受最慢的那个URL影响，而不是叠加。所以异步能几十倍提升IO密集场景的性能。

## 真实生产级并发：带超时与异常处理

刚才的代码太理想化。现实中，三方接口可能超时、可能挂掉、可能返回脏数据。直接一个`gather`崩掉整个批次，这在生产环境绝对不能接受。我们必须给每个任务加“安全气囊”：

```python
import asyncio
import aiohttp
import time
from typing import List, Dict, Any

API_URLS = [
    "https://httpbin.org/delay/0.1",
    "https://httpbin.org/delay/0.2",
    "https://httpbin.org/status/500",   # 故意放一个会炸的
    "https://httpbin.org/delay/0.3",
]

async def fetch_with_timeout(session: aiohttp.ClientSession,
                             url: str,
                             timeout: int = 5) -> Dict[str, Any]:
    """
    单个请求的协程：加超时、加异常捕获
    返回统一格式的字典，方便上游处理
    """
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=timeout)) as resp:
            data = await resp.json()
            return {"url": url, "status": resp.status, "data": data, "error": None}
    except asyncio.TimeoutError:
        return {"url": url, "status": None, "data": None, "error": "timeout"}
    except Exception as e:
        return {"url": url, "status": None, "data": None, "error": str(e)}

async def batch_fetch(urls: List[str], concurrency: int = 10) -> List[Dict[str, Any]]:
    """
    并发拉取一组URL，支持限制并发数（用Semaphore）
    返回所有结果，包含成功和失败的记录
    """
    semaphore = asyncio.Semaphore(concurrency)  # 防止把对方打爆
    async with aiohttp.ClientSession() as session:
        async def bounded_fetch(url):
            async with semaphore:
                return await fetch_with_timeout(session, url)

        tasks = [bounded_fetch(url) for url in urls]
        results = await asyncio.gather(*tasks)  # *解包列表
    return results

async def main():
    t0 = time.time()
    results = await batch_fetch(API_URLS, concurrency=5)
    for r in results:
        print(f"{r['url']} -> {'OK' if not r['error'] else r['error']}")
    print(f"总耗时: {time.time() - t0:.2f}s")

# asyncio.run(main())
```

上面这段代码我特意加了三个生产级细节：

1. **单任务异常隔离**：每个协程内部`try/except`，返回统一结构体。这样一个接口挂了，不会导致整个`gather`抛异常取消其他任务。
2. **Semaphore 限流**：别同时开200个连接打爆下游，`asyncio.Semaphore`就是异步世界的“红绿灯”，简单高效。
3. **ClientSession 复用**：`aiohttp`强烈建议在同一个session下跑所有请求，复用连接池，性能比每次新建session高3-5倍。

## 我亲自踩过的3个大坑（排查到凌晨的那种）

### 坑1：在协程里随手写了 `time.sleep()`

某次发现并发完全没效果，20个请求还是按顺序一个个跑。查了半天，原来在数据处理函数里调了`time.sleep(0.1)`。这玩意是同步阻塞，直接卡死整个事件循环，让其他协程动不了。异步世界只能用`await asyncio.sleep()`。记住：**协程里永远不要调同步阻塞函数**，如果必须用（比如某些老库），用`loop.run_in_executor()`丢到线程池里执行。

### 坑2：`gather`一个异常，全家翻车

`asyncio.gather()`默认行为是：只要有一个子任务抛出未捕获的异常，立刻将异常传播到`gather`调用处，同时**取消其他尚未完成的任务**。这意味着如果你没做单任务异常捕获，一个500就能让整批请求废掉。解决方案就是上面示例里的`try/except`包裹，或者给`gather`传`return_exceptions=True`参数让异常作为结果对象返回。我踩这坑时业务数据丢了一半，监控报警电话直接打过来了。

### 坑3：忘记 `await`，协程悄悄溜走

新手最爱犯的错：写了个`fetch_one(url)`，没加`await`也没塞进`gather`或`create_task`。结果是创建了一个协程对象但没有被执行，程序什么都不报，数据就是没拉回来。更隐蔽的是，如果环境开启了警告，会看到`RuntimeWarning: coroutine was never awaited`，但生产环境经常抑制了这类警告，让你查哭都找不到原因。我现在的肌肉记忆：写`async def`的同时，脑子里就同步想好它被谁`await`。

## 一句话总结

asyncio是IO密集型任务的性能利器，但协程不是魔法——你得理解事件循环的调度机制，管好异常、超时和阻塞，才能真正让它为你提速而不是埋雷。

#Python #异步编程 #asyncio #后端性能优化 #程序员踩坑