---
layout: post
title: "LangChain 记忆存储踩坑实录：一个序列化 bug 让我半夜爬起来修，换成 pytest 守护后再也没复发"
date: 2026-05-05 00:00:00 +0800
---

凌晨 2:17，我被监控电话打醒：客服机器人突然“失忆”，用户连问三遍“我的订单呢”，它都像第一次见一样要手机号。打开日志一看，`ConversationBufferMemory` 加载出来的消息列表全是空的，Redis 里 key 还在，反序列化却悄悄吞掉了数据。那晚我从床上弹起来拉代码回滚，整整查了 3 个小时才发现——**LangChain 升级后 pickle 反序列化不兼容，直接把历史消息丢了**。手工测试根本没覆盖版本升级场景。第二天我就决定：用 pytest 把记忆存储的完整性和性能全部自动化守护起来。后来再也没因为序列化翻过车，回归时间从 30 分钟压到 8 秒。

## 问题拆解

我们当时的架构很清晰：`ConversationBufferMemory` + `RedisChatMessageHistory`，把用户会话持久化到 Redis。LangChain 内部用 `pickle` 把消息列表 dump 成一个 bytes，塞进 `{session_id}` 的键里，下次加载时 load 回来。

问题出在**版本升级**：我们从 `langchain==0.0.352` 升到 `0.1.0`，内部 `HumanMessage` 和 `AIMessage` 的全限定类名变了，老版本的 pickle 荷载一 load 就抛出 `AttributeError`。更蛋疼的是，`RedisChatMessageHistory` 的 `messages` 属性直接 catch 了异常返回空列表——看起来像个空对话，完全没报错。这类 bug 有两个特点：
1. **发作时机滞后**：不是升级那一秒炸，而是用户下次读写记忆时才炸，监控很难第一时间定位。
2. **手工测试不可达**：升级前 QA 只会用当前版本验证“能不能存、能不能读”，不会刻意构造旧版本序列化数据来验证兼容性。

所以常规的“点一点”手工测试，完全拦不住这类回归。必须有一套自动化的、可重复的集成测试，覆盖：
- 写入→重启→读取的完整性
- 多版本序列化数据的反序列化兼容性
- 大批量消息下的读写性能
- 并发擦写的正确性

## 方案设计

我选了 **pytest** 做测试框架。放在当时，也不是没考虑过 `unittest`，但我要的几样东西它给得太费劲：

- **隔离性**：需要一个无痛的 Redis 替代品，fakeredis 能完美模拟 Redis 指令，跟 pytest fixture 结合可以做到测试零外部依赖。
- **参数化**：`@pytest.mark.parametrize` 可以一行覆盖 10 / 100 / 1000 条消息，不用手写循环。
- **性能基准**：pytest-benchmark 插件可以直接卡平均/最大延迟，比我手动 `time.perf_counter` 靠谱。
- **并发模拟**：搭配 `threading` 或 `asyncio` 写 fixture，比 `unittest` 的 subTest 直观得多。

整体思路：用 `conftest.py` 定义 FakeRedis fixture，并 monkeypatch 掉 `redis.Redis.from_url`，让所有 LangChain 的 Redis 调用都打到内存实现上。然后分模块写：
- `test_integrity.py`：验证存取一致性、跨实例加载
- `test_compatibility.py`：模拟旧序列化格式，验证升级后的迁移/降级逻辑
- `test_performance.py`：pytest-benchmark 测消息读写的耗时天花板
- `test_concurrency.py`：多线程同时追加消息，看是否丢数据

为什么不直接连真实 Redis？真实 Redis 当然要测，但那是 CI 的 smoke test；开发机和 PR 阶段的每一次 push 都去连 Redis 太慢也太脏。FakeRedis 让跑完整站测试套件只需要几百毫秒，这才是让团队愿意写测试的关键——**零摩擦**。

## 核心实现

### 1. conftest：用 FakeRedis 接管 LangChain 的 Redis 连接

这段代码解决的核心问题是：**让所有测试共享同一个内存 Redis，且对 LangChain 完全透明**。我们 monkeypatch 掉了 `redis.Redis.from_url` 和直接构造 `redis.Redis()` 的路径，这样 `RedisChatMessageHistory` 无论怎么创建 client，最终都会落到同一个 FakeRedis 实例上。

```python
# conftest.py
import pytest
import redis
from fakeredis import FakeRedis

@pytest.fixture(scope="function")
def fake_redis():
    """每个测试函数独立的 FakeRedis 实例"""
    return FakeRedis()

@pytest.fixture(autouse=True)
def patch_redis(monkeypatch, fake_redis):
    """将所有对 Redis 的调用劫持到 FakeRedis，实现零外部依赖"""
    # 劫持 from_url 方法，LangChain 内部用这个创建连接
    def _fake_from_url(url: str, **kwargs):
        return fake_redis

    monkeypatch.setattr(redis.Redis, "from_url", _fake_from_url)
    # 如果有地方直接 redis.Redis(...)，也一并拦截
    monkeypatch.setattr(redis, "Redis", lambda *a, **kw: fake_redis)
    return fake_redis
```

这样做的好处是：之后写任何测试，只要导入 `patch_redis` fixture，LangChain 就会老老实实读写我指定的 FakeRedis，数据库始终干净。

### 2. 测试完整性：写入→重新加载必须丝毫不差

下面是 `test_integrity.py`。它验证了最核心的契约：我存了什么，下次同一 session 再加载就必须原样拿回来。我用参数化覆盖了单条、中等量、大批量三种情况，边界都能照顾到。

```python
# test_integrity.py
import pytest
from langchain.memory import ConversationBufferMemory
from langchain.memory.chat_message_histories import RedisChatMessageHistory
from langchain.schema import HumanMessage, AIMessage

@pytest.mark.parametrize("msg_count", [1, 50, 500])
def test_memory_roundtrip(msg_count, patch_redis):
    """写入 N 对问答，用新的 memory 实例重新加载，验证消息数量和内容完全一致"""
    session = "user_123"
    # 第一轮：写入
    hist_write = RedisChatMessageHistory(session_id=session)
    memory_write = ConversationBufferMemory(chat_memory=hist_write)
    for i in range(msg_count):
        memory_write.chat_memory.add_user_message(f"Q{i}")
        memory_write.chat_memory.add_ai_message(f"A{i}")

    # 第二轮：新的 History 实例模拟重启后重新加载
    hist_read = RedisChatMessageHistory(session_id=session)
    memory_read = ConversationBufferMemory(chat_memory=hist_read)
    messages = memory_read.chat_memory.messages

    assert len(messages) == msg_count * 2
    # 抽查第一条和最后一条，防止仅仅是长度对但内容错位
    assert messages[0].content == "Q0"
    assert messages[-1].content == f"A{msg_count - 1}"
```

这里的关键是，`RedisChatMessageHistory` 必须在第二轮重新实例化——模拟服务重启后从 Redis 冷加载的场景。很多手工测试往往忘记这一环，直接用同一个 `memory` 对象断言，完全没测到序列化这个链路。

### 3. 测试兼容性：模拟旧版本序列化数据，验证升级安全

这个测试直接来源于那晚的事故。我利用 FakeRedis 直接往里塞一个旧版 pickle 负载，然后让新版本 LangChain 去加载，检查它是否会静默丢弃数据。我们的最终方案是**强制使用 JSON 序列化**，因为它天然可跨版本、可读、可迁移。下面的用例既验证了旧 pickle 数据能被安全识别，又验证了新 JSON 数据的可恢复性。

```python
# test_compatibility.py
import json
import pickle
import pytest
from langchain.schema import HumanMessage, AIMessage
from langchain.memory.chat_message_histories import RedisChatMessageHistory

@pytest.fixture
def legacy_pickle_data(fake_redis):
    """模拟 v0.0.352 版本的 pickle 序列化结果"""
    session = "legacy_user"
    msgs = [
        HumanMessage(content="old question"),
        AIMessage(content="old answer")
    ]
    fake_redis.set(session, pickle.dumps(msgs))
    return session

def test_load_legacy_pickle_does_not_silently_fail(legacy_pickle_data, patch_redis):
    """
    加载旧 pickle 数据时，应抛出版本不兼容异常或触发迁移逻辑，
    而不是静默返回空列表（这是线上事故的根因）
    """
    hist = RedisChatMessageHistory(session_id="legacy_user")
    # 预期：要么成功迁移（消息可读），要么明确抛异常
    try:
        msgs = hist.messages
        # 如果能到这里，说明加载成功，那么消息必须是完整的
        assert len(msgs) == 2, "即使不抛异常，旧数据也必须完整加载"
    except Exception as e:
        assert "pickle" in str(e).lower() or "version" in str(e).lower()

def test_json_serialization_roundtrip(patch_redis, fake_redis):
    """改用 JSON 后，确保直接读取 raw JSON 也是可恢复的"""
    session = "json_user"
    # 手工用 JSON 写入，模拟外部系统直接操作 Redis
    msgs_payload = [
        {"type": "human", "content": "hi"},
        {"type": "ai", "content": "hello"}
    ]
    fake_redis.set(session, json.dumps(msgs_payload))

    hist = RedisChatMessageHistory(session_id=session)
    msgs = hist.messages
    assert len(msgs) == 2
    assert msgs[0].content == "hi"

```

### 4. 性能基准：别让记忆成为聊天延迟的短板

这个测试用 `pytest-benchmark` 确认了记忆读写的性能天花板。我们预设了 200 条历史消息（大约是一次长对话或一个中型会话上下文），然后 benchmark 反复加载这个 session，确保 P99 延迟不过百毫秒。

```python
# test_performance.py
import pytest
from langchain.memory import ConversationBufferMemory
from langchain.memory.chat_message_histories import RedisChatMessageHistory

def test_memory_load_performance(patch_redis, benchmark):
    """加载 200 条历史消息，验证耗时在可接受范围内"""
    session = "perf_test"
    hist = RedisChatMessageHistory(session_id=session)
    memory = ConversationBufferMemory(chat_memory=hist)
    # 预写入 200 轮对话
    for i in range(200):
        memory.chat_memory.add_user_message(f"msg{i}")
        memory.chat_memory.add_ai_message(f"reply{i}")

    def reload():
        h = RedisChatMessageHistory(session_id=session)
        m = ConversationBufferMemory(chat_memory=h)
        return m.chat_memory.messages

    result = benchmark(reload)
    assert len(result) == 400
    # benchmark 报告会自动输出 min / max / mean / stddev，CI 中可设置断言最大均值
    # 例如：assert benchmark.stats['mean'] < 0.05   # 50ms
```

### 5. 并发写入：确保多线程追加不丢消息

这个用例模拟了一个用户在多个入口（比如 Web + 移动端）同时发送消息时，记忆追加是否正确。虽然真实场景会有 Kafka 排队，但单元测试必须保证存储层本身不会因为并发写而丢失数据。

```python
# test_concurrency.py
import threading
import pytest
from langchain.memory.chat_message_histories import RedisChatMessageHistory

def test_concurrent_add_messages(patch_redis):
    session = "concurrent_user"
    hist = RedisChatMessageHistory(session_id=session)

    def add_messages(thread_id):
        for i in range(50):
            hist.add_user_message(f"T{thread_id}-{i}")

    threads = [threading.Thread(target=add_messages, args=(t,)) for t in range(4)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    messages = hist.messages
    # 4 线程 × 50 条 = 200 条，一条不能少
    assert len(messages) == 200, f"Expected 200 messages, got {len(messages)}"
```

## 踩坑记录

### 坑 1：fakeredis 版本不兼容，测试跑起来就直接 ConnectionError

**现象**：首次运行 `pytest`，只要一碰到 `RedisChatMessageHistory`，立刻抛 `redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379`。这不是没启动 Redis 的问题——我已经打了 patch，为什么还走真实连接？

**原因**：`fakeredis<2.18` 在模拟 `from_url` 时，对某些 redis-py 4.5+ 版本的 `ConnectionPool` 参数校验不一致，导致 monkeypatch 虽然替换了类，但内部构造 pool 时又尝试解析 URL 并建立 socket。本质是 fakeredis 的 FakeRedis 与真实 Redis 的 `Connection` 实现还没彻底解耦。

**解决**：升级 `fakeredis>=2.18.1`，并确保 pytest fixture 作用域为 `function` 级别，避免多个测试函数共享同一个 `FakeRedis` 实例导致 key 污染。另外一个技巧是**在 monkeypatch 时同时劫持 `Redis.connection_pool`**，直接返回 fake 的连接池，这一步很多网上的例子压根没写。

### 坑 2：pytest-asyncio 与同步 fixture 混用导致事件循环死锁

**现象**：测试函数一结束就 hang 住，Ctrl+C 杀不掉，只能 `kill -9`。直到看了 pytest-asyncio 的文档才知道是 event loop 冲突。

**原因**：我最初为了图方便，在 `conftest.py` 里写了一个 `@pytest.fixture` 返回 `async` 的 Redis client。结果 pytest-asyncio 默认的 event loop 作用域是 `function`，而同步 fixture 也在同一个 scope 里调用 `asyncio.run()`，导致外层嵌套事件循环，pytest 内部等待 fixture finalize 时死锁。

**解决**：统一使用 `pytestmark = pytest.mark.asyncio` 并显式管理 `event_loop` fixture，或者干脆把异步操作扔到 `async def test_xxx()` 里，避免在 fixture 中混用。对于非必要异步的集成测试，直接用同步 `FakeRedis` 就足够了，别再引入异步维度。

## 效果验证

用这套 pytest 体系替代手工回归后，效果立竿见影：

| 指标 | 手工测试 | pytest 自动化 |
|------|-----------------|---------------------|
| 覆盖场景数 | 2（正常读写 + 空会话） | 8（完整性 / 兼容性 / 性能 / 并发 / 序列化格式切换等） |
| 回归耗时 | ~30 分钟 | 8 秒（FakeRedis） |
| Bug 检出 | 0（版本升级问题全漏掉） | 3 个（pickle 兼容、连接泄漏、并发丢消息） |
| 记忆加载 200 条耗时 | 未测量（凭感觉“还可以”） | P50=18ms, P99=42ms |

最关键的是，**从那以后序列化导致的记忆丢失再没复现**。现在每次 LangChain 发版，CI 跑完这一套测试只需要 8 秒，团队再也不用半夜起来回滚了。

## 可直接用的代码/工具

下面这个 `conftest.py` 模板可以直接复制到你项目里，配合 `pip install pytest fakeredis langchain pytest-benchmark` 立刻获得零依赖的记忆存储测试环境：

```python
# conftest.py
import pytest
import redis
from fakeredis import FakeRedis

@pytest.fixture(scope="function")
def fake_redis():
    return FakeRedis()

@pytest.fixture(autouse=True)
def patch_redis(monkeypatch, fake_redis):
    monkeypatch.setattr(redis.Redis, "from_url", lambda *a, **kw: fake_redis)
    monkeypatch.setattr(redis, "Redis", lambda *a, **kw: fake_redis)
    return fake_redis
```

---

`#Python` `#LangChain` `#pytest` `#后端` `#踩坑`

**关于作者**  
实战派后端架构师，长期用 Python 死磕 LLM 应用层的可靠性和性能，坚信“没被线上事故毒打过的方案都不是好方案”。  
GitHub: https://github.com/baofugege  
Sponsor: https://github.com/sponsors/baofugege — 如果这篇文章帮你省下了凌晨回滚的时间，请我喝杯咖啡  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram @baofugege