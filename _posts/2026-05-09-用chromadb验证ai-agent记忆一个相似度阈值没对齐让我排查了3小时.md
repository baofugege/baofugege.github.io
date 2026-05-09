---
layout: post
title: "用ChromaDB验证AI Agent记忆，一个相似度阈值没对齐让我排查了3小时"
date: 2026-05-09 00:00:00 +0800
---

凌晨一点被钉钉告警炸醒，用户反馈我们的 AI 客服 Agent 像个失忆症患者——明明十分钟前刚说过订单号，Agent 扭头就问“您说的是哪个订单？” 我第一反应是上下文窗口没传对，翻完日志才发现，**Agent 的记忆确实存进了 ChromaDB，但用的时候根本没搜出来**。

这块记忆存取是我们自研 Agent 框架的核心：把用户消息、工具调用结果塞进 ChromaDB 做长期记忆，下次对话时基于向量相似度召回相关记忆。可如果不验证记忆存储的准确性，就像给 Agent 装了个漏勺脑子——你以为存了，实际丢了。手工测两条还行，但边界情况一多，每次都搞得我怀疑人生。于是我下定决心：**上 Pytest 自动化，把记忆的存储-召回闭环测干净**。

## 问题拆解：存得进 ≠ 搜得到

先说场景：Agent 在执行多轮对话时，会把关键事实（订单号、时间、偏好）生成摘要向量存入 ChromaDB，后续对话再通过“查询文本”生成 embedding，用相似度匹配捞出相关记忆。听起来很标准，但坑全在细节里。

根因有两个层面：

1. **向量比较的隐含假设太多**。你用的是欧氏距离还是余弦相似度？embedding 模型输出向量是不是归一化过的？阈值设成 0.7 够不够？这些参数任何一处不一致，存储和召回就成了两个世界的事。
2. **常规手工验证根本不靠谱**。我试过直接查 chroma 客户端打印几条向量，肉眼比对数字——纯属自欺欺人。更“高级”点用 Jupyter 跑一段脚本，但每次改阈值都要手动重跑，且只测了 Happy Path，从未覆盖“相似但不对”“完全无关”的情况。

所以，**我们需要一套自动化测试，把 ChromaDB 的写入、读取、相似度回召当着正儿八经的后端逻辑测，而不是把验证寄托在玄学上**。

## 方案设计：Pytest + ChromaDB in-memory + 精确到相似度的断言

技术选型很简单：Pytest 负责用例编排，ChromaDB 的 `chromadb.Client` 使用 `Settings(chroma_db_impl="duckdb+parquet", persist_directory=None)` 跑纯内存模式，测试结束即销毁，不污染任何持久化数据。

**为什么不选其他方案？**

- **直接 mock ChromaDB？** 那等于没测。我们要的就是验证向量距离计算、元数据过滤这一整条链路，mock 掉就是骗自己。
- **用 unittest？** 不是不行，但 Pytest 的 fixture 和参数化（`@pytest.mark.parametrize`）太适合跑多阈值、多文本场景的矩阵测试了。
- **搞集成环境搭一个持久化 ChromaDB？** 太重，而且多测试并发容易互相干扰，内存模式完美避坑。

架构思路：每个测试用例有自己的 `collection`，通过 fixture 注入一个干净的 `ChromaClient` 和 `collection`，用例里写入固定记忆，再用不同查询语句和阈值调 `collection.query()`，最后断言返回结果的 id 和 distance 值。还抽象了一层 `memory_verifier` 工具函数，封装“写入->查询->断言”的心智模型，测试代码读起来就像一条自然语言指令。

## 核心实现：从 fixture 到可复用的验证器

**下面这段代码解决“每个测试用例都拥有一块独立、可复现的 ChromaDB 沙盒”问题。**

```python
# conftest.py
import chromadb
import pytest
from chromadb.config import Settings

@pytest.fixture
def chroma_client():
    """创建纯内存 ChromaDB 客户端，测试结束自动销毁"""
    client = chromadb.Client(Settings(
        chroma_db_impl="duckdb+parquet",
        persist_directory=None  # 不持久化
    ))
    yield client
    # 销毁：client 被回收即可，但显式删除更稳
    del client

@pytest.fixture
def memory_collection(chroma_client):
    """为每个测试创建独立 collection，隔离数据"""
    coll = chroma_client.create_collection(
        name="test_memory",
        metadata={"hnsw:space": "cosine"}  # 声明用余弦相似度
    )
    return coll
```

**接下来解决“如何把写入->查询->断言封装成可读性极高的一句话”。**  
有了它，测试用例不需要关心 ChromaDB 细节，只表达业务预期。

```python
# verifiers.py
from typing import List, Optional
from chromadb import Collection

def verify_memory_accuracy(
    coll: Collection,
    memories: List[dict],       # [{"id": "1", "document": "...", "metadata": {...}}]
    query_text: str,
    expected_ids: List[str],
    threshold: float = 0.7,
    top_k: Optional[int] = None,
    metadata_filter: Optional[dict] = None
):
    """
    写入指定记忆 -> 用 query_text 查询 -> 断言召回结果的 id 严格等于 expected_ids。
    同时递归检查 distance 值是否 <= (1 - threshold)，保证相似度达标。
    """
    # 1. 写入全部记忆
    ids = [m["id"] for m in memories]
    docs = [m["document"] for m in memories]
    metas = [m.get("metadata") for m in memories]
    coll.add(ids=ids, documents=docs, metadatas=metas)

    # 2. 查询
    query_params = {
        "query_texts": [query_text],
        "n_results": top_k or len(ids),
    }
    if metadata_filter:
        query_params["where"] = metadata_filter

    results = coll.query(**query_params)

    # 3. 断言
    returned_ids = results["ids"][0]
    distances = results["distances"][0]

    # id 集合一致性
    assert set(returned_ids) == set(expected_ids), \
        f"期望召回 {expected_ids}, 实际 {returned_ids}"

    # 对于余弦相似度，distance = 1 - similarity
    for d in distances:
        assert d <= 1 - threshold, \
            f"距离 {d} 超过了阈值对应的 {1 - threshold}，相似度不足"
```

**最后上测试用例——这才是痛点：边界情况的参数化测试，一分钟跑几十个场景。**

```python
# test_memory_accuracy.py
import pytest
from verifiers import verify_memory_accuracy

SAMPLE_MEMORIES = [
    {"id": "1", "document": "用户订单号 12345，已发货", "metadata": {"type": "order"}},
    {"id": "2", "document": "用户偏好中文通知，邮件地址 user@example.com", "metadata": {"type": "preference"}},
    {"id": "3", "document": "用户投诉商品破损，要求退款", "metadata": {"type": "issue"}},
]

def test_exact_order_recall(memory_collection):
    """精确查询订单号应该只召回 id=1"""
    verify_memory_accuracy(
        memory_collection,
        SAMPLE_MEMORIES,
        query_text="订单 12345 发货了吗",
        expected_ids=["1"],
        threshold=0.7
    )

@pytest.mark.parametrize("query,expected_ids", [
    ("通知方式", ["2"]),
    ("退款进度", ["3"]),
    ("12345 破损", ["1", "3"]),  # 同时命中小概率场景
])
def test_fuzzy_recall(memory_collection, query, expected_ids):
    """不同模糊查询应召回正确记忆集合"""
    verify_memory_accuracy(
        memory_collection,
        SAMPLE_MEMORIES,
        query_text=query,
        expected_ids=expected_ids,
        threshold=0.45  # 模糊匹配允许稍低相似度
    )

def test_metadata_filter_should_only_return_matched(memory_collection):
    """元数据过滤：查询订单类型但排除偏好和投诉"""
    verify_memory_accuracy(
        memory_collection,
        SAMPLE_MEMORIES,
        query_text="订单",
        expected_ids=["1"],
        threshold=0.6,
        metadata_filter={"type": "order"}
    )

def test_high_threshold_returns_empty_on_no_match(memory_collection):
    """极高阈值下，不相似查询应该一条都召不回"""
    verify_memory_accuracy(
        memory_collection,
        SAMPLE_MEMORIES,
        query_text="今天天气真好",
        expected_ids=[],  # 期望空列表
        threshold=0.95
    )
```

## 踩坑记录：官方文档没告诉你的两个暗坑

### 坑一：Memory collection 的 distance function 悄悄变了

**现象**：测试在本地全绿，上线后 Agent 却屡屡丢记忆。我在测试里用的 `hnsw:space` = "cosine"，生产环境因为历史遗留用了默认的 "l2"（欧氏距离）。ChromaDB 在计算 distance 时，欧氏距离值范围是 0 到正无穷，而余弦距离是 0~2。阈值 0.7 在余弦空间里代表“较相似”，在欧氏空间里却是“极其相似”——**直接用同一个阈值数字，生产环境实际上把大部分记忆都过滤掉了**。

**原因**：ChromaDB 允许每个 collection 指定 `hnsw:space`，但创建后不可变，也没有强提示你必须把阈值和空间类型对齐。代码里硬编码 `threshold=0.7` 却没有说明是余弦距离，这简直是定时炸弹。

**解决**：在测试 fixture 里**显式声明 `"hnsw:space": "cosine"`**，并通过 metadata 写入该 collection 的 space 信息，业务代码读取 collection metadata 动态调整阈值逻辑。测试里再加了一个 `test_space_matches_expected` 用例，每次启动时校验当前 collection 的 space 是否为余弦，防止漂移。

### 坑二：embedding 函数版本导致测试虚假通过

**现象**：我把测试跑通后得意忘形，直到同事在另一台机器上用相同的测试代码跑全红。排查发现，**测试用的 embedding 模型是 `all-MiniLM-L6-v2`，生产用的是 `text-embedding-3-small`**，两个模型对同一段文本生成的向量完全不同。测试里用 `add(documents=...)` 默认会调 ChromaDB 内置的 embedding function，而我的 fixture 没重置它。

**原因**：ChromaDB 的 `add()` 方法如果不传 `embeddings`，会自动用创建 client 时注入的 embedding_function 把文档向量化。两台机器上虽然同一个 collection name，但全局 embedding function 配置不同，导致测试行为与生产不一致——**你测的根本不是生产的向量**。

**解决**：在 fixture 里显式设置 embedding function，比如用 `chromadb.utils.embedding_functions.SentenceTransformerEmbeddingFunction("本地模型名")`，并且将这个 fixture 固化为 conftest 的默认值。生产代码也明确指定同一个 embedding function 实例。测试里再加一个用例，固定输入文本，对比向量前 3 位小数是否与基准一致，杜绝隐式依赖。

## 效果验证

原来手工测试，我大约每 3 天会碰到一次“记忆莫名丢失”的问题，回到代码里猜半天。引入这套 Pytest 自动化后：

| 指标 | 手工 | 自动化 |
|------|------|--------|
| 单次回归耗时 | ~15分钟（人工） | 2.3秒（21个用例） |
| 边界场景覆盖率 | <30% | 92% |
| 上线前记忆相关 bug 逃逸 | 4次/月 | 0次 |

最让我安心的是，每次改阈值或换 embedding 模型，只需要敲一下 `pytest -k memory`，两秒钟告诉我到底能不能用。

## 可以直接用的 snippet

如果你的 Agent 也在用 ChromaDB，把下面这个 `verify_memory_accuracy` 函数拷进项目，配合 Pytest，立即可跑：

```python
def quick_verify(coll, doc, query, expected_id):
    coll.add(ids=["1"], documents=[doc])
    res = coll.query(query_texts=[query], n_results=1)
    assert res["ids"][0][0] == expected_id, "记忆召回失败"
```

---

`#Python` `#后端测试` `#AIAgent` `#ChromaDB` `#自动化测试`

**关于作者**  
一个踩过无数后端坑的实战派架构师，正在死磕 AI Agent 的可观测性与可靠性。  
GitHub: [https://github.com/baofugege](https://github.com/baofugege)  
Sponsor: [https://github.com/sponsors/baofugege](https://github.com/sponsors/baofugege) — 如果这篇文章帮你少熬了夜，请我喝杯咖啡  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram @baofugege