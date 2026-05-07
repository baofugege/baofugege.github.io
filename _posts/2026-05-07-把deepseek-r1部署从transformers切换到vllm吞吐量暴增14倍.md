---
layout: post
title: "把DeepSeek-R1部署从Transformers切换到vLLM，吞吐量暴增14倍"
date: 2026-05-07 00:00:00 +0800
---

凌晨2点被运营电话吵醒——“计费系统为什么扣了用户两次钱？”爬起来一看，我们的模型API服务在并发超过5时就开始排队超时，我临时加的“重试去重逻辑”被高负载击穿，直接导致重复扣费。这就是半年前我用HuggingFace Transformers + FastAPI 自建推理服务的真实写照。那时候的架构，就像一个用胶带缠起来的水管，随时可能崩。直到我们全面迁移到vLLM + Kong，才把并发、计费、多租户这三座大山一并搬开。这篇文章就是我从血泪里趟出来的实战记录，全程干货，可以直接抄作业。

## 问题拆解：为什么原方案撑不到双十一

我们的场景很直白：对外提供DeepSeek-R1的文本生成API，按token计费，多个客户（租户）各自有独立的API Key和额度限制。起初团队人少，我直接用Transformers加载模型，外层套FastAPI，再手撸了一套Key校验和计数逻辑写入MySQL。

问题很快暴露。根因是**原生Transformers的推理方式极其“奢侈”**：每个请求过来，不管序列长短，都要独占整个GPU显存做一次完整前向传播，没有连续批处理（continuous batching），一个请求没跑完，后面的都得排队。即便开了动态batching，因为padding带来的算力浪费，实际GPU利用率不到30%。这就导致：并发一上来，延迟直接飙到几十秒，然后客户端超时重试，触发我们那脆弱的“幂等”逻辑，最终重复计费。

另外，手写的租户管理和限流逻辑散落在业务代码里，每次改额度都要上线，网关层完全没有防线。我曾尝试在FastAPI里加信号量限流，结果只是把请求堵在了自己家门口，资源已经被占用，连健康检查都崩了。那感觉就像自己把自己锁在门外。

## 方案设计：vLLM 当引擎，Kong 做账房

复盘后我们定了两条铁律：**推理层必须实现连续批处理，让GPU像流水线一样不断档；网关层必须卸掉计费、鉴权、限流这些横切关注点，让业务代码专心做推理。**

推理引擎，候选只有三个：NVIDIA Triton、Text Generation Inference (TGI) 和 vLLM。Triton 太“重”，对我们一个急着修锅的团队来说，模型配置、模型仓库那一套学习曲线太陡。TGI 虽好，但当时对 DeepSeek 系列的支持还不够稳，且其背靠 HuggingFace，定制化稍弱。vLLM 火得有理——PagedAttention 的内存共享机制，让多个请求的 KV cache 可以在显存里动态拼合，几乎零浪费；原生兼容 OpenAI API 格式，迁移成本接近零。于是选它。

网关层，Kong 是我们一直想引入但没腾出手的组件。为什么不自己写？因为“计费、鉴权、限流”这三样看似简单，真做到生产级，要考虑插件热更新、多维度限流、高可用存储、日租户报表……自己写等于再开发半套网关。Kong 的 Key Authentication、Rate Limiting、HTTP Log 三个插件串联，就能搭出一个完整的多租户计费系统：Key 鉴权隔离租户，Rate Limiting 防恶意刷量，HTTP Log 异步把每次请求的 token 消耗写入 Kafka/ClickHouse，计费系统离线算钱。架构一清晰，心里就踏实了。

## 核心实现：从一条命令到完整多租户网关

下面直接给可跑通的代码和配置，我拆成两部分：vLLM 部署和 Kong 配置。**这段代码解决推理服务的启动**，用 Docker 一键拉起，暴露 OpenAI 兼容接口。

```bash
# 要预先下载好 DeepSeek-R1 模型，放在 /data/model/deepseek-r1
docker run -d --gpus all \
  --name vllm-deepseek \
  -p 8000:8000 \
  -v /data/model:/models \
  vllm/vllm-openai:latest \
  --model /models/deepseek-r1 \
  --tensor-parallel-size 2 \    # 双卡，用张量并行
  --max-model-len 8192 \
  --enable-prefix-caching \     # 开启前缀缓存，相同 system prompt 能秒出
  --gpu-memory-utilization 0.92
```

服务起来后，直接 `curl http://localhost:8000/v1/chat/completions` 就能像调 OpenAI 一样调 DeepSeek-R1。这个接口的稳定性和兼容性我验证过无数次，可以直接作为 Kong 上游。

**接下来的 Kong 配置解决多租户鉴权、限流和 token 消耗透传的问题。** 我用 Kong 的 decK 格式（声明式）给出，你复制粘贴扔进 Kong 就能生效。

```yaml
# kong-config.yaml
_format_version: "3.0"
services:
  - name: deepseek-r1
    url: http://vllm-deepseek:8000/v1   # 指向 vLLM 容器
    routes:
      - name: deepseek-chat
        paths:
          - /chat
        strip_path: false               # 保留 /chat 后缀，透传给 vLLM
    plugins:
      - name: key-auth                  # 启用 API Key 认证
        config:
          key_names: ["apikey"]         # 从 header 或 query 取 key
      - name: rate-limiting
        config:
          minute: 100                   # 每个租户每分钟最多100请求
          policy: local                 # 单节点限流，多节点用 redis
      - name: http-log                 # 日志推送到计费系统
        config:
          http_endpoint: http://billing-collector:3000/log
          method: POST
          timeout: 2000
          keepalive: 60000
# 消费者的 API Key 定义
consumers:
  - username: tenant_a
    keyauth_credentials:
      - key: sk-tenantA-xxxxx
  - username: tenant_b
    keyauth_credentials:
      - key: sk-tenantB-yyyyy
```

这里最关键的一步是 **http-log 插件**。vLLM 在响应体中会返回 `usage.total_tokens`，Kong 会把整个响应体原样 POST 到我们的计费 collector。collector 解析 JSON，按 `model`、`consumer`、`tokens` 字段入库，月底算钱。这样网关和业务完全解耦——vLLM 根本不知道“计费”是何物，只管把 token 数如实汇报。

我在 collector 里加了一个幂等键（用 Kong 的 `request_id` 头），彻底杜绝了重复计费。这个设计让我睡了好几个整觉。

## 踩坑记录：官方文档没写的那些事

**坑一：vLLM 启动报 OOM，即便显存貌似够。** 部署 DeepSeek-R1 时，我按官方例子设置了 `--gpu-memory-utilization 0.95`，结果直接 CUDA OOM。原因是 PagedAttention 还需要额外预留部分显存用于临时 KV cache 交换，而且 DeepSeek-R1 的实际激活值比理论值略大。解决：降到 `0.92`，并启用 `--enable-prefix-caching` 减少冗余 KV cache 分配。这点在 vLLM 文档里轻描淡写，但实际卡了我半天。

**坑二：Kong 的 key-auth 插件默认不把 consumer 信息传给上游。** 我需要后端知道是哪个租户在请求，好记录日志。最初我试图从 Kong 的 `X-Consumer-Username` 头获取，结果发现 vLLM 不理自定义头。原来 key-auth 插件虽然鉴权，但**不会自动转发 consumer 信息**，除非额外配置 `config.hide_credentials: false` 并手动加一个 `request-transformer` 插件把 username 注入 `x-consumer-username`。更优雅的做法是直接用 http-log 阶段直接拿 `consumer.username`，连注入都省了。但如果你需要业务端感知，务必记得加 transformer。

**坑三：速率限制的“边界偷袭”。** Kong 的 rate-limiting 默认窗口是固定分钟（minute），比如 100次/分钟，如果在 12:00:59 疯狂发请求，12:01:00 又立即发，可能瞬间打满两个窗口。我改用 `redis` 策略加滑动窗口，在配置里指定 `policy: redis`，并连接 Redis，完美防住突发。这招在官方文档里藏在 Advanced 章节，新手很容易错过。

## 效果验证：用数据告别深夜电话

我们对比了同一批硬件（2×A100-80G）下三种方案的吞吐量：

| 方案 | 并发数 | 平均吞吐(tokens/s) | P99延迟(s) | 峰值显存占用 |
|------|--------|-------------------|------------|-------------|
| HuggingFace + FastAPI | 4 | 284 | 21.4 | 43GB |
| vLLM (单请求) | 1 | 512 | 1.1 | 38GB |
| vLLM (16并发) | 16 | **3960** | 2.3 | 67GB |

并发 16 时，vLLM 吞吐量提升了约 **14 倍**，延迟仍然控制在 2 秒出头。这背后是 PagedAttention 和 continuous batching 的威力。更重要的是，加上 Kong 网关后，多租户隔离和计费完全自动化，不再需要人工对账。连续运营 3 个月，计费准确率 100%，再没出现过半夜报警。

## 抄作业时间

如果你也用 vLLM + Kong，一条命令就能跑起来我的 collector 原型（Node.js），直接接收 http-log 并打印 token 消耗：

```bash
npx -y @baofugege/vllm-kong-billing-collector --port 3000
```

配置 Kong 的 http-log endpoint 指向它，就能在控制台看到实时计费流水。代码完整开源在我的 GitHub。

`#vLLM` `#DeepSeek` `#Kong` `#API网关` `#性能优化`

---

**关于作者**

我是宝哥，一个专啃硬骨头、只写能跑通代码的后端/架构实战派。如果你正在用 LLM 构建商业 API，这篇文章的完整代码和 Docker Compose 全栈示例都在 GitHub 维护。  
GitHub: [https://github.com/baofugege](https://github.com/baofugege)  
Sponsor: [https://github.com/sponsors/baofugege](https://github.com/sponsors/baofugege) —— 如果这套方案帮你省了时间，请我喝杯咖啡。  
提供服务：Python 后端性能优化 / 大模型推理部署定制 / 技术咨询，联系 Telegram [@baofugege](https://t.me/baofugege)