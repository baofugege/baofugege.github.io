---
layout: post
title: "Python asyncio 踩坑实录：以为能并发 1000 请求，结果全线阻塞，这个问题坑了我3小时"
date: 2026-05-01 00:00:00 +0800
---

上周老板甩过来一个需求：把竞品网站的几十万条商品数据全量爬下来。我想都没想，`requests` 一把梭，结果跑了 10 分钟才爬完 200 条——串行请求，一个等一个，CPU 闲得在摸鱼。当时心里门儿清：该上 asyncio 了，协程并发一开，分分钟搞定。

于是我吭哧吭哧把代码改成 `async def`，加上 `await`，运行——**耗时纹丝未动，请求还是一个接一个走**。盯着屏幕反复看了三遍，代码里明明有 `async`、有 `await`，怎么就串行了？这个问题整整坑了我 3 个小时。最后发现，罪魁祸首只有一句话：**在协程里调了同步库**。

这篇文章就把我踩的坑、排查过程、以及正确的生产级写法一次讲透。

---

## 事件循环、协程、并发的本质

Python 的 asyncio 不是什么“多线程”，它只有一个线程，核心是 **事件循环（Event Loop）**。你可以把它想象成一个超级服务员：客人点完菜（发起 IO 请求）之后，服务员不会傻站着等厨房出菜，而是立刻去服务下一桌。厨房什么时候做好了，服务员再回来上菜。

对应到代码里：

- `async def` 定义一个协程（Coroutine），它是一段可以暂停、恢复执行的函数。
- `await` 就是告诉事件循环：“这里需要等 IO，你先去干别的事，等结果回来再继续我。”
- `asyncio.gather()` 同时发起多个协程，总耗时约等于最慢的那个，而不是累加。

听起来很美好，对吧？但有个致命前提：**你等的东西必须支持异步**。如果你 `await` 的是一个同步阻塞调用，事件循环就被你卡死了，其他协程根本没法切换进来。

我当时犯的错，就出在这里。

## 错误写法：为啥加了 async/await 还是串行？

下面这种代码，几乎每个刚学 asyncio 的人都写过（包括我曾经沾沾自喜的版本）：

```python
import asyncio
import time
import requests

async def fetch(url):
    # 注意：requests.get 是同步阻塞的！
    resp = requests.get(url, timeout=10)
    return resp.status_code

async def main():
    urls = [f"https://httpbin.org/delay/1?i={i}" for i in range(10)]
    start = time.perf_counter()
    # 同时发起“协程”
    results = await asyncio.gather(*[fetch(url) for url in urls])
    print(f"耗时: {time.perf_counter() - start:.2f}s, 结果: {results}")

asyncio.run(main())
```

你期望的是 10 个请求同时发出，总耗时接近 1 秒。实际跑下来，耗时 10 秒以上，跟串行一模一样。为什么？

因为 `requests.get()` 是同步阻塞调用。当事件循环执行第一个协程，它把控制权交给 `fetch`，`fetch` 里直接调了 `requests.get()`，整个线程就被这个 `get` 调用阻塞在那里，直到网络返回。期间事件循环拿不回控制权，其他协程全在排队，并发名存实亡。

**asyncio 的并发是“用户态协作式”的，不是抢占式的**。你要是不主动 `await` 一个异步对象把控制权交出去，谁都救不了你。

## 正确写法：全链路异步，看这 10 行就够了

要把并发真正跑起来，需要把整个调用链都换成正儿八经的异步库。HTTP 请求用 `aiohttp`，文件 IO 用 `aiofiles`，数据库用 `asyncpg` / `aiomysql`。一秒都不要让同步阻塞混进来。

下面是生产可用的正确写法，加了并发限制、超时控制、异常处理：

```python
import asyncio
import time
import aiohttp
from asyncio import Semaphore

# 限制最大并发数，防止把目标服务器打爆 / 连接池耗尽
MAX_CONCURRENT = 20
semaphore = Semaphore(MAX_CONCURRENT)

async def fetch(session: aiohttp.ClientSession, url: str):
    async with semaphore:  # 超过上限的协程会在这里阻塞等待
        try:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as resp:
                data = await resp.text()
                return len(data)  # 返回内容长度，示例用
        except Exception as e:
            print(f"请求失败 {url}: {e}")
            return None

async def main():
    urls = [f"https://httpbin.org/delay/1?i={i}" for i in range(50)]
    start = time.perf_counter()
    
    # 创建可复用的连接池，减少握手开销
    connector = aiohttp.TCPConnector(limit=MAX_CONCURRENT)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    
    print(f"并发 {MAX_CONCURRENT}，共 {len(urls)} 个请求，"
          f"耗时: {time.perf_counter() - start:.2f}s，"
          f"成功: {sum(1 for r in results if r is not None)}")

asyncio.run(main())
```

运行效果：50 个请求，每个服务端延迟 1 秒，总耗时约 2.5~3 秒（50/20 批次的叠加），对比串行 50 秒，提速 15～20 倍。这才是 asyncio 该有的样子。

关键点解释：

- **`aiohttp.ClientSession`** 内部维护连接池，用了 `async with` 保证退出时自动关闭，避免文件描述符泄漏。
- **`Semaphore`** 做并发限制，这是对上服务保护、对下不触发反爬的基础操作。不要无脑 `gather` 1000 个协程。
- **`asyncio.gather()`** 返回的是所有任务的结果列表，如果有任务抛异常，`gather` 默认会直接传播，生产环境最好加 `return_exceptions=True` 或外层 try。

## 踩坑 & 注意事项：血泪换来的几条铁律

这些坑不是看文档能看出来的，都是我在真实项目里一个个踩过来的：

**1. 协程对象 ≠ 执行**
`fetch(url)` 只是创建了一个协程对象，什么都没发生。必须 `await` 或者 `asyncio.create_task()` 把它丢进事件循环才算开始干活。这个低级错误我犯过不止一次——盯着日志等半天，结果协程根本没被调度。

**2. `asyncio.sleep()` 不等于 `time.sleep()`**
在协程里写 `time.sleep(1)` 是自杀，整个线程睡死，事件循环停摆。必须用 `await asyncio.sleep(1)`，它会把控制权还给循环。

**3. `asyncio.run()` 只能调一次**
它内部会创建事件循环、运行到结束、再关闭循环。如果你的脚本里多处调用 `asyncio.run()`，第二次会报 “Event loop is already running” 或者创建新循环导致状态混乱。要么只调一次，要么用 `asyncio.get_event_loop()` 手动管理。

**4. 别在 Jupyter / IPython 里 native 运行 asyncio**
Jupyter 自己跑着一个事件循环，你再用 `asyncio.run()` 会冲突。正确姿势是用 `await` 在 cell 顶层直接执行（IPython 支持），或者用 `nest_asyncio` 打补丁，但千万别在生产环境用这个补丁。

**5. `aiohttp` 连接池耗尽**
当你忘记设置 `TCPConnector(limit=xxx)`，默认连接数 100，但如果你开了 1000 个协程同时去连同一个 host，后 900 个就会因为连接池满了被阻塞在连接获取上，报 timeout。`limit` 建议和 `Semaphore` 保持一致。

**6. 不要在异步里偷偷同步**
项目中难免调用第三方 SDK，如果它内部用的是 `requests`，你就不能直接放在协程里。解决方案：用 `asyncio.to_thread()` 把同步调用丢到线程池去跑，不阻塞事件循环。但注意线程安全。

---

## 总结

**asyncio 不是银弹，但当你把所有阻塞点都铲干净之后，它能让你一个线程干出多线程的活，还不用操心 GIL。** 记住：一条同步调用就能毁掉你精心设计的异步架构，全链路非阻塞才是唯一正解。

#Python #asyncio #异步编程 #爬虫 #性能优化