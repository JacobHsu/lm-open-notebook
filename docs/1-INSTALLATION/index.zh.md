> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](index.md)

# 安裝指南

依照你的環境與使用情境,選擇適合的安裝路線。

## 快速決策:選哪條路線?

### 🚀 我想要最簡單的設定(大多數人推薦)
**→ [Docker Compose](docker-compose.zh.md)** - 多容器設定,可用於正式環境
- ✅ 所有功能都能運作
- ✅ 服務之間分工清楚
- ✅ 容易擴充
- ✅ 支援 Mac、Windows、Linux
- ⏱️ 5 分鐘上手

---

### 🏠 我想要全部塞在一個容器裡(已淘汰)
**→ [單一容器](single-container.zh.md)** - 已淘汰,將於 v2 移除
- ⚠️ **已淘汰** —— 請改用 Docker Compose
- v2 發佈前仍會持續支援

---

### 👨‍💻 我想要開發/貢獻程式碼(僅限開發者)
**→ [原始碼安裝](from-source.zh.md)** - Clone 儲存庫,在本機設定
- ✅ 完全掌控程式碼
- ✅ 容易除錯
- ✅ 可以修改與測試
- ⚠️ 需要 Python 3.11+、Node.js
- ⏱️ 10 分鐘上手

---

### 🪟 我用 Windows,且無法使用 Docker/WSL
**→ [Windows 原生安裝](windows-native.zh.md)** - 原生執行,不需要 Docker 或 WSL
- ✅ 支援 Windows ARM64
- ✅ 適用於沒有 Hyper-V/Docker Desktop 的系統
- ⚠️ 需要 Python 3.12+、Node.js、SurrealDB、uv
- ⏱️ 15 分鐘上手

---


## 系統需求

### 最低需求
- **記憶體**:4GB
- **儲存空間**:應用程式需要 2GB,另外還要留空間給文件
- **CPU**:任何現代處理器
- **網路**:需要網際網路(離線設定則為選用)

### 建議規格
- **記憶體**:8GB 以上
- **儲存空間**:文件與模型建議 10GB 以上
- **CPU**:多核心處理器
- **GPU**:選用(可加速本機 AI 模型)

---

## AI 供應商選項

### 雲端型(依用量付費)
- **OpenAI** - GPT-4、GPT-4o,快速且強大
- **Anthropic (Claude)** - Claude 3.5 Sonnet,推理能力優秀
- **Google Gemini** - 多模態,成本效益高
- **Groq** - 超高速推論
- **其他**:Mistral、DeepSeek、xAI、OpenRouter

**費用**:通常每 1K token 約 $0.01-$0.10
**速度**:快(不到一秒)
**隱私**:你的資料會傳送到雲端

### 本機型(免費、私有)
- **Ollama** - 在本機執行開源模型
- **LM Studio** - 本機模型的桌面應用程式
- **Hugging Face 模型** - 下載後自行執行

**費用**:$0(只需要電費)
**速度**:視你的硬體而定(慢到中等)
**隱私**:100% 離線

---

## 選擇一條路線

**已經知道要選哪一種了?** 挑選你的安裝路線:

- [Docker Compose](docker-compose.zh.md) - **大多數使用者**
- [單一容器](single-container.zh.md) - **已淘汰**
- [原始碼安裝](from-source.zh.md) - **開發者**

> **重視隱私?** 任何安裝方式都能搭配 Ollama 實現 100% 本機 AI。參見[本機快速上手](../0-START-HERE/quick-start-local.zh.md)。

---

## 安裝前檢查清單

開始安裝之前,你需要準備:

- [ ] **Docker**(Docker 路線需要)或 **Node.js 18+**(原始碼安裝需要)
- [ ] **AI 供應商 API 金鑰**(OpenAI、Anthropic 等)或是願意使用免費的本機模型
- [ ] **至少 4GB 可用記憶體**
- [ ] **穩定的網際網路連線**(或使用 Ollama 做離線設定)

---

## 詳細安裝步驟

### Docker 使用者
1. 安裝 [Docker Desktop](https://docker.com/products/docker-desktop)
2. 依照 [Docker Compose](docker-compose.zh.md) 進行安裝
3. 依照逐步指南操作
4. 於 `http://localhost:8502` 存取

### 原始碼安裝(開發者)
1. 準備好 Python 3.11+、Node.js 18+、Git
2. 依照 [原始碼安裝](from-source.zh.md) 進行
3. 執行 `make start-all`
4. 於 `http://localhost:8502`(前端)或 `http://localhost:5055`(API)存取

---

## 安裝完成之後

啟動並執行之後:

1. **設定模型** - 在 Settings 中選擇你的 AI 供應商
2. **建立第一個 Notebook** - 開始整理研究內容
3. **新增來源** - PDF、網頁連結、文件
4. **探索功能** - 聊天、搜尋、轉換
5. **閱讀完整指南** - [使用者指南](../3-USER-GUIDE/index.md)

---

## 安裝過程疑難排解

**遇到問題?** 查看你所選安裝指南裡的疑難排解章節,或參考[快速修復](../6-TROUBLESHOOTING/quick-fixes.md)。

---

## 需要協助?

- **Discord**:[加入社群](https://discord.gg/37XJPXfz2w)
- **GitHub Issues**:[回報問題](https://github.com/lfnovo/open-notebook/issues)
- **文件**:參見[完整文件](../index.md)

---

## 正式環境部署

要安裝到正式環境使用?可參考以下額外資源:

- [安全性強化](../5-CONFIGURATION/security.md)
- [反向代理設定](../5-CONFIGURATION/reverse-proxy.md)
- [效能調校](../5-CONFIGURATION/advanced.md)

---

**準備好安裝了嗎?** 從上面挑選一條路線!⬆️
