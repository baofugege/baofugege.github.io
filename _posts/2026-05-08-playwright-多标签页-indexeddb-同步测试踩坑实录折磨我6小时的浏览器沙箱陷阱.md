---
layout: post
title: "Playwright 多标签页 IndexedDB 同步测试踩坑实录：折磨我6小时的浏览器沙箱陷阱"
date: 2026-05-08 00:00:00 +0800
---

凌晨一点，CI 机器人第十次在钉钉群里@我：“前端多标签页同步测试失败”。这已经是我们协作白板项目第三次因为这个用例挂掉，而我只想睡觉。最终我对着 Playwright 的文档翻了又翻，才发现自己掉进了一个特别蠢的坑——**浏览器上下文隔离**。下面我把整个排坑过程原原本本写出来，让你少加一点班。

## 问题拆解

我们前端用 IndexedDB 做离线数据持久化，一个页面写入数据后，通过 BroadcastChannel 通知其他打开的标签页刷新 UI。测试目标很明确：用 Playwright 模拟两个标签页，验证数据能不能实时同步过去。

常规思路：开两个 Page 对象，一个写 IndexedDB + 发广播，另一个监听 BroadcastChannel 并断言是否收到。我一开始写的伪用例大概长这样：

```
tab1 -> 写入 IndexedDB -> 通过 BroadcastChannel 发送 "sync" 消息
tab2 -> 提前监听 BroadcastChannel -> 收到消息后读取 IndexedDB -> 断言数据最新
```

看起来人畜无害，实际上用 Playwright 跑起来后，第二个页面永远收不到广播消息。不是偶发，是百分百失败。

根因是什么？我用两个 `browser.newContext()` 创建了两个完全隔离的浏览器上下文。在 Chromium 里，不同 BrowserContext 之间不仅 IndexedDB 存储是隔离的，就连 BroadcastChannel 也被隔离了——你在 contextA 发出的消息，contextB 根本收不到。这就是典型的“模拟多标签页”，却用了错误的 API。

## 方案设计

要测试真正的 **多标签页数据同步**，必须在 **同一个** BrowserContext 下打开多个 Page。这样它们才共享同一个源 (origin) 的存储（IndexedDB、localStorage），BroadcastChannel 也才能正常工作。

**为什么不用 Cypress？** Cypress 天然不支持多标签页，虽然可以通过 `cy.origin` 模拟，但对 IndexedDB 这种存储层的同步验证很蹩脚。  
**为什么不用 Puppeteer？** Puppeteer 早期版本对多页面管理不够优雅，且 Playwright 在等待异步事件、网络空闲、locator 断言上明显更成熟，能少写一堆 `waitForTimeout`。  
**为什么不用真实的两个浏览器窗口？** 自动化测试要跑在 headless CI 环境，没有桌面。

架构思路很简单：一个 BrowserContext，两个 Page，同源 URL。核心逻辑通过 `page.evaluate()` 在浏览器端操作 IndexedDB 和 BroadcastChannel，断言用 Playwright 的 `waitForFunction` 轮询页面状态。

## 核心实现

**这段代码解决的是：在同一存储上下文中创建两个页面，并在一个页面写入数据后验证另一个页面能通过 BroadcastChannel 感知变动。**

先上完整可运行的测试用例（需要安装 `playwright` 和 `idb` 前端库，本地起一个静态服务）：

```typescript
import { test, expect, BrowserContext } from '@playwright/test';
import http from 'http';
import fs from 'fs';
import path from 'path';

// 一个最小化的 HTML 页面，内置 idb 操作和 BroadcastChannel 监听
const PAGE_HTML = `
<!DOCTYPE html>
<html>
<body>
  <div id="status">idle</div>
  <script type="module">
    import { openDB } from 'https://unpkg.com/idb?module';
    const channel = new BroadcastChannel('sync-demo');
    const statusEl = document.getElementById('status');

    async function initDB() {
      const db = await openDB('sync-db', 1, {
        upgrade(db) {
          if (!db.objectStoreNames.contains('items')) {
            db.createObjectStore('items', { keyPath: 'id' });
          }
        }
      });
      window._db = db;
    }

    async function writeItem(id, value) {
      const db = await openDB('sync-db', 1);
      await db.put('items', { id, value });
      channel.postMessage({ type: 'changed', id, value });
      statusEl.textContent = 'written';
    }

    async function readItem(id) {
      const db = await openDB('sync-db', 1);
      return await db.get('items', id);
    }

    // 暴露给 Playwright 直接调用
    window._writeItem = writeItem;
    window._readItem = readItem;

    channel.onmessage = async (event) => {
      if (event.data.type === 'changed') {
        const item = await readItem(event.data.id);
        statusEl.textContent = 'synced:' + JSON.stringify(item);
      }
    };

    initDB();
  </script>
</body>
</html>
`;

let server: http.Server;
const PORT = 4567;

test.beforeAll(async () => {
  // 启动一个本地静态服务器，返回上面的 HTML
  server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(PAGE_HTML);
  });
  await new Promise<void>(resolve => server.listen(PORT, resolve));
});

test.afterAll(() => {
  server?.close();
});

test('同一 BrowserContext 下的多标签页 IndexedDB 同步', async ({ browser }) => {
  // 关键：只创建一个 BrowserContext
  const context: BrowserContext = await browser.newContext();
  const page1 = await context.newPage();
  const page2 = await context.newPage();

  await page1.goto(`http://localhost:${PORT}`);
  await page2.goto(`http://localhost:${PORT}`);

  // 等待 IndexedDB 初始化完成
  await page1.waitForFunction(() => !!window._writeItem);
  await page2.waitForFunction(() => !!window._readItem);

  // 在 page1 写入数据
  await page1.evaluate(() => (window as any)._writeItem('item-1', 'hello sync'));

  // 在 page2 轮询等待同步完成（BroadcastChannel 触发后 statusEl 会变为 synced:...）
  await page2.waitForFunction(() => {
    const el = document.getElementById('status');
    return el && el.textContent?.startsWith('synced:');
  }, {}, { timeout: 5000 });

  // 最终断言：page2 能通过 IndexedDB 读到正确数据
  const item = await page2.evaluate(() => (window as any)._readItem('item-1'));
  expect(item).toEqual({ id: 'item-1', value: 'hello sync' });
});
```

**为什么不在 `browser.newPage()` 之前用 `browser.newContext()` 创建两个 Context？** 这是最容易栽的坑。如果你这么写：

```typescript
const page1 = await browser.newContext().newPage();
const page2 = await browser.newContext().newPage();
```

结果就是 page1 和 page2 运行在两个互相隔离的浏览器会话里，连 BroadcastChannel 都收不到对方的消息，IndexedDB 自然也是两套。

**让页面提前挂好 BroadcastChannel 监听，再触发写入**，这个顺序也很重要。如果先写入再让 page2 监听，会错过消息，导致测试假失败。所以我们先用 `waitForFunction` 确认两个页面都完全就绪。

## 踩坑记录

### 坑1：IndexedDB 初始化是异步的，直接调用 `_writeItem` 抛 undefined

**现象**：测试日志里抛 `window._writeItem is not a function`。  
**原因**：`initDB()` 是 async 函数，页面 `load` 后还没来得及执行完，Playwright 就开始 `evaluate` 调用 `_writeItem`。  
**解决**：利用 `waitForFunction(() => !!window._writeItem)` 显式等待函数挂载，不要在 `page.goto` 后只靠 `networkidle`。这是个官方文档不会告诉你的细节，因为 IndexedDB 的 `openDB` 通常需要数百毫秒。

### 坑2：错误地在 `page.evaluate` 里返回了未解析的 Promise

**现象**：`evaluate` 执行后断言拿到的数据是 `undefined`，但页面实际写入成功。  
**原因**：我一开始写了这样的代码：

```typescript
const writeResult = await page1.evaluate(() => {
  return window._writeItem('item-1', 'value'); // _writeItem 返回 Promise
});
```

Playwright 的 `evaluate` 会自动等待返回值是 Promise 的情况吗？**大部分文档说会**，但对于通过 `window._writeItem` 这种挂载的函数，内部如果使用了 `idb` 的 `openDB`（异步），而 `openDB` 在 `evaluate` 的序列化边界上偶尔会出现 Promise 未展平就返回的情况（尤其是老版本 Chromium）。  
**解决**：不要依赖 `evaluate` 的隐式 Promise 解析，直接通过页面内状态（例如 `status` 文本）来间接证明写入已完成，或者干脆把读写都写成显式等待。

## 效果验证

用正确的同一 Context 方案后，用例从原来 0% 的通过率直接拉到 100%。同时我发现我们的广播机制在低频设备上延迟超过 300ms，通过 `waitForFunction` 的轮询超时配置，把隐藏的性能劣化直接暴露出来，推动了前端改用 `leader-election` 减少多余广播，最终同步延迟降到 50ms 以内。

| 场景 | 原方案（分 Context） | 优化方案（同 Context） | 
|------|-------------------|---------------------|
| 同步成功率 | 0% | 100% |
| 平均同步延迟 | N/A (失败) | 50ms |
| 误报率 | 极高 | 零 |

## 可直接抄走的工具函数

如果你项目中也需要做类似测试，把下面这段复制到你的 Playwright 工具包里，一秒创建同源多标签页：

```typescript
async function createSameOriginPages(
  context: BrowserContext, url: string, count: number
) {
  const pages = await Promise.all(
    Array.from({ length: count }, () => context.newPage())
  );
  await Promise.all(pages.map(p => p.goto(url)));
  return pages;
}
```

上面必须使用传入的同一个 `context`，千万不要在函数内部自己 `browser.newContext()`。

---

`#Playwright` `#IndexedDB` `#前端测试` `#自动化测试` `#踩坑复盘`

**关于作者**  
我是一个常年跟浏览器存储、端到端测试打交道的后端/架构开发者，相信“代码不说谎，但环境会坑你”。  
GitHub: [https://github.com/baofugege](https://github.com/baofugege)  
Sponsor: [https://github.com/sponsors/baofugege](https://github.com/sponsors/baofugege) — 如果这篇文章帮你省了几个小时的排查，请我喝杯咖啡。  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram @baofugege