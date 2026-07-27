> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](change-playbooks.md)

# 變更手冊

Open Notebook 程式碼庫中常見變更類型的逐步操作指南。每份手冊都會**依序**列出要動到的檔案、每個步驟該做什麼,以及該測試什麼。

> **給 AI 代理人:**在動手實作前,請先閱讀相關的手冊。請依照順序進行——跳過步驟會導致變更不完整,進而破壞其他層。

---

## 如何使用本文件

1. 確認你的 Issue 需要哪一種變更類型
2. 依照手冊逐步進行
3. 如果一項變更橫跨多種類型(例如新增欄位 + 新增端點),請合併相關的手冊
4. 若有疑問,閱讀程式碼庫中既有的範例——透過 `git log` 查看最近類似的變更

---

## 手冊:為既有模型新增欄位

**範例:**「為 Source 新增 `language` 欄位」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `open_notebook/domain/<model>.py` | 新增帶有型別提示與預設值的欄位。遵循類別中既有的模式。 |
| 2 | `open_notebook/database/migrations/N.surrealql` | 建立遷移。使用序列中的下一個編號。新增欄位用 `DEFINE FIELD`,回填既有紀錄用 `UPDATE`。在 `AsyncMigrationManager`(`async_migrate.py`)中註冊它——遷移不是自動探索的。 |
| 3 | `api/models.py` | 在 `*Create`、`*Update`(Optional)與 `*Response` schema 中新增欄位。 |
| 4 | `frontend/src/lib/types/api.ts` | 在對應的 TypeScript 介面(`*Response`、`Create*Request`、`Update*Request`)中新增欄位。 |
| 5 | 前端元件(如果是使用者可見的) | 在相關元件中顯示或編輯該欄位。 |
| 6 | `frontend/src/lib/locales/*/` | 如果該欄位有使用者可見的標籤,新增 i18n 字串。全部 7 個語系都要加。 |
| 7 | 測試 | 新增/更新涵蓋新欄位的測試——至少要有建立/讀取的 API 測試。 |

**驗證:**重新啟動 API(遷移會自動執行),檢查日誌確認遷移成功,透過 `/docs` 測試。

---

## 手冊:新增 API 端點

**範例:**「新增將 notebook 匯出為 PDF 的端點」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `api/models.py` | 定義請求/回應的 Pydantic schema。命名方式:`<Feature>Request`、`<Feature>Response`。 |
| 2 | `api/routers/<resource>.py` | 在既有的 router 中新增端點,或者如果是新資源,建立新的 router 檔案。遵循以下模式:驗證 → 呼叫 service → 回傳回應。 |
| 3 | `api/<resource>_service.py` | 商業邏輯放在這裡,而不是放在 router 中。如有需要,建立新的 service 檔案。 |
| 4 | `api/main.py` | 若是新的 router 檔案:用 `app.include_router()` 註冊。 |
| 5 | `frontend/src/lib/types/api.ts` | 新增與 Pydantic schema 對應的 TypeScript 型別。 |
| 6 | `frontend/src/lib/api/<resource>.ts` | 在 API 模組中新增方法。遵循既有模式(axios 呼叫,回傳 `response.data`)。 |
| 7 | `frontend/src/lib/hooks/use-<resource>.ts` | 新增 React Query hook。GET 用 `useQuery`,POST/PUT/DELETE 用 `useMutation`。要包含快取失效(cache invalidation)與 toast。 |
| 8 | 前端元件/頁面 | 在 UI 中串接這個 hook。 |
| 9 | 測試 | API 測試(狀態碼、驗證、錯誤情況)。 |

**命名慣例:**
- Routers:`@router.get("/resources/{id}")`(複數、小寫,多字詞用 kebab-case)
- Services:函式為 `async`,以描述性的方式命名(`process_source`、`generate_podcast`)
- Hooks:清單用 `useResources()`、單一項目用 `useResource(id)`、mutation 用 `useCreateResource()`

---

## 手冊:新增 LangGraph 工作流程

**範例:**「新增摘要工作流程」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `prompts/<workflow_name>/*.jinja` | 建立 Jinja2 prompt 樣板。使用 ai-prompter 的 `Prompter`。 |
| 2 | `open_notebook/graphs/<workflow_name>.py` | 定義 `StateDict`(TypedDict)、節點函式,並用 `StateGraph` 建立圖。使用 `provision_langchain_model()` 選擇模型。用 `classify_error()` 包裝 LLM 呼叫。 |
| 3 | `api/<resource>_service.py` | 呼叫圖:`await graph.ainvoke(state, config)`。 |
| 4 | `api/routers/<resource>.py` | 開放觸發此工作流程的端點。 |
| 5 | `commands/<workflow>_commands.py` | 如果此工作流程應以非同步方式執行:用 `CommandInput`/`CommandOutput` 建立指令。在 command service 中註冊。 |
| 6 | 前端整合 | API 模組 → hook → 元件。 |
| 7 | 測試 | 用模擬(mocked)的 LLM 回應個別測試圖中的節點。 |

**關鍵模式:**
- 節點是同步函式(LangGraph 的要求),但可以透過 ThreadPoolExecutor 呼叫非同步程式碼
- 用 `classify_error()` 把原始例外轉換成型別化的 `OpenNotebookError` 子類別
- 用 `provision_langchain_model()` 選擇模型——絕不要寫死某個供應商
- 狀態是 TypedDict,而不是 Pydantic 模型

---

## 手冊:錯誤修正(單一層級)

**範例:**「sources 端點的 order_by 參數失效」

| 步驟 | 該做什麼 |
|------|------------|
| 1 | **確認是哪一層。**閱讀 Issue,判斷是:前端、API router、service、domain 模型、資料庫,還是 graph。 |
| 2 | **閱讀相關的 AGENTS.md**(根目錄、`open_notebook/` 或 `frontend/`)以及 `docs/7-DEVELOPMENT/` 中對應的頁面。它們記錄了規則與注意事項。 |
| 3 | **重現。**使用 API 文件(`/docs`)、瀏覽器或測試來確認錯誤。 |
| 4 | **修正。**做出所需的最小變更。不要重構周邊的程式碼。 |
| 5 | **新增測試**,重現該錯誤並驗證修正結果。 |
| 6 | **執行既有測試**以驗證沒有回歸:`uv run pytest tests/` |

---

## 手冊:錯誤修正(跨層級)

**範例:**「透過 URL 建立來源後,沒有顯示在 notebook 中」

| 步驟 | 該做什麼 |
|------|------------|
| 1 | **追蹤資料流。**從使用者看到問題的地方(前端)開始,往回追蹤:元件 → hook → API 呼叫 → router → service → domain → 資料庫。 |
| 2 | **找出鏈路在哪裡斷掉。**用 API 文件獨立於前端測試後端。用 SurrealDB 查詢確認資料是否已持久化。 |
| 3 | **在正確的層級修正。**如果錯誤出在 service,不要在前端修補症狀。 |
| 4 | **修正後驗證整條鏈路。** |
| 5 | **在錯誤發生的那一層新增測試。** |

---

## 手冊:資料庫遷移

**範例:**「在 source.notebook_id 上新增索引以提升查詢效能」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `open_notebook/database/migrations/N.surrealql`(+ `N_down.surrealql`) | 撰寫 SurrealQL。使用序列中的下一個編號。參考既有遷移的寫法。 |
| 2 | `open_notebook/database/async_migrate.py` | 在 `AsyncMigrationManager.__init__` 中註冊新檔案——遷移是寫死的,不是自動探索的。 |
| 3 | Domain 模型(如有 schema 變更) | 更新欄位定義以保持一致。 |
| 4 | API schema(如有新增/變更欄位) | 更新 Pydantic 模型。 |
| 5 | **驗證:**重新啟動 API 並檢查日誌 | 遷移會在啟動時自動執行。查看 Loguru 輸出中是否有錯誤。 |

**重要事項:**
- 遷移有編號,並依序執行
- 它們會被記錄在 `_sbl_migrations` 資料表中——不會重複執行
- 每個需要遷移的 PR 只包含一個遷移,依合併順序編號;一旦遷移已經上到 main 就絕不合併多個遷移(開發用映像會立刻套用它)——見 [ADR-006](decisions/ADR-006-migration-granularity.md)
- 對於破壞性變更(DROP FIELD),要考量資料保存
- 用既有資料測試,而不是只用空的資料庫

---

## 手冊:僅限前端的變更

**範例:**「改善 notebook 清單的載入狀態」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | 找出元件 | 元件位於 `frontend/src/app/`(頁面)或 `frontend/src/components/`(共用元件)。 |
| 2 | 進行變更 | 遵循既有模式:函式型元件、用 hook 管理狀態、用 Tailwind 做樣式。 |
| 3 | i18n 字串 | 如果新增了使用者可見的文字,要新增到 `frontend/src/lib/locales/` 底下的所有語系檔案。 |
| 4 | 在瀏覽器中測試 | 檢查響應式排版、深色模式(如適用)、載入狀態、空狀態、錯誤狀態。 |

**關鍵模式:**
- 使用 hook 的元件,檔案最上方要有 `'use client'` 指令
- 狀態管理:本地狀態用 `useState`,全域狀態用 Zustand,伺服器狀態用 TanStack Query
- 樣式:Tailwind 工具類別,以及來自 `components/ui/` 的 Shadcn/ui 元件
- 型別:定義在 `lib/types/api.ts`,到處匯入使用

---

## 手冊:新增背景指令(Background Command)

**範例:**「新增指令,重建某個 notebook 的所有 embedding」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `commands/<name>_commands.py` | 定義 `CommandInput` 與 `CommandOutput` 這兩個 Pydantic 類別。撰寫指令函式。 |
| 2 | 註冊指令 | 加入 command service,讓它可以透過 `CommandService.submit_command_job()` 提交。 |
| 3 | API 端點 | 新增用來提交該指令、並回傳指令 ID 的端點。 |
| 4 | 前端(輪詢) | 使用 `/commands/{command_id}` 端點輪詢狀態。向使用者顯示進度。 |

**模式:**
- 指令是 fire-and-forget(送出即結束):提交後會立即回傳一個指令 ID
- 重試設定:`max_attempts`、`stop_on` 例外(ValueError = 不重試)
- 針對暫時性失敗,採用帶抖動(jitter)的指數退避(exponential backoff)

---

## 手冊:i18n / 翻譯更新

**範例:**「為新的設定頁面新增翻譯」

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `frontend/src/lib/locales/en-US/index.ts` | 先新增英文字串。依功能分組。 |
| 2 | 所有其他語系檔案 | 把相同的鍵值新增到:`pt-BR`、`zh-CN`、`zh-TW`、`ja-JP`、`ru-RU`、`bn-IN`。若尚無翻譯,先用英文佔位。 |
| 3 | 元件 | 使用 `const { t } = useTranslation()`,並透過 `t('section.key')` 存取。 |

**總共 7 個語系。**不要漏掉任何一個。

### 新增一整個新語言

| 步驟 | 檔案 | 該做什麼 |
|------|---------|------------|
| 1 | `frontend/src/lib/locales/<code>/index.ts` | 從 `en-US/index.ts` 複製結構,並翻譯所有字串。 |
| 2 | `frontend/src/lib/locales/index.ts` | 註冊該語系:匯入它、加入 `resources`,並加入 `languages` 陣列(`{ code, label }`)。 |
| 3 | `frontend/src/lib/utils/date-locale.ts` | 匯入對應的 `date-fns/locale`,並加入 `LOCALE_MAP`。 |
| 4 | **測試** | 透過 UI 的語言切換器切換語言;缺少的鍵值會回退到 en-US。 |

---

## 快速參考:各層級的檔案位置

| 層級 | 位置 | Schema/型別 | 測試 |
|-------|----------|-------------|-------|
| Domain 模型 | `open_notebook/domain/` | Pydantic 欄位 | `tests/` |
| 資料庫 | `open_notebook/database/repository.py` | SurrealQL | `tests/` |
| 遷移 | `open_notebook/database/migrations/*.surrealql` | SurrealQL | 啟動時自動執行 |
| AI/LLM | `open_notebook/ai/` | Esperanto 型別 | `tests/` |
| Graphs | `open_notebook/graphs/` | TypedDict 狀態 | `tests/` |
| Prompts | `prompts/**/*.jinja` | Jinja2 context | — |
| Commands | `commands/` | CommandInput/Output | `tests/` |
| API routers | `api/routers/` | `api/models.py` | `tests/` |
| API services | `api/*_service.py` | — | `tests/` |
| 前端型別 | `frontend/src/lib/types/` | TypeScript 介面 | — |
| 前端 API | `frontend/src/lib/api/` | — | — |
| 前端 hooks | `frontend/src/lib/hooks/` | — | `frontend/src/test/` |
| 前端元件 | `frontend/src/components/` | Props 介面 | `frontend/src/test/` |
| 前端頁面 | `frontend/src/app/` | — | — |
| i18n | `frontend/src/lib/locales/` | — | — |
