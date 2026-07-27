> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](windows-native.md)

# Open Notebook Windows 安裝指南(原生,不使用 Docker)

本指南說明如何在 Windows 上**原生安裝並執行** [Open Notebook](https://github.com/lfnovo/open-notebook),**不需要 Docker 或 WSL**。

## 適用對象

- **Windows ARM64 使用者** - Docker Desktop 與 WSL2 在 ARM64 上有其限制
- **沒有 Hyper-V 的使用者** - 部分 Windows 版本不支援 Docker
- **偏好原生安裝的使用者** - 架構更簡單,除錯更容易

## 本指南涵蓋內容

- 原生 Windows 安裝步驟
- 針對 Windows 的關鍵設定修正
- 常見問題的疑難排解
- 升級與維護腳本

## 前置需求

| 軟體         | 安裝方式                          | 是否必要 |
| ------------ | --------------------------------- | -------- |
| Git          | `winget install Git.Git`         | 是      |
| Python 3.12+ | 透過 uv(自動安裝)                | 是      |
| Node.js 18+  | `winget install OpenJS.NodeJS`   | 是      |
| uv           | `pip install uv`                 | 是      |
| SurrealDB    | `scoop install surrealdb`        | 是      |

## 快速上手

1. **Clone 並設定:**

   ```bash
   cd %USERPROFILE%\Projects  # 或你偏好的位置
   git clone https://github.com/lfnovo/open-notebook.git
   cd open-notebook
   uv sync
   cd frontend && npm install && cd ..
   ```

2. **設定 `.env`:**

   - 將 `.env.example` 複製為 `.env`

   - 加入你的 API 金鑰

   - **重要:**將 `SURREAL_URL` 從 `localhost` 改為 `127.0.0.1`:

     ```env
     SURREAL_URL="ws://127.0.0.1:8000/rpc"
     ```

3. **啟動四個服務**,各自在獨立的終端機視窗中,並在 `open-notebook` 資料夾下執行。

   > Open Notebook 沒有內建啟動腳本 —— 請依照下方步驟手動啟動各服務(或自行包成 `.bat` 檔,參見[選用:一鍵啟動腳本](#選用一鍵啟動腳本))。

   ```batch
   REM 選用:讓 Open Notebook 使用獨立的資料資料夾(參見下方問題 4)。
   REM 在每個終端機執行前設定此變數,或略過此步驟以使用 ./data。
   set DATA_FOLDER=%USERPROFILE%\Projects\open-notebook-data

   REM 終端機 1 — SurrealDB
   surreal start --user root --pass root --bind 127.0.0.1:8000 rocksdb:%DATA_FOLDER%\surrealdb

   REM 終端機 2 — API
   uv run --env-file .env run_api.py

   REM 終端機 3 — Worker(用模組形式執行可避免 Windows 的「canonicalize」錯誤,參見問題 3)
   set PYTHONPATH=%CD%
   uv run --env-file .env python -m surreal_commands.cli.worker --import-modules commands

   REM 終端機 4 — 前端
   cd frontend && npm run dev
   ```

4. **開啟應用程式:** http://127.0.0.1:3000

## 建議的目錄結構

```
YourProjectsFolder\
├── open-notebook\           # 原始碼(git clone)
│   ├── .venv\               # Python 虛擬環境(由 uv 建立)
│   ├── frontend\            # Next.js 前端
│   ├── commands\            # Worker 指令模組
│   └── .env                 # 你的設定
├── open-notebook-data\      # 資料儲存位置(與程式碼分開!)
│   ├── surrealdb\           # 資料庫檔案
│   ├── uploads\             # 已上傳的文件
│   └── sqlite-db\           # LangGraph checkpoint
└── start-open-notebook.bat  # 你自行建立的選用啟動腳本(見下方)
```

**為什麼要分開資料資料夾?** 避免在更新或重新安裝程式碼時不小心遺失資料。

## 選用:一鍵啟動腳本

Open Notebook 沒有內建啟動腳本,但你可以將以下內容儲存為
`start-open-notebook.bat`(放在任何位置皆可),就能用雙擊一次啟動全部四個服務。
請自行調整 `ROOT` 與 `DATA_ROOT` 以符合你的設定。

```batch
@echo off
REM --- 請調整以下兩個路徑 ---
set ROOT=%USERPROFILE%\Projects\open-notebook
set DATA_ROOT=%USERPROFILE%\Projects\open-notebook-data

set DATA_FOLDER=%DATA_ROOT%
set PYTHONPATH=%ROOT%
cd /d %ROOT%

start "SurrealDB" surreal start --user root --pass root --bind 127.0.0.1:8000 rocksdb:%DATA_ROOT%\surrealdb
start "API" cmd /k "uv run --env-file .env run_api.py"
start "Worker" cmd /k "uv run --env-file .env python -m surreal_commands.cli.worker --import-modules commands"
start "Frontend" cmd /k "cd /d %ROOT%\frontend && npm run dev"
```

接著開啟 http://127.0.0.1:3000。

## Windows 關鍵修正

### 問題 1:Python 版本錯誤

**症狀:**

```
ModuleNotFoundError: No module named 'langgraph.checkpoint.sqlite'
```

錯誤追蹤顯示的是系統 Python(例如 `C:\Python314\`),而不是虛擬環境。

**原因:** Windows 上可能安裝了多個 Python 版本。虛擬環境的 `activate.bat` 不一定能正確覆蓋。

**解法:** 用 `uv run` 取代直接呼叫 python:

```batch
REM 錯誤:
.venv\Scripts\python.exe run_api.py

REM 正確:
uv run python run_api.py
```

### 問題 2:資料庫健康檢查逾時

**症狀:**

```
WARNING: Database health check timed out after 2 seconds
```

即使 SurrealDB 正在執行,前端仍顯示「Database is offline」。

**原因:** `.env` 使用 `localhost`,但 SurrealDB 綁定在 `127.0.0.1`。

**解法:** 在 `.env` 中修改為:

```env
# 錯誤:
SURREAL_URL="ws://localhost:8000/rpc"

# 正確:
SURREAL_URL="ws://127.0.0.1:8000/rpc"
```

### 問題 3:Worker 出現「Failed to canonicalize script path」

**症狀:**

```
Failed to canonicalize script path
```

**原因:** `surreal-commands-worker.exe` 找不到 Python 的 `commands` 模組。

**解法:** 改用 Python 模組呼叫方式,並設定 PYTHONPATH:

```batch
set PYTHONPATH=%ROOT%
uv run --env-file .env python -m surreal_commands.cli.worker --import-modules commands
```

### 問題 4:DATA_FOLDER 路徑解析錯誤

**症狀:**

```
warning: Failed to parse environment file .env at position X
```

**原因:** `uv` 無法解析 `.env` 中含有反斜線的 Windows 路徑。

**解法:** 讓 `.env` 中的 `DATA_FOLDER` 保持**註解狀態**,改用批次檔設定:

```batch
set DATA_FOLDER=C:\path\to\open-notebook-data
```

## 設定檔

### 修改 `open_notebook/config.py`

預設的 `config.py` 使用寫死的資料路徑。修改它以從環境變數讀取:

```python
import os

# 根資料資料夾——可透過 DATA_FOLDER 環境變數覆寫
DATA_FOLDER = os.environ.get("DATA_FOLDER", "./data")

# 檔案其餘部分皆使用 DATA_FOLDER...
```

### 必要的 `.env` 設定

```env
# 資料庫——務必使用 127.0.0.1!
SURREAL_URL="ws://127.0.0.1:8000/rpc"
SURREAL_USER="root"
SURREAL_PASSWORD="root"
SURREAL_NAMESPACE="open_notebook"
SURREAL_DATABASE="open_notebook"

# API 金鑰(取消註解並填入)
OPENAI_API_KEY=your-key-here
ANTHROPIC_API_KEY=your-key-here
GOOGLE_API_KEY=your-key-here
```

## 可用的 AI 模型

啟動後,可以在 Settings 中新增模型。常見模型名稱:

| 供應商    | 模型                                                          |
| --------- | ------------------------------------------------------------ |
| OpenAI    | `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo`, `text-embedding-3-small` |
| Anthropic | `claude-sonnet-4-20250514`, `claude-3-5-sonnet-20241022`, `claude-3-5-haiku-20241022` |
| Google    | `gemini-3.5-flash`, `gemini-2.5-flash`, `gemini-2.5-pro`     |
| DeepSeek  | `deepseek-chat`, `deepseek-reasoner`                         |

## 升級

新版本釋出時:

```batch
cd open-notebook
git pull
uv sync
cd frontend && npm install && cd ..
```

接著重新啟動所有服務。你的 `.env` 與資料都會保留。

## 服務與連接埠

| 服務      | 連接埠 | URL                        |
| --------- | ---- | --------------------------- |
| SurrealDB | 8000 | ws://127.0.0.1:8000        |
| API       | 5055 | http://127.0.0.1:5055/docs |
| Frontend  | 3000 | http://127.0.0.1:3000      |

## 疑難排解

### 服務無法啟動

- 檢查連接埠是否已被佔用:`netstat -ano | findstr :8000`
- 終止既有的程序:`taskkill /F /PID <pid>`

### 前端無法連線到 API

- 確認 API 正在執行:http://127.0.0.1:5055/docs
- 檢查 `.env` 中是否有 `API_URL=http://localhost:5055`

### Worker 沒有在處理指令

- 檢查 Worker 視窗是否有錯誤訊息
- 確認啟動腳本中已設定 PYTHONPATH

## 貢獻

發現其他 Windows 特有的問題?歡迎分享你的解法!

---

*已在 Windows 11 ARM64、Open Notebook v1.6.0 上測試*
*建立日期:2026 年 1 月*
