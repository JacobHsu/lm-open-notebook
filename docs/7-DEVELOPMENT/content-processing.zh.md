> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](content-processing.md)

# 內容處理:分塊、嵌入、上下文與加密

`open_notebook/utils/` 底下這些工具程式的設計筆記,負責把原始內容轉換成可搜尋、可供 LLM 使用的資料。這些工具是橫切關注點:sources、notes 與 insights 都會流經它們。

## 分塊(`utils/chunking.py`)

內容會用能感知內容類型的 LangChain 切分器(`HTMLHeaderTextSplitter`、`MarkdownHeaderTextSplitter`、`RecursiveCharacterTextSplitter`)來切分。內容類型偵測優先使用副檔名;當信心度 ≥ 0.8 時,啟發式判斷可以覆寫 PLAIN 副檔名的結果。HTML/Markdown 切分器產生的過大分塊會再做一次二次切分。

**為什麼預設是 400 token。** `OPEN_NOTEBOOK_CHUNK_SIZE` 預設為 400 token——比 BERT 系列嵌入模型(例如 `mxbai-embed-large`)512 token 的上限低約 20%。這個緩衝空間用來吸收三種誤差來源:tokenizer 不一致(我們用 `o200k_base` 計算,嵌入模型則用 WordPiece 斷詞)、切分器的超量,以及特殊 token。對於視窗較大的嵌入模型(OpenAI `text-embedding-3` 系列:8191 token),可以調高這個值,例如:

```bash
export OPEN_NOTEBOOK_CHUNK_SIZE=1500
export OPEN_NOTEBOOK_CHUNK_OVERLAP=150
```

`OPEN_NOTEBOOK_CHUNK_OVERLAP` 預設為分塊大小的 15%。兩者都是以 **token 為單位**(而非字元),最小分塊大小為 100,且需要重新啟動應用程式才會生效。

## 嵌入(`utils/embedding.py`)

- `generate_embedding(text)` —— 統一入口:短文字(≤ 分塊大小)直接嵌入;長文字則先分塊、逐塊嵌入,再透過**平均池化**合併結果(先各自正規化 → 取平均 → 再正規化一次,使用 numpy)。
- `generate_embeddings(texts)` —— 供 `embed_source_command` 使用的批次路徑:每批 50 筆,並具備逐批重試機制,以維持在供應商的酬載限制之下。
- 空白或純空格輸入會丟出 `ValueError`——依設計,背景任務會將此視為永久性(不重試)的失敗。
- 嵌入模型來自 `model_manager`(供應商設定如何解析,請參考 [credentials.zh.md](credentials.zh.md))。

**誰會觸發嵌入**(另請參考 `open_notebook/AGENTS.md` 中的領域規則):

| 內容 | 觸發方式 |
|---|---|
| Note | `Note.save()` 會自動送出 `embed_note` |
| Insight | `create_insight_command` 會送出 `embed_insight` |
| Source | 需明確呼叫 `source.vectorize()` → `embed_source`(儲存時不會自動觸發) |
| 全部 | `rebuild_embeddings_command` 會分派出個別的工作 |

所有嵌入都是透過 surreal-commands 背景 worker 以「發出即忘」(fire-and-forget)方式執行的——如果 worker 沒有在跑,就什麼都不會被嵌入。

## 上下文建構(`utils/context_builder.py`)

同時支援兩種上下文使用情境的單一實作:

- `build_notebook_context()` 是 `POST /api/chat/context`(聊天面板 + Podcast 生成)背後的實作:它會依照納入設定,組合 source/note 的上下文,其中的狀態字串是以文字比對的方式判斷("not in" 代表略過、"insights" 代表使用簡短上下文、"full content" 代表使用完整上下文)。若沒有提供設定,則每個 source 與 note 都會以其簡短上下文被納入。個別項目若失敗會被記錄下來並跳過。
- `build_source_context()` 是 source-chat 圖(graph)背後的實作:針對單一 source,提供其簡短上下文加上其 insights,並依 token 預算截斷(超出時優先捨棄最早取得的 insight)。
- 每次呼叫都會重新抓取資料——沒有快取層。
- Token 計算透過 tiktoken 使用 `o200k_base`,是一個估計值(與實際模型相比誤差約 ±5-10%);若 tiktoken 無法使用,`token_count()` 會退回較粗略的估計方式。

## 加密(`utils/encryption.py`) {#encryption}

針對資料庫中儲存的敏感值(API 金鑰)所做的欄位層級加密,使用 Fernet(AES-128-CBC + HMAC-SHA256)。

- 金鑰來源:`OPEN_NOTEBOOK_ENCRYPTION_KEY_FILE`(Docker secrets)→ `OPEN_NOTEBOOK_ENCRYPTION_KEY`。**沒有預設值**——在金鑰設定之前,憑證儲存功能無法使用。
- 任何字串都可以當作金鑰:會在第一次使用時透過 SHA-256 惰性推導出 Fernet 金鑰。
- 解密具有優雅的後備機制:遇到 `InvalidToken`(舊版未加密資料)時會回傳原始值,讓加密功能導入前建立的資料庫仍能正常運作。
- 金鑰輪替**尚未實作**——更換金鑰會讓先前加密過的值變成無法解密的孤兒資料。

## 文字工具(`utils/text_utils.py`)

`clean_thinking_content()` 會移除模型輸出中的 `<think>…</think>` 區塊(用於具備擴充思考能力的模型);每個處理 LLM 回應的 graph 都會用到它。它能處理格式不正確的輸出(缺少開頭標籤),並在內容超過 100KB 時略過擷取以維持效能。
