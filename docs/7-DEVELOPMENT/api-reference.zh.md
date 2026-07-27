> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](api-reference.md)

# API 參考

Open Notebook 的完整 REST API。所有端點都由 API 後端提供(預設:`http://localhost:5055`)。

**基礎 URL**:`http://localhost:5055`(開發環境)或依環境而定的正式環境 URL

**互動式文件**:使用 FastAPI 內建的 Swagger UI,網址是 `http://localhost:5055/docs`,可進行即時測試與探索。這是所有端點、請求/回應 schema 與即時測試的主要參考來源。

---

## 快速上手

### 1. 身分驗證

簡單的密碼式驗證(僅限開發環境):

```bash
curl http://localhost:5055/api/notebooks \
  -H "Authorization: Bearer your_password"
```

**⚠️ 正式環境**:請改用 OAuth/JWT。詳情請見 [Security Configuration](../5-CONFIGURATION/security.md)。

### 2. 基本 API 流程

大多數操作都遵循以下模式:
1. 建立一個 **Notebook**(研究內容的容器)
2. 新增 **Sources**(PDF、URL、文字)
3. 透過 **Chat** 或 **Search** 查詢
4. 檢視結果與 **Notes**

### 3. 測試端點

與其死背端點,不如使用互動式 API 文件:
- 前往 `http://localhost:5055/docs`
- 直接在瀏覽器中嘗試請求
- 即時查看請求/回應 schema
- 用你自己的資料測試

---

## API 端點總覽

### 主要資源類型

**Notebooks** - 包含 sources 與 notes 的研究專案
- `GET/POST /notebooks` - 列出與建立
- `GET/PUT/DELETE /notebooks/{id}` - 讀取、更新、刪除

**Sources** - 內容項目(PDF、URL、文字)
- `GET/POST /sources` - 列出與新增內容
- `GET /sources/{id}` - 取得來源詳細資訊
- `POST /sources/{id}/retry` - 重試失敗的處理
- `GET /sources/{id}/download` - 下載原始檔案

**Notes** - 使用者建立或 AI 產生的研究筆記
- `GET/POST /notes` - 列出與建立
- `GET/PUT/DELETE /notes/{id}` - 讀取、更新、刪除

**Chat** - 對話式 AI 介面
- `GET/POST /chat/sessions` - 管理聊天 session
- `POST /chat/execute` - 傳送訊息並取得回應
- `POST /chat/context` - 為聊天準備上下文

**Search** - 透過文字或語意相似度尋找內容
- `POST /search` - 全文或向量搜尋
- `POST /ask` - 提出問題(搜尋 + 綜合彙整)

**Transformations** - 用於萃取洞察的自訂 prompt
- `GET/POST /transformations` - 建立自訂萃取規則
- `POST /sources/{id}/insights` - 將 transformation 套用到某個 source

**Models** - 設定 AI 供應商
- `GET /models` - 可用的模型
- `GET /models/defaults` - 目前的預設值
- `POST /models/config` - 設定預設值

**Credentials** - 管理 AI 供應商憑證
- `GET/POST /credentials` - 列出與建立憑證
- `GET/PUT/DELETE /credentials/{id}` - CRUD 操作
- `POST /credentials/{id}/test` - 測試連線
- `POST /credentials/{id}/discover` - 從供應商探索模型
- `POST /credentials/{id}/register-models` - 註冊探索到的模型
- `GET /credentials/status` - 供應商狀態總覽
- `GET /credentials/env-status` - 環境變數狀態
- `POST /credentials/migrate-from-env` - 把環境變數遷移為憑證

**Health & Status**
- `GET /health` - 健康檢查
- `GET /commands/{id}` - 追蹤非同步操作

---

## 身分驗證

### 目前作法(開發環境)

所有請求都需要密碼標頭:

```bash
curl -H "Authorization: Bearer your_password" http://localhost:5055/api/notebooks
```

密碼透過 `OPEN_NOTEBOOK_PASSWORD` 環境變數設定。

> **📖 完整的身分驗證設定、API 範例與正式環境強化,請見 [Security Configuration](../5-CONFIGURATION/security.md)。**

### 正式環境

**⚠️ 不安全。**請改用:
- OAuth 2.0(建議)
- JWT tokens
- API keys

正式環境設定請見 [Security Configuration](../5-CONFIGURATION/security.md)。

---

## 常見模式

### 分頁

```bash
# List sources with limit/offset
curl 'http://localhost:5055/sources?limit=20&offset=10'
```

### 篩選與排序

```bash
# Filter by notebook, sort by date
curl 'http://localhost:5055/sources?notebook_id=notebook:abc&sort_by=created&sort_order=asc'
```

### 非同步操作

某些操作(來源處理、podcast 生成)會立即回傳一個指令 ID:

```bash
# Submit async operation
curl -X POST http://localhost:5055/sources -F async_processing=true
# Response: {"id": "source:src001", "command_id": "command:cmd123"}

# Poll status
curl http://localhost:5055/commands/command:cmd123
```

### 串流回應

`/ask` 端點以 Server-Sent Events 的方式串流回應:

```bash
curl -N 'http://localhost:5055/ask' \
  -H "Content-Type: application/json" \
  -d '{"question": "What is AI?"}'

# Outputs: data: {"type":"strategy",...}
#          data: {"type":"answer",...}
#          data: {"type":"final_answer",...}
```

### Multipart 檔案上傳

```bash
curl -X POST http://localhost:5055/sources \
  -F "type=upload" \
  -F "notebook_id=notebook:abc" \
  -F "file=@document.pdf"
```

---

## 錯誤處理

所有錯誤都會回傳帶有狀態碼的 JSON:

```json
{"detail": "Notebook not found"}
```

### 常見狀態碼

| 代碼 | 意義 | 範例 |
|------|---------|---------|
| 200 | 成功 | 操作完成 |
| 400 | Bad Request | 輸入無效 |
| 404 | Not Found | 資源不存在 |
| 409 | Conflict | 資源已存在 |
| 500 | Server Error | 資料庫/處理錯誤 |

---

## 給開發者的建議

1. **從互動式文件開始**(`http://localhost:5055/docs`)——這是最權威的參考來源
2. **啟用日誌記錄**以利除錯(查看 API 日誌:`docker logs`)
3. **串流端點**需要特別處理(Server-Sent Events,而非標準 JSON)
4. **非同步操作**會立即回傳;在確認完成前務必先輪詢狀態
5. **向量搜尋**需要設定好 embedding 模型(檢查 `/models`)
6. **模型覆寫(overrides)**是逐請求設定的;寫在 body 中,而不是設定檔中
7. **CORS 在開發環境中是開放的**;正式環境請自行設定

---

## 學習路徑

1. **身分驗證**:在所有請求中加上 `X-Password` 標頭
2. **建立一個 notebook**:`POST /notebooks`,帶上名稱與描述
3. **新增一個 source**:`POST /sources`,帶上檔案、URL 或文字
4. **查詢你的內容**:`POST /chat/execute` 來提出問題
5. **探索進階功能**:搜尋、transformations、串流

---

## 正式環境考量

- 用 OAuth/JWT 取代密碼驗證(見 [Security](../5-CONFIGURATION/security.md))
- 透過反向代理(Nginx、CloudFlare、Kong)新增速率限制(rate limiting)
- 啟用 CORS 限制(目前允許所有來源)
- 透過反向代理使用 HTTPS(見 [Reverse Proxy](../5-CONFIGURATION/reverse-proxy.md))
- 建立 API 版本策略(目前是隱含的)

完整的正式環境設定,請見 [Security Configuration](../5-CONFIGURATION/security.md) 與 [Reverse Proxy Setup](../5-CONFIGURATION/reverse-proxy.md)。
