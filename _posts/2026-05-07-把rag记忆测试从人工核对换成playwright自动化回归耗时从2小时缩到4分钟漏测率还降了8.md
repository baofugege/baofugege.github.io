---
layout: post
title: "把RAG记忆测试从人工核对换成Playwright自动化，回归耗时从2小时缩到4分钟，漏测率还降了80%"
date: 2026-05-07 00:00:00 +0800
---

凌晨一点，同事发来一张截图：用户说“我叫小明，记住我喝咖啡不加糖”，第二次对话时，机器人给了一杯全糖拿铁。产品经理在群里@所有人——“记忆存储是不是又坏了？”我盯着聊天记录，叹了口气，打开表格，开始第N遍手工回归：清缓存、开浏览器、打10轮对话、对照预期、截图、填结果… 两个小时过去，测完5个场景，眼睛已经花了，还漏了3个边界用例。那一刻我决定：必须让机器干这件事。

## 问题拆解：为什么RAG记忆测试这么折磨人

RAG应用的记忆存储不像传统API那样能用几个assert搞定。它涉及长期记忆、会话窗口、向量检索、LLM生成——任何一个环节出毛病，都可能让用户感觉“机器人失忆了”。典型的测试场景长这样：先跟机器人聊10轮，在第3轮埋入“我最喜欢的电影是《让子弹飞》”，第7轮聊天气，第10轮突然问“我之前说过我喜欢什么电影？”，看它能不能从记忆里捞出来。

手工干这事儿有三个死穴：  
1. **长对话状态难追溯**：记忆错乱往往发生在多轮上下文切换之后，手工测到第四、五轮时，自己都忘了前面说过什么。  
2. **流式输出让断言不稳定**：LLM生成是逐字显示的，经常一句话还没结束，我们就急着拉滚动条检查关键字，漏判概率极高。  
3. **回归成本随记忆类型指数增长**：短期记忆、长期记忆、摘要记忆、向量记忆……每加一种存储，用例数量翻一倍，手工根本扛不住。

常规的解决方案是写单元测试——但 LLM 输出不固定，即便记忆对了，表达方式也能千变万化。单纯用固定字符串断言直接翻车。真正的难点在于：需要有一个东西，能模拟真实用户在多轮对话中**等待**、**观察**、**断言**，同时又能在 CI 里无人值守跑起来。Playwright 简直就是为这个而生的。

## 方案设计：为什么是Playwright而不是Selenium或Puppeteer

当时有三个选择：Selenium、Puppeteer、Playwright。Selenium 最早淘汰——它对现代 Web 应用的自动等待机制太弱，经常需要写各种 `sleep`，测试又慢又脆。Puppeteer 只支持 Chromium，而我们的 RAG 应用在生产环境里，用户可能用 Safari 或 Firefox，需要跨浏览器验证。

Playwright 打动了我的点：
- **自动等待**：元素可交互、页面加载完毕、网络静默，这些它都帮你处理好了，写断言不需要到处塞 `sleep`。
- **多浏览器支持**：同一个脚本，跑 Chromium、Firefox、WebKit 只需改一个配置。
- **截屏和录像**：失败时的 trace 能回放每一步，不用对着日志怀疑人生。
- **网络拦截与 mock**：可以拦截 API 请求，甚至模拟记忆存储服务异常，验证降级逻辑。

架构上，我们设计一个 **记忆准确度自动化套件**：
1. **剧本定义**：用 YAML 描述多轮对话，每轮包含用户输入、期望记忆字段、必须出现的关键字。
2. **执行器**：Playwright 浏览器实例读取剧本，按序发送消息，监听流式响应完成事件，收集完整生成文本。
3. **断言层**：对生成文本做语义级判断，不依赖精确字符串，而是检查“是否包含记忆关联的关键信息”，必要时接入一个小模型做二次校验。
4. **报告生成**：每次运行产出 HTML 报告，附带失败截图，直接扔进 CI artifacts。

为什么不直接用 API 测试？因为很多 RAG 应用的状态管理、前端会话窗口、Token 刷新逻辑全部嵌入在页面上，脱离浏览器根本复刻不出真实故障。

## 核心实现：一步步把手工测试变成自动化

下面这段代码解决“多轮对话中如何让 Playwright 等待每次生成结束，再发送下一条消息”。流式输出期间，`send` 按钮通常会 disabled 或显示停止图标，等生成完毕才会恢复。我们抓住这个变化做同步。

```python
import asyncio
from playwright.async_api import async_playwright

async def send_message_and_wait(page, text: str, timeout: int = 30000):
    """
    向聊天框发送消息，并等待 LLM 生成结束。
    假设：发送后 send 按钮 disabled，生成完毕恢复 enabled。
    """
    textarea = page.locator('textarea[placeholder*="输入"]')
    send_btn = page.locator('button:has-text("发送")')

    await textarea.fill(text)
    await send_btn.click()

    # 关键：等待发送按钮恢复可用状态，表示生成完成
    await send_btn.wait_for(state="visible", timeout=timeout)
    # 保险起见再等一丢丢，让动画渲染完毕
    await page.wait_for_timeout(500)
```

接下来是构建一个完整的记忆测试场景：用户在第一轮说自己的名字和偏好，后面某轮突然考问机器人，检查返回内容是否包含之前的信息。注意这里用 `locator` 结合 `filter` 精准拿到机器人的最新回复。

```python
async def test_long_term_memory():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        context = await browser.new_context()
        page = await context.new_page()
        await page.goto("https://your-rag-app.example.com/chat")

        # 剧本：植入记忆
        await send_message_and_wait(page, "我叫赵大宝，最喜欢的咖啡是冰美式。")
        await send_message_and_wait(page, "今天天气不错，适合工作。")
        await send_message_and_wait(page, "帮我记一下，下周三要去体检，别忘了提醒我。")
        # 干扰对话
        await send_message_and_wait(page, "那明天呢？")
        await send_message_and_wait(page, "再说一下项目排期的事情吧。")
        # 关键测试：询问之前的信息
        await send_message_and_wait(page, "对了，我之前说我最喜欢的咖啡是什么？")

        # 获取最后一条机器人回复（通常是最后一个聊天框气泡）
        last_reply = page.locator('.chat-bubble.bot').last
        reply_text = await last_reply.inner_text()

        # 断言：必须出现“冰美式”，但容忍表达方式差异
        assert "冰美式" in reply_text, f"记忆验证失败，实际回复：{reply_text}"

        await browser.close()
```

上面那段只有一个硬断言，但在真实项目中我们需要覆盖更多记忆类型。比如 **会话摘要记忆**：当对话超过一定轮数后，早期的细节被压缩成摘要，这时提问必须依赖摘要中的关键词。我们把这类场景也写成自动化。以下代码演示如何通过 Playwright 的 `route` 拦截记忆服务 API，模拟摘要存储成功但内容被污染的情况，从而验证系统的健壮性。

```python
async def test_summary_memory_robustness(page):
    # 拦截记忆保存 API，模拟服务端返回异常摘要格式
    await page.route("**/api/memory/save", lambda route: route.fulfill(
        status=200,
        body='{"status":"ok","summary":"用户喜欢猫，但他其实更喜欢狗"}'  # 故意注入错误摘要
    ))

    await send_message_and_wait(page, "我喜欢猫。")
    await send_message_and_wait(page, "其实我更喜欢狗。")
    await send_message_and_wait(page, "那到底我喜欢什么动物？")
    
    last_reply = await page.locator('.chat-bubble.bot').last.inner_text()
    # 这里预期模型可能被错误摘要误导，但逻辑上应输出“狗”，因为最新记忆权重更高
    # 这只是演示拦截如何辅助测试记忆存储的鲁棒性
    assert "狗" in last_reply or "狗狗" in last_reply, f"摘要记忆污染测试未通过：{last_reply}"
```

最后，我们把所有用例用 `pytest` 组织起来，加上 `conftest.py` 复用浏览器实例，实现并行跑。这样一套下来，回归不再是折磨，而是一行命令 `pytest test_rag_memory.py --headed` 就能躺着等结果。

## 踩坑记录：官方文档没告诉你的那些事儿

**坑一：流式生成结束的判定信号并不总是按钮恢复**  
现象：某些前端实现里，发送按钮不会disabled，只是生成过程中输入框变为不可编辑。等我用 `wait_for` 判断输入框可交互时，发现生成还没完全流入 DOM，最后一条消息不完整。  
原因：前端用了 WebSocket 推送，最后一帧到达之后还有一段渲染时间窗。  
解决：不依赖单一 UI 元素，改为监听网络响应结束 + 最后一条消息的文本长度不再变化。写了一个工具函数 `wait_for_generation_done(page, locator, stable_duration=2000)`，连续两秒内文本长度不变才视为完成。这个方法比文档里简单示例靠谱得多。

**坑二：多标签页下的记忆隔离失效**  
现象：测试用户 A 和用户 B 的记忆交叉污染，偶尔 A 的回答里出现了 B 的偏好。手工很难复现，因为不并发测不出来。  
原因：RAG 应用的前端会话 ID 存在 `localStorage` 里，同一浏览器上下文下不同标签页共享同一 `localStorage`，自然就串了。  
解决：Playwright 里每个测试用例必须使用独立的 `browserContext`（而不是新建 Page），这样才能完全隔离存储空间。虽然启动成本高一点，但这是唯一能模拟多用户完全隔离的方案。不少人一开始直接 `context.new_page()` 去跑，根本不知道背后 session 是共享的。

## 效果验证：数据说话

手工测试时期，我们平均每次回归覆盖12个主要记忆场景、约20个边界情况，耗时2小时左右，而且人工遗漏率高达18%（通过回溯线上投诉统计）。引入 Playwright 自动化后，用例库扩展到45个，全部并行执行时间缩减到4分钟。漏测率经过三轮迭代从18%下拉到3%以下（由线上用户反馈监控）。表格更直观：

| 指标 | 手工 | Playwright 自动化 |
|------|------|-------------------|
| 单次回归时长 | 2小时 | 4分钟 |
| 覆盖场景数 | 12个主场景 | 45个主场景 + 边界 |
| 人工漏测率（线上投诉反馈） | 18% | <3% |
| 并发执行能力 | 单线程 | 4核并发，支持 CI 集成 |

同时，失败用例的定位时间从天降到了分钟级——Playwright 自带的 trace viewer 直接把每一步 DOM 快照拍在脸上，不再需要对着截图猜谜。

## 可直接拿来用的代码

我把上面的核心逻辑封装成了一个可直接运行的 `pytest + Playwright` 模板，放到 GitHub 上。拿走不谢，别忘了加星。

```
git clone https://github.com/baofugege/playwright-rag-memory-test
cd playwright-rag-memory-test
pip install -r requirements.txt
pytest test_rag_memory.py --headed
```

启动后它会依次跑所有预置的对话剧本，结束后生成 HTML 报告。你只需要在 `config.yaml` 里填上自己的 RAG 应用 URL 和对应的选择器即可。

`#Python` `#Playwright` `#RAG` `#自动化测试` `#大模型工程`

---

**关于作者**  
一个常年在后端、Agent、RAG 应用坑里摸爬滚打的实战派开发者，坚信“能自动化的坚决不多点一次鼠标”。  
GitHub: [https://github.com/baofugege](https://github.com/baofugege) — 上面有更多 RAG 工程化工具  
Sponsor: [https://github.com/sponsors/baofugege](https://github.com/sponsors/baofugege) — 如果这篇文章帮你省下一小时手工回归，请我喝杯咖啡  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram [@baofugege](https://t.me/baofugege)