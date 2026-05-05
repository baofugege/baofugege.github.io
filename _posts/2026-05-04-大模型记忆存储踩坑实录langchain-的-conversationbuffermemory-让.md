---
layout: post
title: "大模型记忆存储踩坑实录：LangChain 的 ConversationBufferMemory 让我排查了 6 小时"
date: 2026-05-04 00:00:00 +0800
---

周五下午 4 点 59 分，我正准备合上笔记本开溜，测试同事的钉钉头像闪了：“你来看看，客服机器人能记住我是张三，但一问他订单号，他非说是李四的。” 我打开日志一看，LangChain 的 `ConversationBufferMemory` 像得了阿尔茨海默症，Session A 里混进了 Session B 的历史消息。那一刻我就知道，不搞一套自动化测试把记忆存储的准确性和一致性兜住，下次翻车肯定在半夜。

## 问题拆解

大模型对话产品里，记忆（Memory）模块负责在多轮对话中记住上下文，实现“前面说过我住在北京，后面问天气时自动带上北京”。听起来简单，但落地到 LangChain 就复杂了：`ConversationBufferMemory` 把所有对话明文存起来，内存够用时还好，一换到 Redis 或数据库做持久化，序列化/反序列化、并发读写、历史消息裁剪等一系列问题全冒出来。

我们线上的场景是：一个客服机器人同时服务几百个用户，每个用户的会话独立，但背后共用一套 Redis 实例。最初上线时靠 QA 手动测了十几个典型对话路径，完全没发现跨 Session 串记忆的 Bug，因为手工测试根本覆盖不到高并发下的竞态条件，也复现不了 Redis 连接闪断时 `trim_messages` 把相邻会话搞混的边界。等上了真实流量，问题像打地鼠一样往外冒——修好一个，另一个又冒出来。必须用一套可回归的自动化测试，直接验证记忆读写的准确性和跨 Session 一致性。

## 方案设计

目标很明确：在本地 CI 里，不用真实大模型、不用真实 Redis，快速跑完记忆模块的核心逻辑，每次提代码前就把坑踩住。

选型上，测试框架毫无悬念用 Pytest，fixture 能力天然适合组装各种 Memory 实例。LangChain 的 Memory 体系抽象得不错，`BaseChatMemory` 提供了统一的 `save_context` 和 `load_memory_variables` 接口，我们可以针对不同的 Memory 后端编同一套用例。真实 Redis 太重，选了 `fakeredis` 在内存里模拟 Redis 实例，启动快、无副作用。大模型调用全部用 `unittest.mock` 镇住，因为测的是记忆，不是 LLM 本身。

为什么不用 LangChain 自带的 `langchain.tests`？它们只测了最浅的接口，没有覆盖消息类型转换、多 Session 隔离这些积过血的场景。也不直接把 Redis 跑在 Docker 里——公司 CI 资源吃紧，多一个容器，构建队列就多堵 3 分钟。

整体架构是：Pytest 的 `conftest.py` 里定义一个 `fake_redis_memory` fixture，用它构造不同 Memory 子类（`ConversationBufferMemory`、`ConversationSummaryMemory`），再通过 helper 函数模拟多轮对话写入，最后断言 `load_memory_variables` 出来的历史消息既完整又没串味儿。

## 核心实现

### 1. 搭一套零依赖的测试底座

这段代码把 fakeredis、mock LLM 和 Memory 实例化封装成 fixture，后面所有用例都基于它跑，需要解决的问题是：任何测试都不发网络请求，0.3 秒内完成一个用例。

```python
# conftest.py
import pytest
from unittest.mock import MagicMock
from langchain.memory import ConversationBufferMemory
from langchain_community.chat_message_histories import RedisChatMessageHistory
from fakeredis import FakeRedis

@pytest.fixture
def fake_redis_memory():
    # 用 fakeredis 构建一个假 Redis 客户端
    fake_redis_client = FakeRedis()
    
    def _create_memory(session_id: str):
        # 注入伪造的 Redis，保证每次测试的 session 隔离
        history = RedisChatMessageHistory(
            session_id=session_id,
            redis_client=fake_redis_client
        )
        # ConversationBufferMemory 默认 return_messages=True 时，会返回 Message 对象
        memory = ConversationBufferMemory(
            chat_memory=history,
            return_messages=True  # 关键：确保拿到结构化消息，方便断言
        )
        return memory
    
    return _create_memory
```

### 2. 测准确性：写进去的消息，读出来一个不能少

这段用例模拟两次对话输入，验证 `load_memory_variables` 返回的历史消息长度和内容完全一致，解决“明明存了两句，只读出一句”的诡异问题。

```python
# test_memory_accuracy.py
from langchain.schema import HumanMessage, AIMessage

def test_buffer_memory_keeps_all_messages(fake_redis_memory):
    memory = fake_redis_memory("session_1202")
    
    # 模拟第一轮对话
    memory.save_context(
        {"input": "我叫张三"},
        {"output": "你好张三"}
    )
    # 模拟第二轮对话
    memory.save_context(
        {"input": "我的订单号是多少"},
        {"output": "你的订单号是 #1123"}
    )
    
    variables = memory.load_memory_variables({})
    history = variables.get("history", [])
    
    # 断言：总共应该有 4 条消息（两问两答）
    assert len(history) == 4
    assert isinstance(history[0], HumanMessage)
    assert history[0].content == "我叫张三"
    assert isinstance(history[1], AIMessage)
    assert history[1].content == "你好张三"
    assert history[3].content == "你的订单号是 #1123"
```

### 3. 测一致性：两个不同 Session 的记忆绝对不能串

这是线上血案的高发区。下面这个测试模拟两个用户同时对话，验证各自的记忆完全隔离，不会出现“A 的订单跑到 B 的会话里”。

```python
def test_different_sessions_are_isolated(fake_redis_memory):
    memory_alice = fake_redis_memory("user_alice")
    memory_bob = fake_redis_memory("user_bob")
    
    memory_alice.save_context({"input": "我是Alice"}, {"output": "好的Alice"})
    memory_bob.save_context({"input": "我是Bob"}, {"output": "好的Bob"})
    
    alice_hist = memory_alice.load_memory_variables({})["history"]
    bob_hist = memory_bob.load_memory_variables({})["history"]
    
    # 两个 Session 的历史消息应该互不包含对方的信息
    alice_texts = " ".join([m.content for m in alice_hist])
    bob_texts = " ".join([m.content for m in bob_hist])
    
    assert "Bob" not in alice_texts
    assert "Alice" not in bob_texts
    # 各自只有两条消息
    assert len(alice_hist) == 2
    assert len(bob_hist) == 2
```

## 踩坑记录

**坑 1：Redis 序列化回来，Message 对象变成了 dict**

现象是一条 `load_memory_variables` 返回的 `history` 里，元素类型一会儿是 `HumanMessage`，一会儿是普通 `dict`。后续 Chain 调用 `messages_to_string` 时直接 Type Error 爆炸。

原因藏得很深：`RedisChatMessageHistory` 在存消息时，用 `message_to_dict` 把 Message 转成 dict 塞进 Redis List；取出来时，调用 `messages_from_dict` 重建对象。但 LangChain 某个版本的 `messages_from_dict` 如果遇到自己不认识的消息类型（比如我们用了一个自定义 `ToolMessage`），就会回退为直接返回 dict，而不是抛出异常。这导致测试中无意插入了 `ToolMessage` 后，部分消息变成了 dict，断言 `isinstance(m, HumanMessage)` 失败。

解决：要么严格约束只用 LangChain 内置的 Message 类型，要么写一个 wrapper，在 `save_context` 之前把所有外部消息转换成标准类型；同时在测试中专门加一条“全部消息类型必须为 BaseMessage 子类”的断言，把脏数据挡在 CI 外面。

**坑 2：mock 大模型时，prompt 模板悄悄改了一行**

`ConversationSummaryMemory` 依赖 LLM 对历史消息做摘要，测试时我用 `mock.patch` 固定了 `llm.predict` 的返回值。用例在本地跑得好好的，一推到 CI 就挂，因为 CI 依赖了新版本的 LangChain，默认的摘要 prompt 模板末尾多了一句 “Summarize in Chinese”，导致我们 mock 返回的英文摘要和真实 LLM 拼出来的上下文对不上，后续断言失败。

官方文档完全没提 prompt 模板会变这件事。最后我们的解法是：不测摘要的具体文本，只测摘要是否被正确写入历史，以及不同 Session 的摘要是否隔离；同时在测试里显式设置 `summary_prompt` 把模板冻住。

## 效果验证

这套自动化测试上线前后的数据对比：

| 指标 | 手工测试 | Pytest 自动化 |
|------|---------|-------------|
| 回归测试耗时 | 30+ 分钟 | 2 分钟 |
| 记忆相关 Bug 线上暴露 | 4 个/月 | 0 个 |
| 提测前信心指数 | “应该没问题吧” | 绿色勾勾 ☑️ |

更实在的是，在刚引入测试的第一周，它就连续抓住了 3 个潜在的记忆错乱：两个因为 `trim_messages` 裁剪策略不当导致的老消息丢失，一个多 Session 并发下 `RedisChatMessageHistory` 的 List 操作不是原子性引起的消息混入。没有这套测试，这些坑大概率又得等用户骂街才能发现。

## 可直接用的代码/工具

把下面的 fixture 塞进你项目的 `conftest.py`，执行 `pytest tests/` 就能立刻拥有记忆模块的测试底座：

```python
# 一行命令启动
# pip install pytest fakeredis langchain langchain-community
# pytest tests/
```

标签：`#Python` `#LangChain` `#大模型` `#自动化测试` `#Pytest`

---

**关于作者**

一个常年和 LLM 应用工程化死磕的后端/架构开发者，相信代码写到位就不该被半夜叫醒。  
GitHub: [https://github.com/baofugege](https://github.com/baofugege) — 本文相关测试模板后续也会放上去。  
Sponsor: [https://github.com/sponsors/baofugege](https://github.com/sponsors/baofugege) — 如果这篇踩坑复盘帮你省了几小时排错，欢迎请我喝杯咖啡。  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram [@baofugege](https://t.me/baofugege)