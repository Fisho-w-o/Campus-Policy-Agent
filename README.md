# 教务规章助手（岭南师范学院场景）

> 个人项目（独立完成），**非学校官方产品**。面向岭南师范学院公开教务规章：Hybrid 检索 + Rerank 之上接入 **Agent 多工具循环**——校内规定走知识库，公开/时效信息可走 Tavily 联网搜索。LangGraph 路径下按 `thread_id` 做服务端多轮记忆。前端演示名「教务规章助手」。并用 Ragas 与拒答行为小金标做对照评估。

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688.svg)](https://fastapi.tiangolo.com/)
[![RAG](https://img.shields.io/badge/RAG-Hybrid%20%2B%20Rerank-orange.svg)](https://www.langchain.com/)
[![Agent](https://img.shields.io/badge/Agent-Tools%20%2B%20LangGraph-green.svg)](https://langchain-ai.github.io/langgraph/)
[![Eval](https://img.shields.io/badge/Eval-Ragas-purple.svg)](https://github.com/explodinggradients/ragas)

---

## 项目亮点

1. **检索链路完整**：PDF 按页入库 → Recursive 切块 → BGE Embedding → Chroma → Query 改写 → Hybrid（向量 + BM25 + RRF）→ `bge-reranker-v2-m3` 精排 → 流式生成，并回传来源页码  
2. **Agent 多工具调度**：`POST /agent/stream` 上跑工具循环。默认 LangGraph（`summarize` → `think` → `act` → `observe`）；`AGENT_RUNNER=legacy` 可切回手写循环。校内规定走 `search_lingnan_knowledge_base`，公开/时效可走 `search_web_messages`（Tavily），Prompt 约束联网结果不能冒充官方规章  
3. **多轮记忆（LangGraph）**：前端只传 `question` + `thread_id`（UUID），不传聊天历史。Graph 用 SQLite checkpointer 按 `thread_id` 读写 state；轮次多了先摘要再删旧消息。`LegacyRunner` 忽略 `thread_id`，无跨请求记忆  
4. **有对照实验，不只调通**：用 **50** 道自建题 + Ragas，对比有无 Rerank；四项指标均提升，Context Precision 约 **0.73 → 0.79**，Answer Relevancy 约 **0.65 → 0.83**  
5. **拒答行为有回归集**：另建 **30** 题（该答 10 / 部分答 10 / 该拒 10），人工记误拒与幻觉；当前 **误拒 3、幻觉 0**，不单凭感觉调 Prompt  
6. **前后端可演示**：Streamlit 解析 SSE（`thought` / `token` / `action` / `observation` / `sources` / `done`）；校内来源 PDF 名/页码，联网来源标题+链接；同一问多次检索会合并来源。演示另有 Redis IP/全站限流（**429**）和前端会话次数上限 `DEMO_SESSION_LIMIT`

### Ragas 评估结果（50 题）

| 指标 | 无 Rerank | 有 Rerank | 差值 |
|------|----------:|----------:|-----:|
| Context Precision | 0.73 | **0.79** | +0.06 |
| Context Recall | 0.74 | **0.88** | +0.14 |
| Faithfulness | 0.86 | **0.89** | +0.03 |
| Answer Relevancy | 0.65 | **0.83** | +0.18 |

**怎么理解：** Rerank 改善进入生成的 Top3 质量（Precision / Relevancy 上升）；Recall 也会因最终 Top3 置换而变化，但**不能**捞到 Hybrid Top10 以外的块——两边仍 Recall=0 的题要回到切块 / 初筛上查。

### 拒答 / 幻觉行为回归（30 题）

| 指标 | 结果 |
|------|------|
| 题型 | 该答 10 / 部分答 10 / 该拒 10 |
| 行为准确 | **27/30** |
| 误拒 | **3/30**（心理电话、转专业名额、宿舍房型等「有制度缺精确字段」时整句拒答） |
| 幻觉 | **0/30**（该拒题未编造价格 / 时刻 / 网址等） |

金标与跑分：`evaluation/refusal_gold.json`、`evaluation/eval_refusal.py`。完整过程与逐题分析见 [docs/evaluation_report.md](./docs/evaluation_report.md)。

---

## 项目简介

直接问大模型教务规定，容易答错或编造。本项目把规章 PDF 切块并写入向量库，由 Agent 按需调用工具再生成：

- **校内规定**：优先 `search_lingnan_knowledge_base`（改写 → Hybrid → Rerank），只根据知识库作答  
- **公开/时效资讯**：可调用 `search_web_messages`（Tavily），须标明来自网页、不得冒充官方规章  
- **资料不足**：明确拒答或说明边界，不编造  

本地 MVP：后端负责 Agent 循环与 SSE；前端负责对话、工具步骤与来源。多轮上下文在 Graph 路径下由服务端 checkpoint 保存，前端不回传 `history`。完整规章 PDF **未公开收录**（体积与版权）；本地复现可自备同类 PDF 后按下方脚本入库。

---

## 系统架构

主路径是一次问答：前端 → FastAPI Agent → 工具调用 → 流式生成。用户（Streamlit）以 `question` + `thread_id` 请求 `POST /agent/stream`；后端跑 Agent 循环（校内规定走知识库检索，公开/时效可走联网搜索），以 SSE 流式返回思考、工具步骤、正文与来源。

**兼容路径：** `POST /chat/stream` 仍保留纯 RAG 链式问答（不经 Agent 工具循环，`text/plain` 流式文本 + 末尾来源 JSON），便于对照与评测脚本复用。

**离线入库（一次构建知识库）：** `data/pdfs` → 按页抽文本 → Recursive 切块 → BGE Embedding → `chroma_db`。

---

## 核心能力

| 能力 | 说明 |
|------|------|
| **RAG 闭环** | 入库 → 切块 → Embedding → 检索 → 精排 → 生成 |
| **Query 改写** | 检索前改写用户问题，提升召回稳定性 |
| **混合检索** | 稠密向量 + `rank_bm25`，RRF 融合；中文分词用 jieba |
| **二次精排** | Hybrid Top10 → `bge-reranker-v2-m3` → Top3 |
| **Agent 工具循环** | `bind_tools` + 多轮 tool_call；知识库 / 联网分流。默认 LangGraph；`AGENT_RUNNER=legacy` 切回手写循环 |
| **多轮记忆** | Graph：SQLite checkpointer + `thread_id`；前端不传 history。超过 5 轮先摘要再裁剪消息。Legacy 无此能力 |
| **联网搜索** | Tavily HTTP 检索，封装为 `search_web_messages`；校规结论仍以知识库为准 |
| **SSE 流式** | `POST /agent/stream`：`text/event-stream`（token / thought / action / observation / sources / error / done） |
| **演示限流** | 后端 Redis：IP 小时/日 + 全站日限额（**429**）；前端 `DEMO_SESSION_LIMIT`（默认 5）限制本会话提问次数 |
| **可溯源展示** | 知识库：PDF 名 + 页码；联网：标题 + 链接；同一问多次工具调用会合并来源 |
| **对照实验** | 切块 / Hybrid / Rerank / Ragas / 拒答小金标，见 `docs/` 与 `evaluation/` |
| **附带能力** | FastAPI 分层、JWT 登录、MySQL、用户接口 Redis 缓存（非主线） |

### 运行截图

**1. 知识库问答（保留入学资格 · 附 PDF 来源/页码）**

![校内规章问答：保留入学资格与 PDF 出处](./image/demo_rag_admission.png)



![校内规章问答：本研分情形与表格展示](./image/demo_rag_undergrad.png)

**2. 联网搜索（公开资讯 · 附网页标题与链接）**

![Agent 联网检索：高校人工智能人才培养相关公开新闻](./image/demo_web_search.png)

**FastAPI Swagger（/docs）**

![Swagger API 文档](./image/Swagger.png)

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 前端 | Streamlit、httpx（SSE 客户端） |
| API | FastAPI、Uvicorn、Pydantic |
| Agent | LangChain tools、`bind_tools`、`LegacyRunner` / LangGraph、SQLite checkpointer、DeepSeek |
| RAG | ChromaDB、LangChain、Recursive 切块、BGE Embedding、rank_bm25、jieba、bge-reranker、Query 改写 |
| 联网 | Tavily Search API、httpx |
| 评估 | Ragas、自建 `ground_truth.json`（50 题）、拒答行为集 `refusal_gold.json`（30 题） |
| 数据 / 安全 | MySQL、SQLAlchemy、Redis（限流 + 用户缓存）、JWT；对话 checkpoint 用独立 SQLite（非 MySQL） |
| 工程 | python-dotenv、pytest、Docker Compose |

---

## 实验与评估（摘要）

完整数据、设置与局限说明：[docs/evaluation_report.md](./docs/evaluation_report.md)  
评估脚本与结果：`evaluation/eval_with_ragas.py`、`evaluation/eval_refusal.py`、`evaluation/results_*.json`、`evaluation/refusal_run.json`

| 实验 | 做了什么 | 结论（简） |
|------|----------|------------|
| **切块** | Naive / Character / Recursive，以及 128 / 256 / 512 | Recursive 更利于保留条款边界；当前采用 chunk_size=256、overlap=50 |
| **混合检索** | Dense vs Hybrid（向量 + BM25 + RRF） | Hybrid 稳住关键字侧；问题逐渐变成「候选对了但排序不理想」 |
| **Rerank** | Hybrid Top10 → 精排 Top3，探针看位次变化 | 精排接进生产检索链路，职责是重排，不扩大召回 |
| **Ragas** | 50 题，有无 Rerank 四指标对照 | 四项均提升；Precision +0.06，Recall +0.14，Relevancy +0.18 |
| **拒答小金标** | 30 题：该答 10 / 部分答 10 / 该拒 10，人工记误拒与幻觉 | 行为准确 27/30；误拒 3、幻觉 0；短板是缺细节时过拒 |

> 说明：上表评估主要针对 **知识库 RAG 链路**（检索质量与拒答行为）。Agent + 联网工具是线上主路径，尚未单独做与 Ragas 同规模的对照评测。

---

## 目录结构

```text
lingnan-university-rag/
├── main.py                 # Streamlit「教务规章助手」入口
├── frontend/               # 前端（默认请求 /agent/stream）
│   ├── components/         # 侧栏 / 对话（思考 + 工具步骤 + 来源）
│   ├── api/                # httpx SSE 客户端（thread_id；sources 合并去重）
│   └── config.py           # API 地址、DEMO_SESSION_LIMIT、示例问题
├── app/                    # FastAPI 后端
│   ├── agent/              # Agent 循环、事件与工具
│   │   ├── loop.py         # run()：默认 GraphRunner；AGENT_RUNNER=legacy 切回手写循环
│   │   ├── graph.py        # LangGraph：摘要 / think / act / observe / GraphRunner
│   │   ├── events.py       # token / thought / action / observation / sources / error / done
│   │   ├── sse.py          # SSE 帧编码
│   │   └── tools.py        # 知识库工具 + 联网工具
│   ├── routers/            # users / auth / chat / agent
│   ├── schemas/            # AgentRequest 含 question + thread_id
│   ├── core/               # 配置（含 CHECKPOINT_SQLITE_PATH）、限流、Redis、鉴权
│   └── rag/
│       ├── chain.py        # 检索与上下文格式化
│       ├── rewrite.py      # Query 改写
│       ├── hybrid.py       # 混合检索
│       ├── rerank.py       # Rerank
│       └── web_search.py   # Tavily 联网搜索
├── scripts/                # PDF 入库、检索探针
├── evaluation/             # Ragas / 拒答回归评估与结果
├── docs/
│   └── evaluation_report.md
├── data/                   # pdfs 入库语料；checkpoints.sqlite（Graph 对话存档，默认不提交）
├── playground/             # 早期检索/切块等实验脚本（非主线，可不看）
├── tests/                  # SSE、工具 payload、Legacy / Graph runner、摘要触发
├── docker/                 # Compose：MySQL + Redis + 后端 + 前端
├── image/                  # README 用 Swagger / Streamlit 截图等
├── requirements.txt            # 后端 / Docker API 运行时
├── requirements-frontend.txt   # Streamlit 前端
├── requirements-dev.txt        # 本地全量：后端 + 前端 + 测试/评估
├── .env.example
└── README.md
```

> 目录说明以仓库已跟踪内容为准。本地未提交的实验文件（如更新的 playground 脚本）不写入上表，避免与 GitHub 不一致。

---

## 快速开始

> 阅读本 README 即可了解设计与实验结果。若要本地跑通，需要自备 API Key、MySQL、Redis，以及若干规章类 PDF。联网工具另需 Tavily Key。

### 1. 环境

需要 **Python ≥ 3.11**（Docker 镜像与徽章均为 3.11）。

```bash
git clone https://github.com/wuziqing2003/lingnan-university-rag.git
cd lingnan-university-rag
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
pip install -r requirements-dev.txt
copy .env.example .env
```

Docker 后端镜像只装运行时：`pip install -r requirements.txt`；前端镜像：`pip install -r requirements-frontend.txt`。

在 `.env` 中填写 `DEEPSEEK_API_KEY`、`SiliconFlow_API_KEY`、`TAVILY_API_KEY`、`DB_*`、`SECRET_KEY`、`REDIS_*`、演示限额 `DEMO_*` 等（**不要提交 `.env`**）。

本机用 Docker 只跑 MySQL / Redis 时，`DB_HOST` / `REDIS_HOST` 填 `127.0.0.1`。Compose 一键拉起全套时，后端容器内会覆盖为服务名 `mysql` / `redis`。

Agent 默认 `AGENT_RUNNER=graph`（`.env` 与代码缺省一致），带 SQLite 多轮记忆。对照旧循环可设 `AGENT_RUNNER=legacy`。checkpoint 文件默认 `data/checkpoints.sqlite`，可用 `CHECKPOINT_SQLITE_PATH` 覆盖。Graph 存档是 SQLite，**不要**把 MySQL URL 配给 checkpointer。

将 PDF 放到 `data/pdfs/` 后执行入库（生成 `chroma_db/`，该目录默认不提交）：

```bash
python scripts/ingest_pdfs.py
```

**推荐本地调试（改前端/Agent 时用）：** Docker 只起 MySQL + Redis，后端和 Streamlit 在仓库根用 Python 跑，才能用到当前源码。

```bash
cd docker
docker compose --env-file ../.env up -d mysql redis
```

也可用 Compose 拉起 MySQL、Redis、后端与前端（仍需本机先入库，API 镜像默认不含 `chroma_db`；前端是镜像里的代码，不含你本地未重建的改动）：

```bash
cd docker
docker compose --env-file ../.env up --build
```

### 2. 启动后端

必须在**项目根**执行（直接 `python app/main.py` 可能 `No module named 'app'`）：

```bash
python -m app.main
```

- API 文档：http://127.0.0.1:8000/docs  
- 健康检查：http://127.0.0.1:8000/health  

默认即为 LangGraph（`.env` 中 `AGENT_RUNNER=graph`，代码缺省也是 `graph`）。启动时 lifespan 会创建 SQLite checkpointer 并 compile 图。对照旧循环：

```bash
# Windows PowerShell
$env:AGENT_RUNNER="legacy"
python -m app.main
```

### 3. 启动前端

另开终端，仍在项目根：

```bash
streamlit run main.py
```

侧边栏显示「后端已连接」后即可提问。前端为每个浏览器会话生成 `thread_id`，每问只发送当前问题和该 id；点「清空对话」会换新 UUID，服务端当作新会话。

额度：前端 `DEMO_SESSION_LIMIT`（默认 5 次/会话）用尽后不再发请求；后端 Redis 超额返回 **429**，前端会展示说明。

### 4. 测试与评估（可选）

在项目根用 pytest（仓库已设 `pythonpath = .`）。不要 `python tests/xxx.py`，否则会 `No module named 'app'`。单测会加载 `app.core.config`，需已配置 `.env`。

```bash
pytest -v
python evaluation/eval_with_ragas.py --mode rerank
python evaluation/eval_refusal.py
```

当前测试覆盖 SSE 编解码、工具 payload、Legacy / Graph 事件顺序与工具失败回退、以及 Graph 摘要触发 / 同 thread 续聊。

### 流式问答示例

Agent（与前端一致，SSE）：

请求体必须带 `thread_id`（UUID），否则 **422**。同一对话复用同一 id，新对话换新 id。

```bash
curl -N -X POST http://127.0.0.1:8000/agent/stream ^
  -H "Content-Type: application/json" ^
  -H "Accept: text/event-stream" ^
  -d "{\"question\":\"研究生国家奖学金和国家助学金有什么区别？\",\"thread_id\":\"11111111-1111-1111-1111-111111111111\"}"
```

纯 RAG 兼容接口（纯文本流 + 末尾来源 JSON）：

```bash
curl -N -X POST http://127.0.0.1:8000/chat/stream ^
  -H "Content-Type: application/json" ^
  -d "{\"question\":\"研究生国家奖学金和国家助学金有什么区别？\"}"
```

---

## 当前不足

- 评估集现为 Ragas 50 题 / 拒答 30 题，结论仍不宜外推到全部规章问答；Agent + 联网路径尚未单独做同规模评测  
- 仍有题目两边 Context Recall 为 0：所需段落可能未进入 Hybrid Top10，Rerank 帮不上，需要回头查切块、分词或初筛  
- 拒答回归里仍有 3 题误拒：有相关制度但缺精确字段（电话 / 名额 / 宿舍房型）时，模型会整句拒答，偏「过拒」  
- 对照路径 `AGENT_RUNNER=legacy` 仍可用，但不读 `thread_id`、无跨请求记忆；演示续聊请保持默认 `graph`  
- 单图 + 一份系统提示同时绑定知识库与联网工具，规章/网页分流主要靠 Prompt，不是物理隔离的子图  
- Rerank / Tavily 依赖外部 API，会增加延迟与对密钥/网络的依赖  
- 模型无内置实时时钟：像「今天几号」这类问题需注入当前日期或增加时间工具，不能单靠知识库/网页碰运气  

---

## 附录：主要 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/health` | 健康检查 |
| POST | `/agent/stream` | **主接口**：`{ question, thread_id }`，SSE（`text/event-stream`） |
| POST | `/chat/stream` | 兼容：纯 RAG 流式问答（`text/plain`，答案后附带来源 JSON） |
| POST | `/user` | 用户注册 |
| GET | `/user/{id}` | 按 id 查询（Redis 缓存） |
| GET | `/user` | 用户分页列表 |
| POST | `/login` | 登录，返回 JWT |
| GET | `/profile` | 当前用户信息（Bearer Token） |

`/agent/stream` 事件类型：

| event | 含义 |
|-------|------|
| `thought` | 调用工具前的思考（LangGraph；前端画在步骤区） |
| `action` | 即将调用的工具名与参数 |
| `observation` | 工具返回摘要 |
| `token` | 最终回答的流式增量 |
| `sources` | 知识库页码或网页链接（同一问多批由前端合并去重） |
| `error` | 失败信息（工具失败时不假装已检索到规章） |
| `done` | 本轮结束 |
