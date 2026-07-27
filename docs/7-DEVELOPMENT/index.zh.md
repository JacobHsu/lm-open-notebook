> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](index.md)

# 開發指南

歡迎來到 Open Notebook 開發文件!無論你是要貢獻程式碼、了解我們的架構,還是要維護這個專案,都能在這裡找到指引。

## 🎯 選擇你的路線

### 👨‍💻 我想貢獻程式碼

先看 **[貢獻指南](contributing.zh.md)** 了解整體工作流程,接著參考:
- **[快速上手](quick-start.zh.md)** - 5 分鐘完成 clone、安裝與驗證
- **[開發環境設定](development-setup.zh.md)** - 完整的本機環境設定指南
- **[程式碼標準](code-standards.zh.md)** - 如何寫出符合我們風格的程式碼
- **[測試](testing.zh.md)** - 如何撰寫與執行測試

**第一次來?** 請參考[貢獻指南](contributing.zh.md),了解 Discussions → Issues → PRs 的工作流程。

### 🔒 我想了解安全實踐

**[安全指南](security.zh.md)** 涵蓋:
- 資料庫查詢安全(防止 SurrealQL 注入)
- 樣板渲染安全(防止 SSTI)
- 檔案處理安全(防止路徑遍歷與 LFI)
- 機密資訊管理與 CORS 設定
- 程式碼審查安全檢查清單

---

### 🏗️ 我想了解架構

**[架構概觀](architecture.zh.md)** 涵蓋:
- 三層系統設計
- 技術堆疊與選型理由
- 關鍵元件與工作流程
- 我們採用的設計模式

想深入了解特定子系統:
- **[憑證](credentials.zh.md)** - 供應商憑證儲存、加密、佈建
- **[內容處理](content-processing.zh.md)** - 分塊、嵌入、上下文建構、加密
- **[Podcast](podcasts.zh.md)** - 設定檔系統、模型註冊表、工作生命週期
- **[Prompts](prompts.zh.md)** - Prompt 工程模式
- **[前端](frontend.zh.md)** - Next.js 分層與資料流

給編碼代理人(以及趕時間的人類)的規範性規則,放在專案根目錄、`open_notebook/` 與 `frontend/` 底下的 `AGENTS.md` 檔案中。

### 🧭 為什麼會是這樣設計?

- **[VISION.md](../../VISION.md)** - 產品定位與目前方向
- **[決策紀錄](decisions/README.md)** - ADR 與 PDR:結構性選擇背後長期有效的「原因」

---

### 👨‍🔧 我是維護者

**[維護者指南](maintainer-guide.zh.md)** 涵蓋:
- Issue 分流與管理
- Pull Request 審查流程
- 溝通範本
- 最佳實踐

---

## 📚 快速連結

| 文件 | 適用對象 | 用途 |
|---|---|---|
| [快速上手](quick-start.zh.md) | 新開發者 | 5 分鐘完成 clone、安裝與驗證設定 |
| [開發環境設定](development-setup.zh.md) | 本機開發 | 完整的環境設定指南 |
| [貢獻指南](contributing.zh.md) | 社群與程式碼貢獻者 | 工作流程:Discussion → Issue → PR |
| [程式碼標準](code-standards.zh.md) | 撰寫程式碼 | Python、FastAPI、資料庫的風格指南 |
| [測試](testing.zh.md) | 測試程式碼 | 如何撰寫與執行測試 |
| [架構](architecture.zh.md) | 了解系統 | 系統設計、技術堆疊、工作流程 |
| [憑證](credentials.zh.md) | 了解系統 | 供應商憑證子系統 |
| [內容處理](content-processing.zh.md) | 了解系統 | 分塊、嵌入、上下文建構 |
| [Podcast](podcasts.zh.md) | 了解系統 | Podcast 設定檔與工作生命週期 |
| [Prompts](prompts.zh.md) | 了解系統 | Prompt 工程模式 |
| [前端](frontend.zh.md) | 了解系統 | Next.js 架構與資料流 |
| [設計原則](design-principles.zh.md) | 所有開發者 | 工程實踐與反模式 |
| [VISION.md](../../VISION.md) | 所有開發者 | 產品定位與目前方向 |
| [決策紀錄](decisions/README.md) | 所有開發者 | ADR/PDR——事情為什麼會是這樣 |
| [變更手冊](change-playbooks.zh.md) | 貢獻者與代理人 | 常見變更的逐步操作指南 |
| [API 參考](api-reference.zh.md) | 建構整合 | 完整的 REST API 文件 |
| [安全](security.zh.md) | 所有開發者 | 安全實踐與漏洞防範 |
| [維護者指南](maintainer-guide.zh.md) | 維護者 | 管理 Issue、PR、標籤 |

---

## 🚀 目前的開發重點

我們目前正在尋找以下方面的協助:

1. **前端強化** - 用即時更新改善 Next.js/React UI
2. **效能** - 非同步處理與快取最佳化
3. **測試** - 擴大各元件的測試涵蓋率
4. **文件** - API 範例與開發者指南
5. **整合** - 新的內容來源與 AI 供應商

可留意標記為 `good first issue` 或 `help wanted` 的 GitHub Issues。

---

## 💬 尋求協助

- **Discord**:[加入我們的伺服器](https://discord.gg/37XJPXfz2w)進行即時討論
- **GitHub Discussions**:用於提問、發想、功能建議、產品方向、設計與架構討論
- **GitHub Issues**:用於可重現的錯誤回報與已核准的工作項目

別害羞!我們很樂意協助新貢獻者順利上手。

---

## 📖 其他資源

### 外部文件
- [FastAPI 文件](https://fastapi.tiangolo.com/)
- [SurrealDB 文件](https://surrealdb.com/docs)
- [LangChain 文件](https://python.langchain.com/)
- [Next.js 文件](https://nextjs.org/docs)

### 我們的函式庫
- [Esperanto](https://github.com/lfnovo/esperanto) - 多供應商 AI 抽象層
- [Content Core](https://github.com/lfnovo/content-core) - 內容處理
- [Podcast Creator](https://github.com/lfnovo/podcast-creator) - Podcast 生成

---

準備好開始了嗎?前往 **[快速上手](quick-start.zh.md)**!🎉
