> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](podcasts.md)

# Podcast 子系統

Podcast 生成如何被建模與執行:雙層設定檔系統、model-registry 參照,以及刻意不自動重試的政策。

## 雙層設定檔系統(`open_notebook/podcasts/models.py`)

- **SpeakerProfile** —— 語音設定:一個 `voice_model`(供 TTS 使用的 `record<model>` 參照)加上 1 到 4 位講者(名稱、voice_id、背景故事、個性)。個別講者可以覆寫設定檔的 `voice_model`。
- **EpisodeProfile** —— 生成設定:`outline_llm` / `transcript_llm`(`record<model>` 參照)、`language`(BCP 47,例如 `pt-BR`)、片段數量(3–20)、briefing 樣板。它會以名稱參照某個 SpeakerProfile。
- **PodcastEpisode** —— 一集已生成的節目。連結內容、設定檔與非同步工作(`command` 欄位 → surreal-commands 的 RecordID)。

## Model registry 參照,而非字串

設定檔欄位參照的是 `Model` 紀錄,而不是原始的供應商/模型字串。在生成階段,`_resolve_model_config(model_id)` 會載入該 Model、解析其關聯的憑證(若無則退回 `provision_provider_keys()`),並回傳 `(provider, model_name, config)` 給 podcast-creator 使用。

早於 registry 出現的舊版字串欄位(`tts_provider`、`outline_provider` 等)已由 SQL migration 22(#1107)移除。該 migration 會盡力把尚未解析的設定檔對應到既有的 `model` 紀錄(依 provider + name + type);沒有對應紀錄的設定檔則維持未解析狀態——UI 已經會標示這些設定檔需要重新選擇模型,使用者只需重新選一次即可。舊版的啟動期資料 migration(`open_notebook/podcasts/migration.py`)已經移除。

## 設定檔快照

`PodcastEpisode` 會把 `episode_profile` 與 `speaker_profile` 儲存成**字典(快照)**,而不是參照。編輯設定檔不會回溯影響過去的集數——這是刻意的設計。推論結果:刪除設定檔不會連帶刪除已生成的集數。

## 工作生命週期與重試政策

生成流程是在 surreal-commands 背景 worker 上執行的 `generate_podcast_command` 工作:

- 該 command 會在呼叫 podcast-creator 之前,先解析**所有**設定檔的模型設定與憑證,並驗證 `outline_llm`、`transcript_llm` 與 `voice_model` 皆已設定。
- **`max_attempts: 1`——沒有自動重試。** 因為集數紀錄是在執行過程中建立的,若在生成過程中重試會產生重複的集數紀錄。失敗的集數會被標記為 `failed` 並附上錯誤訊息;重試需要使用者明確透過 `POST /podcasts/episodes/{id}/retry` 觸發。
- 狀態追蹤:`get_job_status()` / `get_job_detail()` 會查詢 surreal-commands,失敗時回傳 `"unknown"` 而不是拋出例外。列表端點使用批次化的 `get_job_details_for_commands()`,讓 N 集節目只需要一次狀態查詢,而不是 N 次。
- TTS 失敗時會回退為無聲音訊,而不是讓整集失敗。
