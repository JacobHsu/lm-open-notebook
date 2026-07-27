> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](security.md)

# 安全指南

本文件說明 Open Notebook 開發時的安全實踐。內容參考自透過與 [CERT-EU](https://cert.europa.eu) 協同揭露而發現的真實漏洞,所有貢獻者都應將其視為必讀內容。

## 回報漏洞

如果你發現安全漏洞,**請不要開公開的 GitHub issue**。請改為:

1. 使用 [GitHub Security Advisories](https://github.com/lfnovo/open-notebook/security/advisories/new) 私下回報
2. 或直接寄信給維護者

我們採用協同漏洞揭露(coordinated vulnerability disclosure),會在任何公開公告之前與你一起處理修復。

---

## 資料庫查詢(SurrealQL 注入)

**規則:絕不要透過 f-string 把使用者輸入內插進 SurrealQL 查詢中。**

SurrealQL 注入等同於 SQL 注入。使用者可控的值必須以 `$variable` 語法作為參數化的綁定變數傳入。

### 參數化查詢(安全)

```python
# Good: parameterized query
result = await repo_query(
    "SELECT * FROM source WHERE id = $id",
    {"id": ensure_record_id(source_id)}
)
```

### F-string 內插(有漏洞)

```python
# Bad: user input in f-string
result = await repo_query(f"SELECT * FROM source WHERE id = {source_id}")
```

### 無法參數化的 ORDER BY 及其他子句

`ORDER BY`、`LIMIT` 以及類似的子句,在 SurrealDB 中通常無法接受綁定變數。請改用**允許清單驗證(allowlist validation)**:

```python
# Good: validate against allowlist, then interpolate
allowed_fields = {"name", "created", "updated"}
allowed_directions = {"asc", "desc"}

parts = order_by.strip().lower().split()
if parts[0] not in allowed_fields:
    raise HTTPException(status_code=400, detail="Invalid sort field")
if len(parts) > 1 and parts[1] not in allowed_directions:
    raise HTTPException(status_code=400, detail="Invalid sort direction")

query = f"SELECT * FROM notebook ORDER BY {validated_order_by}"
```

排序參數驗證的參考實作請見 `api/routers/sources.py`。

### 檢查清單

- [ ] 所有使用者提供的值都使用 `$variable` 綁定
- [ ] 查詢中任何 f-string 只包含已驗證過或寫死的值
- [ ] `ORDER BY`、`LIMIT` 等使用允許清單驗證
- [ ] 用於後續查詢的資料庫值也一併參數化(避免二階注入)

---

## 樣板渲染(Server-Side Template Injection)

**規則:渲染含有使用者提供內容的 Jinja2 樣板時,一律使用 `SandboxedEnvironment`。**

[ai-prompter](https://github.com/lfnovo/ai-prompter) 函式庫(>= 0.4.0)預設使用 `SandboxedEnvironment`,會封鎖對危險 Python 屬性的存取,例如 `__globals__`、`__subclasses__` 與 `__init__`。

### SandboxedEnvironment 阻擋的內容

```jinja2
{# These are blocked and raise SecurityError #}
{{ cycler.__init__.__globals__.os.popen('id').read() }}
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

### 準則

- 絕不要把 ai-prompter 降版到 0.4.0 以下
- 如果直接使用 Jinja2(在 ai-prompter 之外),一律使用 `jinja2.sandbox.SandboxedEnvironment`
- 絕不要把使用者提供的字串直接傳給 `jinja2.Environment` 或 `jinja2.Template`

### 具有相同結構的第三方函式庫

`podcast_creator` 函式庫的 `configure("templates", {...})` 會把傳入的字串直接編譯為 Jinja2 樣板原始碼(其 `config.py` 中的 `Prompter(template_text=...)`)——與上述漏洞完全相同的模式。`commands/podcast_commands.py` 從未呼叫它(已確認:本儲存庫中的 podcast 生成一律使用以檔案為基礎的 `prompts/podcast/*.jinja` 樣板),所以目前處於休眠狀態,並非可被利用的漏洞。如果之後新增了「自訂 podcast 樣板」功能,請把使用者/設定檔文字導向一個固定、由開發者撰寫的樣板,並將文字以純變數方式傳入——不要把它接進 `configure("templates", ...)`。

---

## 檔案處理(路徑遍歷與本機檔案納入)

### 檔案上傳

**規則:一律清理檔名(sanitize)並驗證解析後的路徑。**

```python
import os
from pathlib import Path

# 1. Strip directory components
safe_filename = os.path.basename(original_filename)

# 2. Validate resolved path stays within target directory
resolved = (Path(upload_folder) / safe_filename).resolve()
if not str(resolved).startswith(str(Path(upload_folder).resolve()) + os.sep):
    raise ValueError("Path traversal detected")
```

重點:

- 使用 `os.path.basename()` 從使用者提供的檔名中去除目錄部分
- 使用 `Path.resolve()` 解析 symlink 與 `..` 部分
- 使用 `startswith()` 搭配**結尾的 `os.sep`**,以防止相鄰目錄繞過(例如 `/uploads_evil/` 誤符合 `/uploads`)

### 檔案路徑輸入

**規則:驗證任何使用者提供的檔案路徑都落在預期的目錄內。**

```python
uploads_resolved = Path(UPLOADS_FOLDER).resolve()
file_resolved = Path(user_provided_path).resolve()
if not str(file_resolved).startswith(str(uploads_resolved) + os.sep):
    raise HTTPException(status_code=400, detail="Invalid file path")
```

絕不要在未經驗證的情況下,把使用者提供的檔案路徑直接傳給讀檔或內容擷取函式。

### 檢查清單

- [ ] 上傳的檔名有用 `os.path.basename()` 清理過
- [ ] 解析後的路徑有用 `startswith(directory + os.sep)` 驗證過
- [ ] 使用者提供的 `file_path` 值在使用前有經過驗證
- [ ] 不會根據使用者輸入建立目錄(`mkdir` 搭配可遍歷路徑)

---

## 身分驗證與 CORS

### 身分驗證

Open Notebook 目前使用簡單的密碼式中介層(`PasswordAuthMiddleware`)。這適用於單一使用者的自架部署,但正式環境應加以強化:

- 明確設定 `OPEN_NOTEBOOK_PASSWORD`——沒有寫死的預設密碼;如果未設定,身分驗證會完全停用(所有請求都不經檢查直接通過)
- 更換預設的加密金鑰(`OPEN_NOTEBOOK_ENCRYPTION_KEY`)
- 考慮部署在具備正式身分驗證(OAuth、OIDC)的反向代理之後

### CORS

預設的 CORS 設定允許所有來源(`allow_origins=["*"]`)。`allow_credentials` 與此連動:預設萬用字元時為 `False`(避免 Starlette 在帶有 credentials 的情況下反射任意 Origin),一旦 `CORS_ORIGINS` 明確限縮為特定來源,就會自動變成 `True`。正式環境部署時,仍應將 `CORS_ORIGINS` 限制為僅前端網址。

---

## 機密資訊管理

### 加密金鑰

`OPEN_NOTEBOOK_ENCRYPTION_KEY` 用於加密儲存在 SurrealDB 中的 API 金鑰。在正式環境中:

- 設定一組強度足夠且獨一無二的金鑰(不要使用預設值)
- 盡可能透過 `OPEN_NOTEBOOK_ENCRYPTION_KEY_FILE` 使用 Docker secrets
- 絕不要記錄(log)或外洩這個值

### 環境變數

- 敏感值(API 金鑰、密碼、加密金鑰)絕不應出現在日誌中
- 謹慎使用 `loguru`——避免記錄完整的請求內容或環境變數傾印
- Docker 容器預設以 root 執行;應考慮改以非 root 使用者執行

---

## 程式碼審查安全檢查清單

審查 PR 時,請檢查以下項目:

1. **查詢注入**:SurrealQL 查詢中是否有任何包含使用者輸入的 f-string
2. **樣板注入**:是否有使用者提供的字串在未經沙盒化(sandboxing)的情況下傳給 Jinja2
3. **路徑遍歷**:使用者提供的檔名或路徑是否未經清理就被使用
4. **資訊洩漏**:錯誤訊息是否暴露內部路徑、堆疊追蹤或設定內容
5. **SSRF**:使用者提供的 URL 是否未經驗證就傳給伺服器端的 HTTP 請求
6. **日誌中的機密資訊**:是否有任何等級的日誌記錄了敏感值

---

## 過往漏洞

以下漏洞由 CERT-EU 回報,在此記錄作為學習範例:

| 版本 | 漏洞 | 嚴重程度 | 公告 |
|---------|--------------|----------|------|
| <= 1.8.2 | 透過 `order_by` 參數的 SurrealDB 注入 | High (8.7) | [GHSA-5wj9-f8q5-8f9c](https://github.com/lfnovo/open-notebook/security/advisories/GHSA-5wj9-f8q5-8f9c) |
| <= 1.8.3 | 透過 transformations 中的 Jinja2 SSTI 造成的 RCE | Critical (9.2) | [GHSA-f35w-wx37-26q7](https://github.com/lfnovo/open-notebook/security/advisories/GHSA-f35w-wx37-26q7) |
| <= 1.8.3 | 透過路徑遍歷造成的任意檔案寫入 | High (7.0) | [GHSA-x4q2-89g5-594v](https://github.com/lfnovo/open-notebook/security/advisories/GHSA-x4q2-89g5-594v) |
| <= 1.8.3 | 透過 LFI 造成的任意檔案讀取 | High (8.2) | [GHSA-842v-h4cj-r646](https://github.com/lfnovo/open-notebook/security/advisories/GHSA-842v-h4cj-r646) |
