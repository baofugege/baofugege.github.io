---
layout: post
title: "LLM 记忆测试踩坑实录：用 Playwright+Allure 将幻觉复现定位时间从 2 小时压到 5 分钟"
date: 2026-05-05 00:00:00 +0800
---

凌晨两点，客户群里炸了——“你们的 AI 助手把我上周交代的客户背景全忘了，还编了一段新的！”我爬起来打开日志，对着几万行对话记录像大海捞针。那一夜，我花了快两个小时才定位到：当对话轮次超过 24 轮时，我们的历史裁剪逻辑会悄悄丢弃中间的系统提示，导致记忆完全断裂。第二天我就决定：这种折磨，必须用自动化终结。如果你也在做 LLM 应用，并且被“偶尔失忆、偶尔幻觉”的 bug 折磨，这篇文章会把一套可直接复用的端到端记忆测试方案拍在桌上。

## 问题拆解：LLM 的“失忆”比幻觉更可怕

LLM 应用上线后，用户会默认它能记住对话历史。可实际情况是：任何涉及上下文窗口管理、RAG 检索、多 Agent 状态传递的环节，都可能让模型“失忆”——该用到的信息没传进去，或者中间被截断、覆盖。这种 Bug 的危险在于它不是每次都错，往往只在特定对话长度、特定消息顺序下触发，手工复现极其困难。

常规测试手段无非两类：一是单元测试测模型接口，只验证单轮输入输出，根本碰不到记忆逻辑；二是手动在 UI 上点几十轮，费时费力，且无法精确复现同一路径。一旦测试人员记错操作顺序，这个 Bug 又溜走了。我们真正缺的，是一个能像真实用户一样跟系统连续对话、并能清晰记录每一轮“它记得什么、从哪一轮开始忘”的自动化方案。

## 方案设计：为什么是 Playwright + Allure 组合？

面对这个需求，我考察了几条路：

- **API 直接调用：** 快，但绕过了前端可能存在的消息拼接、时间戳、用户身份注入等逻辑，测不到真实全链路。
- **Selenium：** 生态老，但异步等待和现代前端框架的 Shadow DOM 处理起来很痛苦，而且社区已明显偏向 Playwright。
- **Cypress：** 仅限 JS/TS 生态，我们的后端和模型服务都基于 Python，技术栈统一成本太高。

最终选 Playwright 的理由很直接：原生支持同步/异步、auto-wait 机制能大幅减少 flaky test，而且可以对 Chromium 做到完美控制，能截取每一个关键步骤的截图。而选 Allure 是因为它生成的报告自带步骤树、附件展示、失败高亮，非技术角色（比如产品经理）也能一眼看懂哪一轮对话的记忆断掉了。

整个架构很简单：`pytest + playwright + allure-pytest`。测试用例里用 Playwright 操作线上前端页面，执行多轮对话，每轮断言关键信息是否仍被模型记住。Allure 负责把每轮的对话内容、模型响应、断言结果、页面截图打包成一份 HTML 报告，复现路径一目了然。

## 核心实现：从打开浏览器到自动断言记忆

**第一段代码：解决浏览器复用和登录态问题**  
很多时候测试会因为登录流程而慢得离谱，每次都要从头扫码或输入验证码。我们直接把已登录的`storage_state`保存下来复用。

```python
# conftest.py - 复用登录态，避免每次测试都登录
import pytest
from playwright.sync_api import sync_playwright
from pathlib import Path

STATE_FILE = Path(__file__).parent / "auth_state.json"

@pytest.fixture(scope="session")
def browser():
    with sync_playwright() as p:
        # 如果想看到执行过程可设为 False
        browser = p.chromium.launch(headless=True, slow_mo=100)
        yield browser
        browser.close()

@pytest.fixture(scope="function")
def page(browser):
    context = browser.new_context(
        viewport={"width": 1280, "height": 720},
        storage_state=STATE_FILE if STATE_FILE.exists() else None
    )
    page = context.new_page()
    yield page
    context.close()
```

如果`auth_state.json`不存在，就在一个单独的脚本里用`page.context.storage_state(path=STATE_FILE)`手动保存一次。之后 CI 里完全静默运行。

---

**第二段代码：解决记忆测试的核心逻辑——模拟多轮对话并断言记忆**  
这里的场景是：用户在第一轮告诉 AI 自己叫“张三”，然后在第 N 轮回头问“我叫什么名字？”。我们用一个数据驱动的方式，让同一套逻辑覆盖不同轮次，精准找出从哪一轮开始记忆丢失。

```python
# test_llm_memory.py
import allure
import pytest

# 模拟真实用户的多轮对话——第一轮先种下“记忆锚点”
def send_message(page, text):
    """封装消息发送：定位输入框、输入、点击发送"""
    # 给AI一定思考时间，使用 locator 和 auto-wait
    input_box = page.locator("textarea[placeholder*='输入消息']")
    input_box.click()
    input_box.fill(text)
    # 关键：有些应用的发送按钮在输入框聚焦后才出现
    send_btn = page.locator("button:has-text('发送')")
    send_btn.click()

def get_last_response(page):
    """获取AI最后一条回复文本"""
    # 等待新消息出现，假设最后一条聊天气泡有特定 class
    page.wait_for_selector(".message-item:last-of-type .message-content")
    return page.locator(".message-item:last-of-type .message-content").inner_text()

@allure.feature("LLM 长期记忆")
@pytest.mark.parametrize("rounds", [5, 10, 15, 20, 24, 30])
def test_name_memory_across_rounds(page, rounds):
    """
    这段测试解决的核心问题：验证模型在多轮后是否还记得最开始的用户名。
    通过参数化 rounds，我们能快速探测记忆断裂的临界点。
    """
    page.goto("https://your-llm-app.com/chat")

    with allure.step("第一轮：告诉AI我的名字"):
        send_message(page, "我的名字是张三，请你记住。")
        response = get_last_response(page)
        # 截图作为证据，万一断言失败能肉眼复核
        allure.attach(page.screenshot(), "第一轮截图", allure.attachment_type.PNG)
        # 非强制断言：看AI是否确认收到
        assert "张三" in response or "好的" in response, "AI 未按预期确认名字"

    # 中间填充对话，把上下文“撑长”，但不覆盖原始名字
    with allure.step(f"进行{rounds}轮中性对话，扩充上下文"):
        for i in range(1, rounds + 1):
            send_message(page, f"今天是测试的第{i}天，天气不错。")
            # 只读最后一轮回复，不特别断言内容，保证对话持续
            get_last_response(page)

    with allure.step("最后一轮：问AI我叫什么"):
        send_message(page, "请问我叫什么名字？")
        final_response = get_last_response(page)
        allure.attach(page.screenshot(), "最终对话截图", allure.attachment_type.PNG)
        # 核心断言：AI 必须还记得“张三”
        assert "张三" in final_response, (
            f"在{rounds}轮中性对话后，模型记忆丢失！实际回复: {final_response}"
        )
```

有了这个用例，一次性就能跑出从 5 轮到 30 轮的记忆保持情况，不用人工一遍遍手点。每一轮失败的参数都会在 Allure 报告里留下完整对话历史截图，开发拿到直接复现。

---

**第三段代码：将 Allure 报告自动化集成到 CI**  
很多团队只会在本地跑 Allure generate，其实它完全可以变成 CI 制品，让所有人在 PR 阶段就看到 Memory 测试结果。

```python
# pytest.ini 配置
[pytest]
addopts = --alluredir=./allure-results --clean-alluredir
```

然后在 GitHub Actions 或 GitLab CI 加一步：

```yaml
- name: 执行端到端记忆测试
  run: pytest tests/test_llm_memory.py
- name: 生成 Allure 报告
  if: always()  # 即使测试失败，报告仍然有价值
  run: allure generate ./allure-results -o ./allure-report --clean
- name: 上传报告为 artifacts
  uses: actions/upload-artifact@v3
  with:
    name: llm-memory-report
    path: ./allure-report
```

这样任何人点击 CI 的 artifacts 就能直接打开报告 HTML，看到一条漂亮的步骤时间线。

## 踩坑记录：官方文档不会告诉你的细节

### 坑 1：Playwright 的 auto-wait 在 SPA 应用中会“假成功”

**现象：** 测试有时会报 `TimeoutError`，说找不到元素，但截图里明明就有。  
**原因：** 我们的前端是 React SPA，发送消息后会先出一个空占位的 `message-item`，稍后才被真实内容替换。Playwright 的 `wait_for_selector` 看到 DOM 节点立刻返回，但节点内部的文本还是空的，这时 `inner_text()` 返回空字符串，后续断言就挂了。  
**解决：** 放弃一次性等待，改用 `expect` + `to_contain_text` 或循环等待：

```python
from playwright.sync_api import expect

expect(page.locator(".message-item:last-of-type .message-content")).to_contain_text(
    "张三", timeout=10000
)
```

这样会不断重试直到文本确实出现，彻底解决“已经找到元素但内容还在路上”的问题。

### 坑 2：Allure 报告里中文显示为方块或乱码

**现象：** CI 下载的 Allure 报告中所有中文都变成□□□。  
**原因：** Allure 命令行在生成报告时，默认使用的字体不包含中文字符，且你的 CI 机器上没有相关字体。  
**解决：** 在 CI 中提前安装中文字体（比如 `apt install fonts-noto-cjk`），然后在生成报告前设置环境变量 `ALLURE_OPTS="-Dallure.encoding=UTF-8"`。更彻底的做法是定制 Allure 的 CSS，但安装字体就可以解决 90% 的问题。

## 效果验证：从两小时到五分钟的质变

以前手工复现一次记忆 Bug，平均耗时 1.5 - 2 小时，且因为人工疲劳，经常第 20 轮忘了之前说过什么，导致错误结论。接入这套自动化后，跑完全部 6 组参数（5/10/15/20/24/30 轮对话）仅需 **4 分 32 秒**。更重要的是，在一次回归中，脚本精准暴露出我们自研的“智能上下文压缩”功能在达到 24 轮后会错误地删除 System Prompt 中的名字信息——这个 Bug 已经线上飘了快一周，因为测试人员从来没测到过第 24 轮。

| 维度 | 手工测试 | Playwright + Allure |
|------|----------|---------------------|
| 单场景耗时 | ~120 min | 4.5 min |
| 可复现性 | 低（依赖人工记忆） | 100%（每次执行路径一致） |
| 故障定位手段 | 肉眼翻看聊天记录 | 步骤树 + 截图 + 精确到轮的断言 |
| 是否可加入 CI | 否 | 是，每次 PR 自动运行 |

## 可直接用的代码 / 工具

拿走这个最小可运行测试模板，替换成你自己的应用选择器和 URL，就能在本地跑起来：

```bash
pip install playwright pytest allure-pytest
python -m playwright install chromium
pytest test_llm_memory.py --alluredir=./allure-results
allure serve allure-results
```

完整 Demo 仓库（含 `auth_state.json` 保存脚本、CI 配置）见我的 GitHub。

`#LLM` `#Playwright` `#自动化测试` `#Allure` `#AI工程化`

---

**关于作者**  
我是 bao，一个在 AI 应用工程化领域反复踩坑、并把坑填平的后端 / 架构开发者。  
GitHub: [https://github.com/baofugege](https://github.com/baofugege)  
Sponsor: [https://github.com/sponsors/baofugege](https://github.com/sponsors/baofugege) — 如果这篇文章帮你省下了排查时间，可以请我喝杯咖啡。  
提供服务：Python 后端性能优化 / 工具链定制 / 技术咨询，联系 Telegram [@baofugege](https://t.me/baofugege)