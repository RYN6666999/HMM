
Project Golem — System Flow v1.1

最後更新：2026-04-15 | 對應：constitution.md v1.1
倉庫：https://github.com/RYN6666999/HMM

1. 完整系統流程圖

flowchart TB
    NLM["🔬 NotebookLM知識研究"] --> SPEC["📋 Spec Kit規格撰寫"]
    SPEC --> STITCH["🎨 Stitch SDKUI 設計"]
    STITCH --> CODE["⌨️ Claude Code / OpenCode程式碼實作"]
    CODE --> MCP["🔌 MCP Server Core15 支 Tool"]
    MCP --> HERMES["🤖 Hermes Agent任務執行 + 自學習"]
    HERMES --> USER["👤 使用者Telegram / Discord / Web"]
    CODE --> GH["📦 GitHub CI/CD測試 + sync_check"]

    MCP --- MEMORY["🧠 Pyramid Memory3 namespaces"]
    MCP --- BITL["🌐 Browser-in-the-Loop三層 Fallback"]
    MCP --- GOOGLE["☁️ Google ToolsStitch · NotebookLM · AI Studio"]
    MCP --- CLI["🖥️ CLI-Anything"]
    MCP --- FS["📁 檔案系統file_read · file_write · file_list"]

    %% 閉環 1：學習迴圈
    HERMES -->|"經驗存入namespace: task_experience"| MEMORY
    MEMORY -->|"經驗召回memory_search + token_count"| MCP

    %% 閉環 2：設計迭代
    CODE -->|"UI 不滿意 → screen.edit()"| STITCH

    %% 閉環 3：Spec 演化
    CODE -->|"發現 spec 缺漏 → spec_update"| SPEC
    MCP -->|"發現邊界缺陷 → spec_update"| SPEC

    %% 閉環 4：知識迴圈
    HERMES -->|"未知領域 → free_notebooklm"| NLM
    NLM -->|"研究結果存入namespace: research_knowledge"| MEMORY

    %% 閉環 2 ↔ 3 同步檢查
    GH -->|"design.md 變更 → sync_check"| SPEC
    GH -->|"spec-*.md 變更 → sync_check"| STITCH

    %% 三層 Fallback
    BITL -->|"失敗"| API["⚡ Gemini API免費額度"]
    API -->|"失敗"| OLLAMA["🏠 Ollama本地模型"]

    %% Context Budget
    MCP -->|"呼叫 LLM 前"| NS["🧮 NeuroShuntercontext budget 管理"]
    NS -->|"裁剪後的 context"| BITL

2. 四大閉環詳述

2.1 閉環 1 — 學習迴圈（Learning Loop）

觸發條件：Hermes 完成任何任務。

資料流向：Hermes 執行任務 → 結果 + metadata 呼叫 memory_store（namespace: task_experience）存入 Pyramid Memory → 未來相似任務時 memory_search（namespace: task_experience）召回經驗 → 經驗附帶 token_count 傳入 NeuroShunter → NeuroShunter 按 context budget 裁剪後注入 LLM prompt → 輸出品質提升。

壓縮機制：memory_compress 按 namespace 分別執行，不會將 task_experience 和 research_knowledge 混合壓縮。壓縮階層：即時 → 日 → 週 → 月 → 年，長期容量約 3 MB / 50 年。

頻率：每次任務自動觸發。

預期效益：約 40% 效率提升（經驗累積後減少重複錯誤與搜索時間）。

2.2 閉環 2 — 設計迭代（Design Iteration Loop）

觸發條件：UI 審查不通過。

資料流向：Stitch SDK 產出 design.md + HTML/截圖 → Claude Code/OpenCode 依據 design.md 生成 UI 程式碼 → Agent 或人工審查 → 不滿意則呼叫 screen.edit(prompt) 回到 Stitch → 匯出更新後的 design.md → 程式碼重新生成。

同步機制：每次 design.md 被 commit 至 GitHub，CI pipeline 自動呼叫 sync_check，比對 design.md 中的元件名稱與 spec-*.md 中的 API 端點。不一致時在 PR comment 中列出差異並標記 warning。

頻率：每個 UI 模組約 2-5 次迭代。

2.3 閉環 3 — Spec 演化（Spec Evolution Loop）

觸發條件：實作中發現 spec 缺漏、邊界案例未覆蓋、或 API 契約需變更。

資料流向：Claude Code/OpenCode 實作時發現問題 → 呼叫 spec_update（指定 spec 路徑、章節、新內容）→ 自動更新 spec 並 commit → 任務清單重新生成 → 程式碼重新實作。MCP Server 運行時發現缺陷也可觸發 spec_update。

同步機制：每次 spec-*.md 被 commit，CI 自動呼叫 sync_check 反向比對 design.md，確保雙向一致。

頻率：每個模組約 1-3 次更新。

2.4 閉環 4 — 知識迴圈（Knowledge Loop）

觸發條件：Hermes 遇到未知領域或需要深度研究。

資料流向：Hermes 呼叫 free_notebooklm（query + 參考來源）→ NotebookLM 進行研究 → 回傳 summary + key_findings → 研究結果呼叫 memory_store（namespace: research_knowledge）存入 Pyramid Memory → 未來相同領域查詢時 memory_search（namespace: research_knowledge）直接召回，不再重複研究。

頻率：低頻但高價值。

3. 三層 Fallback 機制（free_llm_query）

當 free_llm_query 被呼叫時，依序嘗試三個後端：

第一層 — Browser-in-the-Loop：Playwright 開啟瀏覽器 → 登入 Google → 送 prompt 到 Gemini → 擷取回應。若失敗（CAPTCHA、session 過期、DOM 變更），自動重試一次（含重新載入頁面）。延遲目標  task_memory > spec 摘要 > research_knowledge > design 片段。

第三步，從最低優先級開始，超出預算的來源被截斷或只保留摘要。

第四步，budget_usage 欄位回報各來源佔用的 token 數與被截斷的來源名稱，供除錯用。

5. MCP Tool 完整角色表（15 支）

| Tool | 使用時機 | 參與閉環 | 核心引擎 |
|------|---------|---------|----------|
| free_llm_query | 需要 LLM 回應時 | 1, 4 | Browser-in-the-Loop + 三層 Fallback |
| memory_store | 任務完成 / 研究完成 / 設計決策時 | 1, 4 | Pyramid Memory（含 namespace） |
| memory_search | 開始新任務 / 需要歷史經驗時 | 1, 4 | Pyramid Memory（含 namespace + token_count） |
| memory_compress | 定期維護 / 記憶過多時 | 1 | Pyramid Memory（按 namespace 分別壓縮） |
| parse_protocol | 每次呼叫 LLM 前 | 1, 2, 3, 4 | NeuroShunter（含 context budget） |
| free_notebooklm | 遇到未知領域需研究時 | 4 | CLI-Anything-Web |
| spec_read | 實作前讀取規格 / 檢查驗收標準 | 3 | 檔案系統 |
| spec_update | 實作中發現 spec 缺漏時 | 3 | 檔案系統 + Git |
| file_read | 讀取 log / config / 任意檔案 | — | 檔案系統（受 allowed_paths 限制） |
| file_write | 寫入 config / 生成檔案 | — | 檔案系統（受 allowed_paths 限制） |
| file_list | 列出目錄結構 | — | 檔案系統（受 allowed_paths 限制） |
| sync_check | design.md 或 spec 被 commit 時 | 2, 3 | 檔案系統（CI 自動觸發） |
| desktop_control | 需要操控桌面應用時 | — | Playwright |
| design_generate | 需要 UI 設計時 | 2 | Stitch SDK |
| code_generate | 需要程式碼生成時 | — | Gemini API / OpenCode |

6. 統一介面架構

主介面 — Hermes Dashboard（http://127.0.0.1:9119）
處理 90% 的日常操作：對話、指令發送、MCP Tool 呼叫、Cron 排程、Session 管理、Token 用量監控、Skill 啟停、API Key 管理。平板可透過 --host 0.0.0.0 遠端存取（建議搭配 VPN）。

輔助介面 — Golem Dashboard
處理 Pyramid Memory 的深度視覺化：記憶分區（namespace）瀏覽、壓縮狀態、日記輪轉、容量趨勢。從 Hermes Dashboard 加入連結跳轉。

開發介面 — Claude Code / OpenCode
處理程式碼實作。平板環境透過 GitHub Codespaces + Claude Code Remote Control 使用。

設計介面 — Stitch（https://stitch.withgoogle.com/）
處理 UI 設計，產出 design.md。平板瀏覽器可直接使用。

驗證介面 — Opal（Phase 0 限定）
處理流程概念驗證，不納入自動化。

7. 日常運作總覽

持續運行：閉環 1（學習迴圈）— 每次 Hermes 任務自動觸發。

按需觸發：閉環 2（設計迭代）— UI 模組開發期間；閉環 3（Spec 演化）— 實作發現缺漏時；閉環 4（知識迴圈）— 遇到未知領域時。

自動執行：sync_check — 每次 design.md 或 spec-*.md 被 commit 時由 CI 觸發。memory_compress — 可設為 Hermes Cron 定期執行（如每日凌晨）。

版本紀錄
- v1.0（2026-04-15）：初版建立
- v1.1（2026-04-15）：同步 constitution.md v1.1 修正——新增 namespace 標註於 Mermaid 圖、sync_check 觸發路徑、三層 Fallback 機制說明、Context Budget 管理章節、Tool 角色表擴充至 15 支、統一介面架構章節

