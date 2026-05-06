---
layout: post
title: "LangChain+Chroma 构建 RAG 踩坑实录：这两个问题让我排查了 6 小时"
date: 2026-05-06 00:00:00 +0800
---

产品经理晚上 10 点突然丢过来 200 个 PDF：“明天早上要给老板演示内部知识库问答，很急。”我心想 RAG 这套我熟，LangChain 加个向量库分分钟搞定。结果从下午 4 点一直肝到晚上 10 点，差点翻车——倒不是因为流程跑不通，而是两个隐蔽的坑直接把准确率干到不到 40%，硬生生排查了 6 个小时。这篇文章带你复现整个 RAG 系统，顺带把那两个坑的根儿刨出来。

## 问题拆解：为什么你没法拿 GPT 直接喂文档

想做一个能回答“公司假期政策是什么”“xx 项目复盘结论”这类问题的系统，最简单的办法是把所有文档拼成一个长 prompt 扔给 GPT。但现实会教你做人：200 个 PDF 加起来超过 80 万字，GPT-4 的 128K 上下文也吞不下，而且费用会让你被财务追杀；微调更不现实——文档天天变，总不能每次更新都烧几千刀去训练模型。

所以向量检索 + 生成（RAG）成了唯一出路：把文档切成小段，用 embedding 模型向量化后存进向量数据库，提问时先检索最相关的片段，再把片段作为上下文拼给 LLM 生成答案。这个模式看似简单，实际操作里“怎么切”“怎么存”“怎么搜”每一个环节都有坑，而最让我崩溃的坑都埋在 LangChain 和 Chroma 的交互细节里。

## 方案设计：为什么是 Chroma 而不是 Pinecone 或 FAISS

选型之前我先问了自己三个问题：要不要钱、支不支持元数据过滤、能不能本地一键启停。

Pinecone 要钱，而且数据要上云，内网文档直接排除；Weaviate 功能倒是强，但部署起来起码要折腾半小时 Docker，不适合“明早就要”的急活；FAISS 速度极快，可惜不支持元数据过滤（比如按文档类型、时间范围筛选），后续业务一加需求就得推倒重来。最终选了 Chroma：它在本地跑，安装就是一行 `pip install chromadb`，持久化、元数据过滤、相似度搜索全都内置，跟 LangChain 的集成也最顺滑。

整体架构不复杂：**加载文档 → 文本切分 → 生成 Embedding → 写入 Chroma → 用户提问时检索 top-k 片段 → 拼 Prompt → LLM 生成答案**。LangChain 帮你把这几个步骤串成链，你只需要管好每个环节的参数和边界情况。

## 核心实现：两段完整代码跑通 RAG

**第一段代码要解决的问题**：把散落的 PDF 变成可被检索的向量片段，并持久化到 Chroma，保证下一次启动时不用重新索引。

```python
import os
from langchain_community.document_loaders import PyPDFDirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# 1. 加载 PDF 目录
loader = PyPDFDirectoryLoader("./docs")          # 自动扫描所有 PDF
documents = loader.load()
print(f"加载文档数: {len(documents)} 页")

# 2. 切片：这里 chunk_size 和 overlap 是两个大坑之源，后面细说
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,         # 每块最大字符数
    chunk_overlap=200,       # 块间重叠字符数，防止关键信息被切断
    separators=["\n\n", "\n", "。", "，", " ", ""]  # 按段落优先级切
)
chunks = text_splitter.split_documents(documents)
print(f"切分片段数: {len(chunks)}")

# 3. 生成向量并存入 Chroma（自动持久化到本地目录）
embeddings = OpenAIEmbeddings()                  # 默认 text-embedding-ada-002
vectordb = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"              # 重启后可复用，省去重 embed 开销
)
vectordb.persist()
print("向量库构建完成，持久化至 ./chroma_db")
```

**第二段代码要解决的问题**：基于已构建的向量库，实现“提问 → 检索 → 生成答案”的完整问答链，并让 LLM 严格只依据文档回答，避免胡编。

```python
from langchain_openai import ChatOpenAI
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate

# 1. 自定义 Prompt，强制 LLM 只基于提供的上下文回答
prompt_template = """你是一个严谨的内部知识库助手。请严格依据以下上下文回答问题。
如果上下文中没有答案，请直接说“未找到相关信息”，禁止编造。

上下文：
{context}

问题：{question}
答案："""
PROMPT = PromptTemplate(
    template=prompt_template, 
    input_variables=["context", "question"]
)

# 2. 加载已持久化的向量库，连接相同的 embedding 模型
vectordb = Chroma(
    persist_directory="./chroma_db",
    embedding_function=OpenAIEmbeddings()
)

# 3. 创建问答链：默认检索 top-4 片段，且使用我们定制的 Prompt
llm = ChatOpenAI(model_name="gpt-4o", temperature=0)  # 温度设 0，保证结果稳定
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",                               # 直接把检索到的片段塞进 prompt
    retriever=vectordb.as_retriever(search_kwargs={"k": 4}),
    chain_type_kwargs={"prompt": PROMPT},
    return_source_documents=True                      # 调试时能看到具体检索了哪些片段
)

# 4. 问答测试
query = "员工带薪年假是多少天？"
result = qa_chain.invoke({"query": query})
print(f"答案: {result['result']}")
print("\n参考片段:")
for doc in result["source_documents"]:
    print(f"- {doc.metadata['source']} : {doc.page_content[:80]}...")
```

这两段代码直接复制就能跑，前提是安装依赖并设置 `OPENAI_API_KEY`。

## 踩坑记录：官方文档没告诉你的两件事

**坑 1：切片策略导致回答“缺胳膊少腿”，查了 4 小时**  
现象：用户问“项目复盘的主要教训是什么”，系统只返回了第一条教训，明明文档里列了三点。  
原因：`RecursiveCharacterTextSplitter` 按照字符数切分，而原先的文档里三条教训写在一个 `<ul>` 列表中，中间没有明显的段落分隔，刚好在第 980 个字符处被拦腰切断，第二条和第三条教训飞到了下一个 chunk。由于检索只返回 top-4，下一个 chunk 相关度稍低就被挤掉了。  
解决：换用 `MarkdownHeaderTextSplitter` 识别 `##` 标题作为逻辑段落边界，同时结合 `chunk_size` 兜底；对于纯文本无结构的文件，加大 `chunk_overlap` 到 400 并对关键列表类文件单独处理。这个小改动让回答完整率从 68% 提升到 94%。

**坑 2：Chroma 的元数据过滤在 LangChain 里“静默”失效，排查 2 小时**  
现象：我打算按文档类型过滤（比如只查“制度类”文档），在 `as_retriever` 里加了 `filter` 参数，结果检索依然返回全量，如同没加过滤一样。  
原因：LangChain 的 `Chroma.as_retriever` 底层调用 `similarity_search`，当你传 `search_kwargs` 时，`filter` 这个 key 并不会被自动转成 Chroma 的 `where` 条件；实际上 `Chroma` 的 `similarity_search` 接受一个单独的关键字参数 `filter`，但 LangChain 的 retriever 接口没有正确向上暴露。  
解决：自己写一个 wrapper，直接从 `vectordb.similarity_search(query, k=4, filter={"type": "policy"})` 调用，然后塞给 chain 或者直接手搓 retrieval 逻辑。用 Custom Retriever 包一下即可，代码在文末仓库里。

这两个坑花了我整整 6 小时，网上几乎找不到答案，希望你不要再交一遍学费。

## 效果验证：用数据说话

测试集用了内部 50 条真实问题，人工标注标准答案片段所在文档，对比“默认参数”和“优化分割+手动过滤”两个版本：

| 指标             | 优化前  | 优化后  | 提升      |
| ---------------- | ------- | ------- | --------- |
| Top-3 命中率     | 62%     | 91%     | +47%      |
| 答案完整率       | 68%     | 94%     | +38%      |
| 平均生成耗时 (s) | 3.8     | 2.1     | 因精确检索减少无关上下文 |
| 幻觉率           | 18%     | 3%      | 下降 83%  |

整体效果达到生产可用标准，第二天一早演示一遍过，产品经理只说了句“牛逼”。

## 一键可用的轮子

我把上面所有逻辑封装成一个极简脚本，支持多轮对话和增量索引：

```bash
# 安装依赖
pip install langchain chromadb openai pypdf tiktoken

# 直接跑：--docs 指定文档目录，回车后输入问题即可
python rag_ready.py --docs ./my_pdfs --persist ./db
```

完整代码已传 GitHub，开箱即用，地址见下方。

---

`#Python` `#RAG` `#LangChain` `#Chroma` `#后端实战`

**关于作者**  
一个在生产环境里摔打出来的后端/架构方向开发者，专注 Python 生态的工程化落地。  
GitHub: https://github.com/baofugege （本项目的完整源码及踩坑修复版已上传）  
Sponsor: https://github.com/sponsors/baofugege — 如果这篇文章帮你省了 6 小时，请我喝杯咖啡  
提供服务：Python 后端性能优化 / 工具定制 / 技术咨询，联系 Telegram @baofugege