---
layout: post
title: "Pytest + Docker 验证 AI Agent 记忆存储踩坑实录：这 3 个 Bug 让我排查了 8 小时"
date: 2026-05-06 00:00:00 +0800
---

凌晨 1:23，运维群突然炸了：用户投诉 Agent 完全失忆，上下文全丢，每一轮对话都像第一次见面。我翻日志发现记忆模块返回了空列表，可数据库里记录明明在。不是模型幻觉，是**记忆存储的一致性被偷偷打破了**——两个并发请求把 session 的最后一条消息覆盖成了旧版本，而单元测试里 mock 出来的 MemoryStore 永远不会告诉你这个。就是那晚我决定用 Pytest + Docker 搭一套可复现的真·存储验证，结果从半夜搞到天亮，踩的坑比预想的多得多。这篇复盘完整记录下来，让你少熬几次夜。

## 问题拆解：为什么 mock 测不出记忆的致命伤

AI Agent 的记忆存储通常不复杂：每次对话插入一行，带上 `session_id`、`role`、`content`、`created_at`，查询时按 session 取最近 N 条拼成上下文。我们用的是 PostgreSQL，代码里边用 SQLAlchemy + asyncpg，看起来人畜无害。可一旦并发上来，幺蛾子就冒出来：

- **并发插入顺序错乱**：不靠数据库自增序列，而是代码里生成 `created_at` 时间戳，但服务器时钟不同步或者 Python `datetime.utcnow()` 在协程里被调度打乱，导致后面的消息排到前面。
- **读写分离下的“写入消失”**：主库写入成功，但查询走了只读副本，主从延迟让刚写入的消息查不到，Agent 就“忘词”了。
- **事务隔离“假”快照**：默认 READ COMMITTED 下，一个长事务里多次读取同一 session，可能看到不同版本，上下文组装时出现幽灵行。

常规单元测试怎么测的？用 `unittest.mock` 把仓库类替换掉，断言“调用了 insert 方法”。这玩意根本碰不到真实存储引擎，隔离级别、并发调度、网络延迟全部凭空消失。**用 mock 测记忆存储，就好比在模拟器上练车，永远学不会侧方停车。**

## 方案设计：用 Docker 把真实数据库拉进测试里

要验证正确性与一致性，必须动真格的。方案很明确：**Pytest 负责组织用例，Docker 提供即抛即用的真实数据库。** 测试跑起来时，拉一个 PostgreSQL 容器，等它健康检查通过后建表、跑并发场景，最后一把销毁，每次都是纯净环境。

为什么不选其他方案？

- ❌ **Testcontainers-Python**：想法很好，但在 CI 里要连 Docker daemon，而且 API 抽象得不够透，出了问题你都不知道是容器没起还是端口映射错了。
- ❌ **SQLite 内存模式**：隔离级别和并发模型跟 PostgreSQL 差异太大，既测不出事务冲突，也模拟不了主从延迟，白测。
- ✅ **Docker Compose**：一个 yaml 描述依赖，CI/本地统一，运维怎么编排测试就怎么编排，能还原 90% 的生产行为。

架构思路如下图（文字描述）：

```
测试启动
  ├─ docker compose up -d (postgres, 可选: pgvector, redis)
  ├─ 等待健康检查通过
  ├─ 执行 alembic 迁移 / 建表
  ├─ pytest 用例 (正确性 + 并发一致性)
  └─ docker compose down -v
```

这里有一个容易被忽略的点：**并发测试必须在真实异步 IO 下跑，不能用 `pytest-asyncio` 的默认 loop 随便糊。** 我们必须控制事件循环的生命周期，让所有异步 fixture 共享同一个 loop，这样协程调度才和生产一致。

## 核心实现：一步步搭起来

先看 `docker-compose.yml`，只用最精简的配置，但健康检查一定要写对，不然后面踩死坑。

```yaml
# docker-compose.yml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: agent_test
      POSTGRES_PASSWORD: test_pass
      POSTGRES_DB: memory_test
    ports:
      - "0:5432"            # 随机端口，避免本地冲突
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "agent_test"]
      interval: 1s
      timeout: 3s
      retries: 10
```

端口映射写 `0:5432`，Docker 会分配一个宿主机随机端口，我们在 Python 里通过 `docker compose port` 命令拿这个端口，并行跑测试也不撞。

接着是 `conftest.py`，负责容器生命周期和数据库连接池。**这里踩的坑够多了，我直接把最终能跑的版本贴出来。**

```python
# conftest.py
import asyncio
import subprocess
import time

import asyncpg
import pytest
import pytest_asyncio


def _get_port() -> int:
    """通过 docker compose port 获取容器映射出来的宿主机端口"""
    result = subprocess.run(
        ["docker", "compose", "port", "postgres", "5432"],
        capture_output=True, text=True, check=True,
    )
    # 输出格式: "0.0.0.0:54321"
    return int(result.stdout.strip().split(":")[1])


@pytest.fixture(scope="session")
def docker_services():
    """启动 docker compose 服务，返回服务端口映射"""
    subprocess.run(["docker", "compose", "up", "-d"], check=True)
    # 等待健康检查通过，而不是死等 sleep
    for _ in range(30):
        try:
            port = _get_port()
            # 用 pg_isready 再次确认
            subprocess.run(
                ["pg_isready", "-h", "127.0.0.1", "-p", str(port),
                 "-U", "agent_test", "-d", "memory_test"],
                check=True, capture_output=True,
            )
            break
        except subprocess.CalledProcessError:
            time.sleep(0.5)
    else:
        raise RuntimeError("Postgres 容器未能在 15s 内就绪")
    yield {"postgres": port}
    subprocess.run(["docker", "compose", "down", "-v"], check=True)


@pytest.fixture(scope="session")
def event_loop():
    """必须手动创建 session 级别的事件循环，否则异步 fixture 会报错"""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest_asyncio.fixture(scope="session")
async def db_pool(docker_services, event_loop):
    """创建 session 级连接池，所有测试复用，避免频繁连接开销"""
    port = docker_services["postgres"]
    pool = await asyncpg.create_pool(
        host="127.0.0.1",
        port=port,
        user="agent_test",
        password="test_pass",
        database="memory_test",
        min_size=2,
        max_size=10,
    )
    # 建表（可换成 alembic upgrade head）
    async with pool.acquire() as conn:
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS agent_memory (
                id SERIAL PRIMARY KEY,
                session_id TEXT NOT NULL,
                role TEXT NOT NULL,
                content TEXT NOT NULL,
                created_at TIMESTAMPTZ DEFAULT now()
            );
            CREATE INDEX IF NOT EXISTS idx_session_time
                ON agent_memory(session_id, created_at DESC);
        """)
    yield pool
    await pool.close()
```

关键点：`event_loop` fixture 必须声明为 session scope，并手动 `new_event_loop()`，否则 `pytest-asyncio` 默认每个 function 独立 loop，session 级的异步 fixture 会报 `RuntimeError: Task <Task pending ...> attached to a different loop`。这个坑我查了 2 小时才从 GitHub issue 里翻出来。

现在写第一个测试——**写后读一致性**：

```python
# test_memory_consistency.py
import pytest
import pytest_asyncio


@pytest_asyncio.fixture
async def db_conn(db_pool):
    """每个测试函数一个独立连接，用完归还"""
    async with db_pool.acquire() as conn:
        yield conn


async def _insert_msg(conn, session_id: str, role: str, content: str):
    await conn.execute(
        "INSERT INTO agent_memory(session_id, role, content) VALUES($1, $2, $3)",
        session_id, role, content,
    )


async def _fetch_msgs(conn, session_id: str, limit=5):
    rows = await conn.fetch(
        "SELECT role, content FROM agent_memory WHERE session_id=$1 "
        "ORDER BY created_at DESC LIMIT $2",
        session_id, limit,
    )
    return [dict(r) for r in rows]


@pytest.mark.asyncio
async def test_write_then_read_should_return_same_content(db_conn):
    """正确性验证：写入一条消息后立即查询，内容应该一致"""
    session = "sess-1"
    await _insert_msg(db_conn, session, "user", "我叫小明")
    msgs = await _fetch_msgs(db_conn, session)
    assert len(msgs) == 1
    assert msgs[0]["content"] == "我叫小明"
    assert msgs[0]["role"] == "user"
```

但这只是单协程理想场景，真正的魔鬼在并发里。下面这个测试模拟 **两个协程同时写入同一个 session，检查最终条目数和顺序是否基于数据库时间戳而非客户端时间排序**。

```python
import asyncio


@pytest.mark.asyncio
async def test_concurrent_inserts_should_preserve_order(db_pool):
    """一致性验证：并发写入时序，最终查询出的顺序应由 server 时间决定"""
    session = "sess-concurrent"
    async def insert_user1():
        async with db_pool.acquire() as conn:
            await _insert_msg(conn, session, "user", "消息A")
            # 模拟极小间隔后再写一条
            await asyncio.sleep(0.05)
            await _insert_msg(conn, session, "user", "消息C")

    async def insert_user2():
        async with db_pool.acquire() as conn:
            await _insert_msg(conn, session, "user", "消息B")

    # 让协程交替执行
    await asyncio.gather(insert_user1(), insert_user2())

    # 检查最终顺序（按 created_at 降序）
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT content FROM agent_memory WHERE session_id=$1 "
            "ORDER BY created_at DESC",
            session,
        )
    contents = [r["content"] for r in rows]
    # 期望：C（最晚）-> B -> A
    assert contents == ["消息C", "消息B", "消息A"], f"顺序错乱: {contents}"
```

这里**我用 `asyncio.sleep(0.05)` 人为制造“消息C”写入延迟，确保真实物理时间先后，而不是靠 Python 代码执行速度**。如果依赖客户端传 `created_at`，这个测试大概率挂。

## 踩坑记录：官方文档没说的那些细节

1. **`pg_isready` 返回成功，`asyncpg` 连接却报 `Cannot assign requested address`**  
   **现象**：容器刚启动，`pg_isready` 检查通过，但 `asyncpg.connect()` 偶尔抛 OSError。  
   **原因**：`pg_isready` 只检查 `postmaster` 进程是否在监听，但数据库还没完成 crash recovery 和 WAL 回放。在生产中用 `healthcheck` 的 `pg_isready` 够用，但在极快启动的测试容器里，内核 TCP backlog 队列还塞着 SYN 包，应用连过去直接被 RST。  
   **解决**：健康检查加入 `pg_isready` 重试的同时，在 `conftest` 的等待循环里加一次 `asyncpg.connect` 探测（实际操作：用 `psycopg2` 短连接探测比 asyncpg 更可靠，因为它不依赖事件循环）。

2. **`pytest-asyncio` 的 `event_loop` scope 和 session fixture 互锁**  
   **现象**：定义了 `pytest.fixture(scope="session") async def db_pool` 后运行测试，直接报 `RuntimeError: no running event loop`，或者 pool 创建成功但第一个测试函数卡死。  
   **原因**：官方例子里 `event_loop` 默认是 function scope，`pytest-asyncio` 旧版本还不支持 session 级别的异步 fixture。即使在 0.21+ 版本里，也必须显式声明 `@pytest.fixture(scope="session") def event_loop() -> asyncio.AbstractEventLoop`。  
   **解决**：就是上面 `conftest.py` 里那样，手动 new loop 并 yield。另外注意 CI 里如果 `uvloop` 被替换，要用 `asyncio.new_event_loop()` 而不能 `asycio.get_event_loop()`。

3. **Docker Compose `down -v` 删卷但 CI 并发跑时端口映射冲突**  
   **现象**：多个 pytest 进程并行（比如 `pytest -n auto`），端口映射 `"0:5432"` 会导致不同进程的容器在宿主机监听不同随机端口，但 Docker Compose project name 默认按目录名，导致先后启动的两套测试会操作同一组服务，down 的时候把对方的容器也杀了。  
   **解决**：在 `docker compose` 命令里加 `--project-name` 参数，用 `pytest` 的 `xdist` worker id 作为后缀，让每个 worker 启动独立的 compose 栈。命令改成 `subprocess.run(["docker", "compose", "-p", f"test_memory_{os.getpid()}", "up", "-d"])`。

## 效果验证：从半夜救火到 15 分钟卡住 bug

这套测试集成进 CI 后，我刻意往记忆模块引入了一个问题：在批量插入时，用客户端时间戳排序，而客户端时钟有 2 秒偏差。改成这套真数据库测试后，`test_concurrent_inserts_should_preserve_order` 直接红灯，把问题拦在了合并请求里，没机会上生产。跟之前对比：

| 指标 | 手工测试时代 | Docker + Pytest 集成测试后 |
|------|---------------|--------------------------|
| 一致性用例覆盖 | 0（只测单线程） | 5 个并发场景 |
| bug 发现时间 | 上线后 2 天（用户反馈） | 代码提交后 15 分钟 |
| 环境搭建成本 | 手动 docker run + 清数据 | 一键 `pytest` |
| 新人接手时间 | 半天看 wiki 配环境 | 5 分钟 clone 就跑通 |

## 拿来即用的模板

我把最小可用的 `conftest.py` 和 `docker-compose.yml` 提炼成一个模板，改一下数据库名和建表语句就能塞进你的项目：

- `docker compose up -d` 启动服务，`conftest.py` 自动等待就绪
- 支持 session 级连接池，所有异步测试复用
- 并发测试直接用 `asyncio.gather` 模拟真实调度

模板仓库见 GitHub，地址放在文末“关于作者”。

---

`#Python` `#Pytest` `#Docker` `#AI Agent` `#后端测试`

**关于作者**  
一个专啃后端集成测试硬骨头的工程师，常用 Pytest + Docker 把隐藏的并发坑提前挖出来。  
GitHub: https://github.com/baofugege — 本文配套模板在 `pytest-docker-memory-demo` 仓库。  
Sponsor: https://github.com/sponsors/baofugege — 如果这篇文章帮你省了几小时抓虫时间，可以请我喝杯咖啡。  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram @baofugege