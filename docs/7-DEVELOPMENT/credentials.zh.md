> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](credentials.md)

# 憑證系統

Open Notebook 如何儲存、加密與佈建 AI 供應商憑證——從設定畫面的 UI 一路到 Esperanto 模型的實例化。這是這個子系統唯一的參考文件;前端與後端兩個部分刻意寫在同一份文件裡。

## 概觀

使用者可以透過 UI 設定供應商憑證,不必使用環境變數。金鑰以個別的 `Credential` 紀錄儲存在 SurrealDB 中,以 Fernet 加密,並在模型佈建時以「資料庫優先、環境變數為後備」的策略解析。

```
Settings UI ──► /credentials API ──► Credential record (encrypted, SurrealDB)
                                          │
                Model record ──credential─┘        (preferred: direct link)
                     │
        ModelManager.get_model()
                     │
        credential.to_esperanto_config()  ──►  Esperanto AIFactory
                     │
        (no linked credential?)
                     └──► key_provider.provision_provider_keys()  ──►  env vars ──► Esperanto
```

## `Credential` 領域模型(`open_notebook/domain/credential.py`)

- 每筆紀錄對應一組憑證(例如「My OpenAI Key」、「Work Anthropic」)——支援同一個供應商設定多組憑證。
- 欄位:`name`、`provider`、`modalities`、`api_key`(Pydantic `SecretStr`,在日誌中會被遮蔽),以及供應商特定設定(`base_url`、`endpoint`、`api_version`、依模式而異的端點、`project`、`location`、`credentials_path`)。
- `api_key` 在儲存前以 `encrypt_value()` 加密,讀取時解密(`get()` / `get_all()` 已覆寫)。加密需要設定 `OPEN_NOTEBOOK_ENCRYPTION_KEY`(加密工具本身請參考 [content-processing.zh.md](content-processing.zh.md#encryption))。
- `to_esperanto_config()` 會建構傳給 Esperanto `AIFactory.create_*` 的設定字典。
- `provider_config.py` 之所以還存在,只是為了遷移舊版的 `ProviderConfig` 紀錄。

## 佈建:兩種路徑

1. **憑證關聯的模型(建議做法)。** `Model` 紀錄有一個 `credential` 欄位指向某個 Credential。`ModelManager.get_model()` 會呼叫 `credential.to_esperanto_config()` 並直接傳入該設定——不會改動環境變數,同一供應商的多組憑證也能自然運作。
2. **環境變數後備(`open_notebook/ai/key_provider.py`)。** 當模型沒有關聯任何憑證時,`provision_provider_keys(provider)` 會把資料庫中儲存的金鑰複製進 `os.environ`,讓 Esperanto 能讀取到;若資料庫中沒有設定,既有的環境變數不會被動到。`key_provider.py` 中的 `PROVIDER_CONFIG` 對照表定義了簡單供應商的「環境變數 ↔ 設定欄位」對應;多欄位的供應商(Vertex、Azure、OpenAI-compatible)則由專屬的 `_provision_vertex()` / `_provision_azure()` / `_provision_openai_compatible()` 函式處理。

## API 介面(`api/routers/credentials.py`)

CRUD 加上生命週期操作:`POST /credentials/{id}/test`(連線檢查)、`/discover`(列出可用模型)、`/register-models`(依探索結果建立 Model 紀錄),以及兩個遷移端點(`/migrate-from-env`、`/migrate-from-provider-config`)。`/docs` 的 Swagger 文件記載了各自的資料形狀。

**支援的供應商(17 個)**統一定義在供應商註冊表(`open_notebook/ai/provider_registry.py` 的 `PROVIDERS`)中——環境變數、modalities、測試用模型、探索用 URL 與文件連結都放在那裡,而 `connection_tester.TEST_MODELS`、`credentials_service.PROVIDER_ENV_CONFIG` / `PROVIDER_MODALITIES` 以及 `model_discovery.OPENAI_COMPAT_PROVIDERS` 都是由它衍生而來。`GET /api/providers` 會把這份註冊表公開給客戶端——前端在執行期透過 `useProviders()`(`frontend/src/lib/hooks/use-providers.ts`)抓取,並依回傳順序(即註冊表宣告的順序)顯示供應商。目前仍有一處需要手動同步,並由 `tests/test_credential_provider_validation.py` 強制檢查:`api/models.py` 中的 `SupportedProvider` Literal(型別無法在執行期推導):

- 單純 API 金鑰型:openai、anthropic、google、groq、mistral、deepseek、xai、openrouter、voyage、elevenlabs、deepgram、dashscope、minimax
- 以 URL 為基礎:ollama
- 多欄位:azure、vertex、openai_compatible

**安全性特性**:

- 任何端點都不會回傳 API 金鑰的值——只回傳中繼資料(`has_api_key`、數量統計)。
- 每個 URL 欄位都會經過 `validate_url()`(SSRF 防護);私有 IP/localhost 依設計是允許的,以支援自架服務(Ollama、LM Studio)。主機名稱解析會在 `asyncio.to_thread` 中執行,避免阻塞事件迴圈。
- Vertex 憑證的連線測試會把「檔案遺失/不是 JSON/格式錯誤」等錯誤全部收斂成同一則通用訊息,避免測試功能被當成檔案系統探測工具濫用。

## 連線測試(`open_notebook/ai/connection_tester.py`)

`test_provider_connection()` 會用每個供應商最便宜的模型(`TEST_MODELS` 對照表)發出一次最小規模的 API 呼叫。以 URL 為基礎的供應商則改為對伺服器做 ping(Ollama 用 `/api/tags`,OpenAI-compatible 用 `/models`)。錯誤訊息會針對 UI 做正規化處理:401 → 「Invalid API key」、rate-limit → 視為成功(「connection works」)、model-not-found → 視為成功(「key valid」)。

## 前端部分

- `src/lib/api/credentials.ts` —— 對應上述端點的型別化客戶端。`Credential` 介面永遠不會帶有金鑰值,只有 `has_api_key`。
- `src/lib/hooks/use-credentials.ts` —— TanStack Query hooks(`useCredentials`、`useCreateCredential`、`useTestCredential` 等),搭配 toast 回饋。Mutation 會使 `CREDENTIAL_QUERY_KEYS.all` 以及供應商/模型相關的快取鍵失效;測試結果保存在本地 state,不放進 query cache。

## 遷移路徑

兩個遷移端點都是冪等的摘要式操作(`migrated` / `skipped` / `errors`):

- **從環境變數遷移**:為已設定環境變數的供應商建立 Credential 紀錄。
- **從舊版 ProviderConfig 遷移**:把舊的單例紀錄轉換成個別的 Credential。
