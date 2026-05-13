# Module 01 总结：从零搭建 RAG 系统

## 1. LLM 是什么

LLM（大语言模型）本质上是一个"续写机器"：给定一段文字，它预测最合理的下一段。训练数据量极大，所以它的回答感觉像在和一个有智慧的人对话。

但 LLM 有三个核心限制：
- **知识截止日期**：只知道训练数据截止前的事
- **不知道你的数据**：不了解你的内部文档、数据库
- **会幻觉**：有时会自信地给出错误答案

---

## 2. RAG 是什么，为什么需要它

RAG = Retrieval-Augmented Generation（检索增强生成）。

直接问 LLM "我能加入这门课吗？" → 它给出泛泛的回答，因为它不知道课程的具体政策。

解决方法：把相关文档找出来，塞进 prompt，让 LLM 基于这些内容回答。

```
用户提问 → 搜索知识库 → 找到相关文档 → 拼进 prompt → LLM 生成答案
```

RAG 的三个组件：**Search（搜索）+ Prompt（提示词）+ LLM（大模型）**

---

## 3. 数据集

FAQ 数据来自 DataTalks.Club，包含多门课程的问答，每条文档结构如下：

```python
{
    'id': '0e38656cfb',
    'course': 'llm-zoomcamp',
    'section': 'General Course-Related Questions',
    'question': 'How do I submit homework?',
    'answer': '...'
}
```

- `text_fields`（question、section、answer）：用于全文搜索
- `keyword_fields`（course）：用于精确过滤，确保只返回当前课程的结果

---

## 4. Search（搜索）

用 **minsearch** 做关键词搜索：

```python
index = Index(text_fields=['question', 'section', 'answer'], keyword_fields=['course'])
index.fit(documents)

results = index.search(
    query,
    boost_dict={'question': 2.0, 'section': 0.5},  # question 更重要
    filter_dict={'course': 'llm-zoomcamp'},          # 只搜这门课
    num_results=5
)
```

**boost_dict**：给不同字段加权，question 出现匹配词比 section 更重要。  
**filter_dict**：精确过滤，只返回指定课程的结果。

minsearch 的局限：纯关键词匹配，搜 "enroll" 找不到写 "join" 的文档。这个问题由 Module 02 的向量搜索解决。

---

## 5. Prompt（提示词构建）

Prompt 分两部分：

**Instructions（系统指令，每次不变）**：
```
Your task is to answer questions from the course participants
based on the provided context. If the answer is not found,
respond with "I don't know."
```

**User Prompt（每次随问题变化）**：
```
QUESTION: {question}

CONTEXT:
{section}
Q: {question}
A: {answer}
...
```

Instructions 的作用是把 LLM 的回答"锚定"在提供的文档上，减少幻觉。

---

## 6. LLM 调用

发送 message history（消息历史）给 LLM：

```python
message_history = [
    {'role': 'system', 'content': INSTRUCTIONS},  # 系统指令
    {'role': 'user', 'content': prompt}            # 用户提问 + 检索到的文档
]
```

**Memory** 和 **RAG** 本质上是一回事：都是往 prompt 里塞信息。RAG 塞的是知识库文档，Memory 塞的是历史对话。LLM 本身无状态，"记忆"靠 prompt 实现。

---

## 7. 完整 RAG 流程

```python
def rag(query):
    search_results = search(query)       # 1. 搜索
    prompt = build_prompt(query, search_results)  # 2. 构建 prompt
    answer = llm(prompt)                 # 3. 调用 LLM
    return answer
```

---

## 8. 代码封装：ingest.py 和 rag_helper.py

为了复用，把代码拆成两个文件：

**ingest.py**：负责数据加载和索引构建
- `load_faq_data()`：从网上拉 FAQ 数据
- `build_index(documents)`：用 minsearch 建索引（内存）
- `build_sqlite_index(documents)`：用 sqlitesearch 建索引（磁盘）

**rag_helper.py**：负责 RAG 逻辑，封装成 `RAGBase` 类
- `search()` / `build_prompt()` / `llm()` / `rag()`
- index 和 llm_client 作为参数传入，方便替换后端

用法：
```python
assistant = RAGBase(index=index, llm_client=client)
answer = assistant.rag('Can I still join the course?')
```

---

## 9. 持久化索引：Ingestion 与 Querying 分离

**minsearch 的问题**：索引在内存里，程序重启就消失，每次都要重新建索引。

**解决方案**：把数据处理和查询拆开。

| | minsearch | sqlitesearch |
|---|---|---|
| 存储位置 | 内存，重启消失 | 磁盘（faq.db），永久保存 |
| 每次启动 | 重新建索引 | 直接打开现有索引 |
| 适合规模 | 小数据 | 大数据、生产环境 |

**两步流程**：
```
Ingestion（只跑一次）：拉数据 → 写入 faq.db
Querying（每次查询）：打开 faq.db → 搜索 → 返回结果
```

因为 sqlitesearch 和 minsearch 的 API 完全一样，替换时 RAG 代码一行不用改，只换 index 对象。

---

## 10. RAG vs 微调

为什么用 RAG 而不是微调（Fine-tuning）？

- 微调需要 GPU 和专门工具，成本高
- 数据更新时需要重新训练，维护麻烦
- RAG 更灵活，任何 LLM 都能用，数据随时可更新

**结论：先用 RAG，只有在 RAG 真的不够用时才考虑微调。**

---

## 后续模块预告

- **Module 02**：向量搜索（Vector Search）—— 理解语义而不只是关键词匹配
- **Module 03**：Agents —— 让 LLM 智能决定搜什么、怎么搜
- **Elasticsearch**：生产级搜索引擎，支持全文搜索 + 向量搜索 + 分布式扩展
