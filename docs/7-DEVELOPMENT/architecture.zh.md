> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](architecture.md)

# Open Notebook 架構

## 高階概觀

Open Notebook 採用職責分明的三層架構:

```
┌─────────────────────────────────────────────────────────┐
│  Your Browser                                           │
│  Access: http://your-server-ip:8502                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
         ┌───────────────┐
         │   Port 8502   │  ← Next.js Frontend (what you see)
         │   Frontend    │    Also proxies API requests internally!
         └───────┬───────┘
                 │ proxies /api/* requests ↓
                 ▼
         ┌───────────────┐
         │   Port 5055   │  ← FastAPI Backend (handles requests)
         │     API       │
         └───────┬───────┘
                 │
                 ▼
         ┌───────────────┐
         │   SurrealDB   │  ← Database (internal, auto-configured)
         │   (Port 8000) │
         └───────────────┘
```

**重點:**
- **v1.1+**:Next.js 會自動將 `/api/*` 請求代理到後端,簡化了反向代理的設定
- 瀏覽器會從 8502 連接埠載入前端
- 前端需要知道 API 的位置——遠端存取時,請設定:`API_URL=http://your-server-ip:5055`
- **在反向代理後面?** 現在你只需要代理到 8502 連接埠!參見[反向代理設定](../5-CONFIGURATION/reverse-proxy.md)

---

## 詳細架構

Open Notebook 建構於**三層、非同步優先的架構**之上,設計目標是可擴展性、模組化,以及多供應商 AI 的彈性。系統將關注點分散在前端、API 與資料庫三層,由 LangGraph 驅動智慧工作流程,並由 Esperanto 提供與 17 家 AI 供應商的無縫整合。

**核心理念**:
- 隱私優先:使用者掌控自己的資料與 AI 供應商選擇
- 全程 async/await:非阻塞操作帶來回應迅速的使用體驗
- Domain-Driven Design:明確區分領域模型、repository 與 orchestrator
- 多供應商彈性:更換 AI 供應商不需修改應用程式碼
- 支援自架:所有元件都可以部署在隔離環境中

---

## 三層架構

### 第一層:前端(React/Next.js @ 3000 連接埠)

**目的**:提供研究、筆記、聊天與 Podcast 管理所需的、具回應性且互動式的使用者介面。

**技術堆疊**:
- **框架**:Next.js 15 搭配 React 19
- **語言**:TypeScript,啟用嚴格型別檢查
- **狀態管理**:Zustand(輕量 store)+ TanStack Query(伺服器端狀態)
- **樣式**:Tailwind CSS + Shadcn/ui 元件庫
- **建構工具**:Webpack(透過 Next.js 打包)

**主要職責**:
- 渲染 notebooks、sources、notes、聊天 session 與 podcasts
- 處理使用者互動(建立、讀取、更新、刪除操作)
- 管理複雜的 UI 狀態(modal、檔案上傳、即時搜尋)
- 串流來自 API 的回應(聊天、Podcast 生成)
- 顯示嵌入、向量搜尋結果與 insights

**通訊模式**:
- 所有資料都透過 REST API 非同步請求取得(連接埠 5055)
- 設定的基礎 URL:`http://localhost:5055`(開發環境)或依環境而定(正式環境)
- TanStack Query 負責快取、重新抓取與資料同步
- Zustand 儲存全域狀態(使用者、notebooks、選取的上下文)
- API 端已啟用 CORS 以支援跨來源請求

**元件架構**:
- `/src/app/`:Next.js App Router(頁面、layout)
- `/src/components/`:可重用的 React 元件(按鈕、表單、卡片)
- `/src/hooks/`:自訂 hooks(useNotebook、useChat、useSearch)
- `/src/lib/`:工具函式、API 客戶端、驗證器
- `/src/styles/`:全域 CSS、Tailwind 設定

---

### 第二層:API(FastAPI @ 5055 連接埠)

**目的**:對 notebooks、sources、notes、聊天 session 與 AI 模型提供操作介面的 RESTful 後端。

**技術堆疊**:
- **框架**:FastAPI 0.104+(非同步 Python web 框架)
- **語言**:Python 3.11+
- **驗證**:Pydantic v2(請求/回應 schema)
- **日誌**:Loguru(結構化 JSON 日誌)
- **測試**:Pytest(單元測試與整合測試)

**架構**:
```
FastAPI App (main.py)
  ├── Routers (HTTP endpoints)
  │   ├── routers/notebooks.py (CRUD operations)
  │   ├── routers/sources.py (content ingestion, upload)
  │   ├── routers/notes.py (note management)
  │   ├── routers/chat.py (conversation sessions)
  │   ├── routers/search.py (full-text + vector search)
  │   ├── routers/transformations.py (custom transformations)
  │   ├── routers/models.py (AI model configuration)
  │   └── routers/*.py (11 additional routers)
  │
  ├── Services (business logic)
  │   ├── *_service.py (orchestration, graph invocation)
  │   ├── command_service.py (async job submission)
  │   └── middleware (auth, logging)
  │
  ├── Models (Pydantic schemas)
  │   └── models.py (validation, serialization)
  │
  └── Lifespan (startup/shutdown)
      └── AsyncMigrationManager (database schema migrations)
```

**主要職責**:
1. **HTTP 介面**:接受 REST 請求、驗證、回傳 JSON 回應
2. **商業邏輯**:協調領域模型、repository 操作與工作流程
3. **非同步工作佇列**:提交長時間執行的任務(Podcast 生成、source 處理)
4. **資料庫遷移**:在啟動時執行 schema 更新
5. **錯誤處理**:攔截例外,回傳適當的 HTTP 狀態碼
6. **日誌**:追蹤操作以利除錯與監控

**啟動流程**:
1. 載入 `.env` 環境變數
2. 以 CORS + auth middleware 初始化 FastAPI app
3. 執行 AsyncMigrationManager(建立/更新資料庫 schema)
4. 註冊所有 routers(20+ 個端點)
5. 伺服器在連接埠 5055 上就緒

**請求—回應循環**:
```
HTTP Request → Router → Service → Domain/Repository → SurrealDB
                                       ↓
                                  LangGraph (optional)
                                       ↓
Response ← Pydantic serialization ← Service ← Result
```

---

### 第三層:資料庫(SurrealDB @ 8000 連接埠)

**目的**:內建向量嵌入、語意搜尋與關聯管理能力的圖形資料庫。

**技術堆疊**:
- **資料庫**:SurrealDB(多模型、ACID 交易)
- **查詢語言**:SurrealQL(類 SQL 語法,支援圖形操作)
- **非同步驅動**:Python 用的非同步 Rust 客戶端
- **遷移**:`open_notebook/database/migrations/` 底下的 `.surrealql` 檔案,註冊於 `AsyncMigrationManager`(於 API 啟動時自動執行)

**核心資料表**:

| 資料表 | 用途 | 主要欄位 |
|-------|---------|-----------|
| `notebook` | 研究專案容器 | id, name, description, archived, created, updated |
| `source` | 內容項目(PDF、URL、文字) | id, title, full_text, topics, asset, created, updated |
| `source_embedding` | 語意搜尋用的向量嵌入 | id, source, embedding, chunk_text, chunk_index |
| `note` | 使用者建立的研究筆記 | id, title, content, note_type (human/ai), created, updated |
| `chat_session` | 對話 session | id, notebook_id, title, messages (JSON), created, updated |
| `transformation` | 自訂轉換規則 | id, name, description, prompt, created, updated |
| `source_insight` | 轉換輸出結果 | id, source_id, insight_type, content, created, updated |
| `reference` | 關聯:source → notebook | out (source), in (notebook) |
| `artifact` | 關聯:note → notebook | out (note), in (notebook) |

**關聯圖**:
```
Notebook
  ↓ (referenced_by)
Source
  ├→ SourceEmbedding (1:many for chunked text)
  ├→ SourceInsight (1:many for transformation outputs)
  └→ Note (via artifact relationship)
    ├→ Embedding (semantic search)
    └→ Topics (tags)

ChatSession
  ├→ Notebook
  └→ Messages (stored as JSON array)
```

**向量搜尋能力**:
- 嵌入原生儲存於 SurrealDB 中
- 對 `source.full_text` 與 `note.content` 提供全文搜尋
- 對嵌入向量進行餘弦相似度搜尋
- 語意搜尋整合進 search 端點

**連線管理**:
- 非同步連線池(池大小可設定)
- 支援多筆紀錄操作的交易
- 透過遷移進行 schema 自動驗證
- 查詢逾時保護(避免無窮迴圈查詢)

---

## 技術堆疊選型理由

### 為什麼選 Python + FastAPI?

**Python**:
- 豐富的 AI/ML 生態系(LangChain、LangGraph、transformers、scikit-learn)
- 快速原型開發與部署
- 完善的非同步支援(asyncio、async/await)
- 強型別提示(Pydantic、mypy)

**FastAPI**:
- 現代、非同步優先的框架
- 自動產生 OpenAPI 文件(Swagger UI @ /docs)
- 內建請求驗證(Pydantic)
- 出色的效能(效能表現接近 C/Rust)
- 容易撰寫 middleware/依賴注入

### 為什麼選 Next.js + React + TypeScript?

**Next.js**:
- 支援 SSR/SSG 的全端 React 框架
- 基於檔案系統的路由(直覺的專案結構)
- 內建 API routes(可選擇性地將後端共置)
- 最佳化的 image/程式碼分割
- 容易部署(Vercel、Docker、自架)

**React 19**:
- 元件化 UI(可重用、可測試)
- 出色的工具鏈與社群
- 客戶端狀態管理(Zustand)
- 伺服器端狀態同步(TanStack Query)

**TypeScript**:
- 型別安全能在編譯期捕捉錯誤
- 更好的 IDE 自動完成與重構能力
- 透過型別達成文件化(程式碼自我說明)
- 讓新貢獻者更容易上手

### 為什麼選 SurrealDB?

**SurrealDB**:
- 原生圖形資料庫(關聯是第一等公民)
- 內建向量嵌入(不需要另外的向量資料庫)
- ACID 交易(資料一致性)
- 多模型(關聯式 + 文件式 + 圖形式)
- 一次查詢同時支援全文搜尋 + 語意搜尋
- 可自架(不像受管理的 Pinecone/Weaviate)
- 靈活的 SurrealQL(類 SQL 語法)

**曾考慮過的替代方案**:PostgreSQL + pgvector(更成熟,但需要另外的擴充套件)

### 為什麼用 Esperanto 處理 AI 供應商?

**Esperanto 函式庫**:
- 統一介面串接 17 家供應商(OpenAI、Anthropic、Google、Groq、Ollama、Mistral、DeepSeek、xAI、OpenRouter、Azure、Vertex 等)
- 多供應商嵌入(OpenAI、Google、Ollama、Mistral、Voyage)
- TTS/STT 整合(OpenAI、Groq、ElevenLabs、Google)
- 智慧供應商選擇(後備邏輯、成本最佳化)
- 支援逐請求模型覆寫
- 支援本機 Ollama(完全自架的選項)

**曾考慮過的替代方案**:LangChain 的供應商抽象層(較冗長、彈性較低)

---

## LangGraph 工作流程

LangGraph 是一個狀態機函式庫,用來編排多步驟的 AI 工作流程。Open Notebook 使用五種核心工作流程:

### 1. **Source 處理工作流程**(`open_notebook/graphs/source.py`)

**目的**:擷取內容(PDF、URL、文字)並準備好供搜尋/insights 使用。

**流程**:
```
Input (file/URL/text)
  ↓
Extract Content (content-core library)
  ↓
Clean & tokenize text
  ↓
Generate Embeddings (Esperanto)
  ↓
Create SourceEmbedding records (chunked + indexed)
  ↓
Extract Topics (LLM summarization)
  ↓
Save to SurrealDB
  ↓
Output (Source record with embeddings)
```

**State 字典**:
```python
{
  "content_state": {"file_path" | "url" | "content": str},
  "source_id": str,
  "full_text": str,
  "embeddings": List[Dict],
  "topics": List[str],
  "notebook_ids": List[str],
}
```

**呼叫來源**:Sources API(`POST /sources`)

---

### 2. **聊天工作流程**(`open_notebook/graphs/chat.py`)

**目的**:與 AI 模型進行多輪對話,並參照 notebook 的上下文。

**流程**:
```
User Message
  ↓
Build Context (selected sources/notes)
  ↓
Add Message to Session
  ↓
Create Chat Prompt (system + history + context)
  ↓
Call LLM (via Esperanto)
  ↓
Stream Response
  ↓
Save AI Message to ChatSession
  ↓
Output (complete message)
```

**State 字典**:
```python
{
  "session_id": str,
  "messages": List[BaseMessage],
  "context": Dict[str, Any],  # sources, notes, snippets
  "response": str,
  "model_override": Optional[str],
}
```

**主要特性**:
- 訊息歷史紀錄保存於 SurrealDB(SqliteSaver checkpoint)
- 透過 `build_context_for_chat()` 工具函式建構上下文
- Token 計數以避免超出上限
- 支援逐訊息模型覆寫

**呼叫來源**:Chat API(`POST /chat/execute`)

---

### 3. **Ask 工作流程**(`open_notebook/graphs/ask.py`)

**目的**:透過搜尋 sources 並綜合結果來回答使用者的問題。

**流程**:
```
User Question
  ↓
Plan Search Strategy (LLM generates searches)
  ↓
Execute Searches (vector + text search)
  ↓
Score & Rank Results
  ↓
Provide Answers (LLM synthesizes from results)
  ↓
Stream Responses
  ↓
Output (final answer)
```

**State 字典**:
```python
{
  "question": str,
  "strategy": SearchStrategy,
  "answers": List[str],
  "final_answer": str,
  "sources_used": List[Source],
}
```

**串流**:使用 `astream()` 即時發送更新(策略 → 逐步回答 → 最終答案)

**呼叫來源**:Search API(`POST /ask`,啟用串流)

---

### 4. **轉換工作流程**(`open_notebook/graphs/transformation.py`)

**目的**:對 sources 套用自訂轉換(擷取摘要、重點等)。

**流程**:
```
Source + Transformation Rule
  ↓
Generate Prompt (Jinja2 template)
  ↓
Call LLM
  ↓
Parse Output
  ↓
Create SourceInsight record
  ↓
Output (insight with type + content)
```

**轉換範例**:
- 摘要(五句話的總覽)
- 重點(條列清單)
- 引句(值得注意的摘錄)
- 問答(自動產生的問題與答案)

**呼叫來源**:Sources API(`POST /sources/{id}/insights`)

---

### 5. **Prompt 工作流程**(`open_notebook/graphs/prompt.py`)

**目的**:通用的 LLM 任務執行(例如自動產生筆記標題、內容分析)。

**流程**:
```
Input Text + Prompt
  ↓
Call LLM (simple request-response)
  ↓
Output (completion)
```

**使用場景**:筆記標題生成、內容分析等。

---

## AI 供應商整合模式

### ModelManager:集中式工廠

位於 `open_notebook/ai/models.py`,ModelManager 負責:

1. **供應商偵測**:檢查環境變數以確認可用的供應商
2. **模型選擇**:根據上下文大小與任務選擇最合適的模型
3. **後備邏輯**:若主要供應商無法使用,嘗試備援供應商
4. **成本最佳化**:簡單任務優先使用較便宜的模型
5. **Token 計算**:在呼叫 LLM 前估算成本

**用法**:
```python
from open_notebook.ai.provision import provision_langchain_model

# Get best LLM for context size
model = await provision_langchain_model(
    task="chat",  # or "search", "extraction"
    model_override="anthropic/claude-opus-4",  # optional
    context_size=8000,  # estimated tokens
)

# Invoke model
response = await model.ainvoke({"input": prompt})
```

### 多供應商支援

**LLM 供應商**:
- OpenAI(gpt-4、gpt-4-turbo、gpt-3.5-turbo)
- Anthropic(claude-opus、claude-sonnet、claude-haiku)
- Google(gemini-3.5-flash、gemini-2.5-pro)
- Groq(mixtral、llama-2)
- Ollama(本機模型)
- Mistral(mistral-large、mistral-medium)
- DeepSeek(deepseek-chat)
- xAI(grok)

**嵌入供應商**:
- OpenAI(text-embedding-3-large、text-embedding-3-small)
- Google(embedding-001)
- Ollama(本機嵌入)
- Mistral(mistral-embed)
- Voyage(voyage-large-2)

**TTS 供應商**:
- OpenAI(tts-1、tts-1-hd)
- Groq(無 TTS,後備使用 OpenAI)
- ElevenLabs(多語言語音)
- Google TTS(文字轉語音)

### 逐請求覆寫

每一次 LangGraph 呼叫都接受一個 `config` 參數來覆寫模型:

```python
result = await graph.ainvoke(
    input={...},
    config={
        "configurable": {
            "model_override": "anthropic/claude-opus-4"  # Use Claude instead
        }
    }
)
```

---

## 設計模式

### 1. **Domain-Driven Design(DDD)**

**領域物件**(`open_notebook/domain/`):
- `Notebook`:研究容器,與 sources/notes 建立關聯
- `Source`:內容項目(PDF、URL、文字),含有嵌入
- `Note`:使用者建立或 AI 生成的研究筆記
- `ChatSession`:某個 notebook 的對話歷史
- `Transformation`:用於擷取 insights 的自訂規則

**Repository 模式**:
- 資料庫存取層(`open_notebook/database/repository.py`)
- `repo_query()`:執行 SurrealQL 查詢
- `repo_create()`:插入紀錄
- `repo_upsert()`:合併紀錄
- `repo_delete()`:刪除紀錄

**Entity 方法**:
```python
# Domain methods (business logic)
notebook = await Notebook.get(id)
await notebook.save()
notes = await notebook.get_notes()
sources = await notebook.get_sources()
```

### 2. **非同步優先架構**

**所有 I/O 都是非同步的**:
- 資料庫查詢:`await repo_query(...)`
- LLM 呼叫:`await model.ainvoke(...)`
- 檔案 I/O:`await upload_file.read()`
- Graph 呼叫:`await graph.ainvoke(...)`

**好處**:
- 非阻塞的請求處理(FastAPI 能同時服務多個並行請求)
- 更好的資源利用率(等待 I/O 時不會卡住 CPU)
- 自然貼合 Python 的 async/await 語法

**範例**:
```python
@router.post("/sources")
async def create_source(source_data: SourceCreate):
    # All operations are non-blocking
    source = Source(title=source_data.title)
    await source.save()  # async database operation
    await graph.ainvoke({...})  # async LangGraph invocation
    return SourceResponse(...)
```

### 3. **Service 模式**

Services 負責協調領域物件、repository 與工作流程:

```python
# illustrative example (see api/podcast_service.py for a real service)
class NotebookService:
    async def get_notebook_with_stats(notebook_id: str):
        notebook = await Notebook.get(notebook_id)
        sources = await notebook.get_sources()
        notes = await notebook.get_notes()
        return {
            "notebook": notebook,
            "source_count": len(sources),
            "note_count": len(notes),
        }
```

**職責**:
- 驗證輸入(Pydantic)
- 協調資料庫操作
- 呼叫工作流程(LangGraph graphs)
- 處理錯誤並回傳適當的狀態碼
- 記錄操作日誌

### 4. **串流模式**

對於長時間執行的操作(ask 工作流程、Podcast 生成),以 Server-Sent Events 的形式串流結果:

```python
@router.post("/ask", response_class=StreamingResponse)
async def ask(request: AskRequest):
    async def stream_response():
        async for chunk in ask_graph.astream(input={...}):
            yield f"data: {json.dumps(chunk)}\n\n"
    return StreamingResponse(stream_response(), media_type="text/event-stream")
```

### 5. **工作佇列模式**

對於非同步背景任務(source 處理),使用 Surreal-Commands 工作佇列:

```python
# Submit job
command_id = await CommandService.submit_command_job(
    app="open_notebook",
    command="process_source",
    input={...}
)

# Poll status
status = await source.get_status()
```

---

## 服務間通訊模式

### 前端 → API

1. **REST 請求**(HTTP GET/POST/PUT/DELETE)
2. **JSON 請求/回應主體**
3. **標準 HTTP 狀態碼**(200、400、404、500)
4. **選用的串流**(長時間操作使用 Server-Sent Events)

**範例**:
```typescript
// Frontend
const response = await fetch("http://localhost:5055/sources", {
  method: "POST",
  body: formData,  // multipart/form-data for file upload
});
const source = await response.json();
```

### API → SurrealDB

1. **SurrealQL 查詢**(類似 SQL)
2. **非同步驅動**,搭配連線池
3. **型別安全的紀錄 ID**(record_id 語法)
4. **支援多步驟操作的交易**

**範例**:
```python
# API
result = await repo_query(
    "SELECT * FROM source WHERE notebook = $notebook_id",
    {"notebook_id": ensure_record_id(notebook_id)}
)
```

### API → AI 供應商(透過 Esperanto)

1. **Esperanto 統一介面**
2. **逐請求供應商覆寫**
3. **失敗時自動後備**
4. **Token 計數與成本估算**

**範例**:
```python
# API
model = await provision_langchain_model(task="chat")
response = await model.ainvoke({"input": prompt})
```

### API → 工作佇列(Surreal-Commands)

1. **非同步工作提交**
2. **發出即忘(fire-and-forget)模式**
3. **透過 `/commands/{id}` 端點輪詢狀態**
4. **工作完成回呼(選用)**

**範例**:
```python
# Submit async source processing
command_id = await CommandService.submit_command_job(...)

# Client polls status
response = await fetch(f"http://localhost:5055/commands/{command_id}")
status = await response.json()  # returns { status: "running|queued|completed|failed" }
```

---

## 資料庫 Schema 概觀

### 核心 Schema 結構

**資料表**(20+ 張):
- Notebooks(以 `archived` 欄位實現軟刪除)
- Sources(內容 + metadata)
- SourceEmbeddings(向量分塊)
- Notes(使用者建立 + AI 生成)
- ChatSessions(對話歷史)
- Transformations(自訂規則)
- SourceInsights(轉換輸出結果)
- Relationships(notebook→source、notebook→note)

**遷移**:
- 於 API 啟動時自動執行
- 位於 `open_notebook/database/migrations/`
- 依序編號(`1.surrealql`、`2.surrealql` 等),並明確註冊於 `AsyncMigrationManager`(不會自動探索)
- 於 `_sbl_migrations` 資料表中追蹤
- 透過 `N_down.surrealql` 檔案回滾(手動執行)

### 關聯模型

**圖形關聯**:
```
Notebook
  ← reference ← Source (many:many)
  ← artifact ← Note (many:many)

Source
  → source_embedding (one:many)
  → source_insight (one:many)
  → embedding (via source_embedding)

ChatSession
  → messages (JSON array in database)
  → notebook_id (reference to Notebook)

Transformation
  → source_insight (one:many)
```

**查詢範例**(取得某個 notebook 中所有 sources 及其計數):
```sql
SELECT id, title,
  count(<-reference.in) as note_count,
  count(<-embedding.in) as embedded_chunks
FROM source
WHERE notebook = $notebook_id
ORDER BY updated DESC
```

---

## 關鍵架構決策

### 1. **全程非同步**

所有 I/O 操作都是非阻塞的,以最大化並行處理能力與回應速度。

**取捨**:程式碼稍微複雜一些(async/await 語法) vs. 高吞吐量。

### 2. **從第一天就支援多供應商**

內建支援 17 家 AI 供應商,避免被單一供應商綁定。

**取捨**:ModelManager 增加的複雜度 vs. 彈性與成本最佳化。

### 3. **以 Graph 為核心的工作流程**

以 LangGraph 狀態機處理複雜的多步驟操作(ask、chat、transformations)。

**取捨**:較陡的學習曲線 vs. 可維護、可除錯的工作流程。

### 4. **自架資料庫**

用 SurrealDB 在同一套系統中同時處理圖形與向量搜尋(沒有外部依賴)。

**取捨**:需要自行承擔維運責任 vs. 簡化的架構與成本節省。

### 5. **長時間任務使用工作佇列**

非同步工作提交(source 處理、Podcast 生成)可避免請求逾時。

**取捨**:最終一致性 vs. 具回應性的使用體驗。

---

## 重要的特殊行為與注意事項

### API 啟動

- **每次啟動都會自動執行遷移**;請檢查日誌是否有錯誤
- **啟動 API 前必須先啟動 SurrealDB**(lifespan 中有連線測試)
- **Auth middleware 相當基本**(僅密碼驗證);正式環境請升級為 OAuth/JWT

### 資料庫操作

- **紀錄 ID 使用 SurrealDB 語法**(table:id 格式,例如「notebook:abc123」)
- **ensure_record_id()** 輔助函式可避免格式錯誤的 ID
- **軟刪除**透過 `archived` 欄位實現(資料不會被移除,只會被標記為非啟用)
- **時間戳記使用 ISO 8601 格式**(created、updated 欄位)

### LangGraph 工作流程

- **狀態持久化**透過 `/data/sqlite-db/` 中的 SqliteSaver 實現
- **沒有內建逾時機制**;長時間執行的工作流程可能會阻塞請求(建議使用串流以改善使用體驗)
- **模型後備**在主要供應商無法使用時會自動觸發
- **Checkpoint ID** 必須在每個 session 中維持唯一(避免碰撞)

### AI 供應商整合

- **Esperanto 函式庫**處理所有供應商 API(不會有直接的 API 呼叫)
- **逐請求覆寫**透過 RunnableConfig 實現(暫時性,不會持久化)
- **成本估算**透過 token 計數實現(不是 100% 精確,僅供參考)
- **後備邏輯**在主要模型失敗時會嘗試較便宜的模型

### 檔案上傳

- **儲存於 `/data/uploads/`** 目錄(而非資料庫)
- **唯一檔名生成**可避免覆寫(附加編號後綴)
- **Content-core 函式庫**可從 50+ 種檔案類型中擷取文字
- **大型檔案**可能會短暫阻塞 API(同步進行內容擷取)

---

## 效能考量

### 最佳化策略

1. **連線池**:SurrealDB 非同步驅動,池大小可設定
2. **查詢快取**:前端使用 TanStack Query(客戶端快取)
3. **嵌入重用**:向量搜尋使用預先計算好的嵌入
4. **分塊**:Sources 會被切分成分塊以提升搜尋相關性
5. **非同步操作**:非阻塞 I/O 以支援高並行量
6. **延遲載入**:前端只請求需要的資料(分頁)

### 瓶頸

1. **LLM 呼叫**:延遲取決於供應商(通常為 1-30 秒)
2. **嵌入生成**:所需時間與內容大小及供應商相關
3. **向量搜尋**:需要對所有嵌入計算相似度
4. **內容擷取**:source 處理中的同步操作

### 監控

- **API 日誌**:檢查 loguru 輸出以查看錯誤與緩慢的操作
- **資料庫查詢**:可透過管理 UI 取得 SurrealDB 的指標
- **Token 使用量**:透過 `estimate_tokens()` 工具函式估算
- **工作狀態**:輪詢 `/commands/{id}` 以取得非同步操作的狀態

---

## 擴充點

### 新增工作流程

1. 建立 `open_notebook/graphs/workflow_name.py`
2. 定義 StateDict 與節點函式
3. 用 `.add_node()` / `.add_edge()` 建構 graph
4. 在 `api/workflow_service.py` 中建立 service
5. 在 `api/main.py` 中註冊 router
6. 在 `tests/test_workflow.py` 中新增測試

### 新增資料模型

1. 在 `open_notebook/domain/model_name.py` 中建立模型
2. 繼承自 BaseModel(領域物件)
3. 實作 `save()`、`get()`、`delete()` 方法(CRUD)
4. 若需要複雜查詢,新增 repository 函式
5. 在 `open_notebook/database/migrations/` 中建立資料庫遷移(並在 `AsyncMigrationManager` 中註冊)
6. 在 `api/` 中新增 API 路由與模型

### 新增 AI 供應商

1. 為新供應商設定 Esperanto(參見 .env.example)
2. ModelManager 會透過環境變數自動偵測
3. 可透過逐請求設定覆寫(不需要修改程式碼)
4. 若供應商無法使用,測試後備邏輯是否正常運作

---

## 部署考量

### 開發環境

- 所有服務都在 localhost 上(3000、5055、8000)
- 檔案變動時自動重新載入(Next.js、FastAPI)
- 資料庫遷移支援熱重載
- 在 http://localhost:5055/docs 開啟 API 文件

### 正式環境

- **前端**:部署到 Vercel、Netlify 或 Docker
- **API**:Docker 容器(參見 Dockerfile)
- **資料庫**:SurrealDB 容器或受管理服務
- **環境**:妥善保護含有 API 金鑰的 .env 檔案
- **SSL/TLS**:反向代理(Nginx、CloudFlare)
- **速率限制**:在代理層新增
- **驗證**:以 OAuth/JWT 取代 PasswordAuthMiddleware
- **監控**:日誌彙整(CloudWatch、DataDog 等)

---

## 總結

Open Notebook 的架構為隱私優先、AI 驅動的研究工作提供了穩固的基礎。關注點分離(前端/API/資料庫)、非同步優先的設計,以及多供應商彈性,共同促成了快速開發與容易部署。LangGraph 工作流程負責編排複雜的 AI 任務,而 Esperanto 則抽象化了供應商的實作細節。最終呈現的是一套可擴展、可維護的系統,讓使用者能夠掌控自己的資料與 AI 供應商選擇。
