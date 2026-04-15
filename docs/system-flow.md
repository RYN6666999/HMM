docs/system-flow.md v1.2

最後更新：2026-04-15 | 對應：constitution.md v1.2
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
    MCP --- GOOGLE["☁️ Google ToolsStitch · NotebookLM · AI Studio"]
    MCP --- CLI["🖥️ CLI-Anything"]
    MCP --- FS["📁 檔案系統file_read · file_write · file_list"]

    %% weblm-driver 與三層 Fallback
    MCP --- BL["🔌 browser-loop.js三層 Fallback 調度器"]
    BL --> WEBLM["🌐 weblm-driver v0.1.5GeminiWebDriver四級恢復 · 134 tests"]
    BL -->|"weblm-driver 全部失敗"| API["⚡ Gemini API免費額度 15次/分"]
    BL -->|"API 也失敗"| OLLAMA["🏠 Ollama本地模型"]

    %% weblm-driver 內部四級恢復
    WEBLM -.->|"refresh → reopen → restart → rebuild"| WEBLM

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

    %% Context Budget
    MCP -->|"呼叫 LLM 前"| NS["🧮 NeuroShuntercontext budget 管理"]
    NS -->|"裁剪後的 context"| BL

2. 四大閉環詳述

2.1 閉環 1 — 學習迴圈（Learning Loop）

觸發條件：Hermes 完成任何任務。

資料流向：Hermes 執行任務 → 結果 + metadata 呼叫 memory_store（namespace: task_experience）存入 Pyramid Memory → 未來相似任務時 memory_search（namespace: task_experience）召回經驗 → 經驗附帶 token_count 傳入 NeuroShunter → NeuroShunter 按 context budget 裁剪後注入 LLM prompt → 輸出品質提升。

壓縮機制：memory_compress 按 namespace 分別執行，不會將 task_experience 和 research_knowledge 混合壓縮。壓縮階層：即時 → 日 → 週 → 月 → 年，長期容量約 3 MB / 50 年。

頻率：每次任務自動觸發。

預期效益：約 40% 效率提升。

2.2 閉環 2 — 設計迭代（Design Iteration Loop）

觸發條件：UI 審查不通過。

資料流向：Stitch SDK 產出 design.md + HTML/截圖 → Claude Code/OpenCode 依據 design.md 生成 UI 程式碼 → Agent 或人工審查 → 不滿意則呼叫 screen.edit(prompt) 回到 Stitch → 匯出更新後的 design.md → 程式碼重新生成。

同步機制：每次 design.md 被 commit 至 GitHub，CI 自動呼叫 sync_check 比對 spec 中的 API 端點與 UI 元件名稱，不一致時標記 warning。

頻率：每個 UI 模組約 2-5 次迭代。

2.3 閉環 3 — Spec 演化（Spec Evolution Loop）

觸發條件：實作中發現 spec 缺漏或 API 變更。

資料流向：Claude Code/OpenCode 實作時發現問題 → 呼叫 spec_update（指定 spec 路徑、章節、新內容）→ 自動更新 spec 並 commit → 任務清單重新生成 → 程式碼重新實作。MCP Server 運行時發現缺陷也可觸發 spec_update。

同步機制：每次 spec-*.md 被 commit，CI 自動呼叫 sync_check 反向比對 design.md。

頻率：每個模組約 1-3 次更新。

2.4 閉環 4 — 知識迴圈（Knowledge Loop）

觸發條件：Hermes 遇到未知領域或需要深度研究。

資料流向：Hermes 呼叫 free_notebooklm（query + 參考來源）→ NotebookLM 進行研究 → 回傳 summary + key_findings → 研究結果呼叫 memory_store（namespace: research_knowledge）存入 Pyramid Memory → 未來相同領域查詢時 memory_search（namespace: research_knowledge）直接召回。

頻率：低頻但高價值。

3. 三層 Fallback 機制（free_llm_query）

free_llm_query 被呼叫時，由 browser-loop.js 調度三個層級。與 v1.1 的最大差異是：第一層不再自己管 Playwright，而是委託 weblm-driver（super-engine）的 GeminiWebDriver。

3.1 第一層 — weblm-driver（GeminiWebDriver）

browser-loop.js 呼叫 GeminiWebDriver.generate({ prompt, timeoutMs }) → weblm-driver 內部處理完整的 Playwright 生命週期：啟動瀏覽器 → 偵測登入狀態 → 提交 prompt → 雙階段輪詢擷取輸出（first-token timeout + stability check + stop-button 偵測）→ 回傳 GenerateOutput。

若 generate() 拋出 recoverable DriverError，browser-loop.js 呼叫 driver.recover(error.code) 進入 weblm-driver 的四級恢復：

refresh-page    → 重新整理頁面，清除卡住的 generation 狀態
reopen-page     → 關閉再重開頁面，重新載入 SPA
restart-browser → 關閉整個瀏覽器 context，重新啟動
rebuild-session → 完全重建 session（最後手段）

恢復成功後重試一次 generate()。若恢復失敗（recovery.ok === false）或第二次 generate() 仍然拋錯，才進入第二層。

weblm-driver 提供的額外保障（HMM 不需要自己實作）：
- outputKind 分類（normal / provider-error / unknown）— browser-loop.js 檢查此欄位，非 normal 視為失敗進入 fallback。
- ConcurrentGenerationError — 防止同時多次呼叫。
- health() — 非拋出式健康檢查（5 個維度），browser-loop.js 可定期呼叫回報到 MCP Server 日誌。
- Structured logging — JSON-Lines 記錄到 stderr，15 種事件涵蓋完整生命週期。
- 23 個 CSS selector fallback — 應對 Gemini UI 改版。

延遲目標： task_memory > spec 摘要 > research_knowledge > design 片段。

第三步，從最低優先級開始，超出預算的來源被截斷或只保留摘要。

第四步，budget_usage 欄位回報各來源佔用的 token 數與被截斷的來源名稱，供除錯用。

5. MCP Tool 完整角色表（15 支）

| Tool | 使用時機 | 參與閉環 | 核心引擎 |
|------|---------|---------|----------|
| free_llm_query | 需要 LLM 回應時 | 1, 4 | weblm-driver + Gemini API + Ollama |
| memory_store | 任務完成 / 研究完成 / 設計決策時 | 1, 4 | Pyramid Memory（含 namespace） |
| memory_search | 開始新任務 / 需要歷史經驗時 | 1, 4 | Pyramid Memory（含 namespace + token_count） |
| memory_compress | 定期維護 / 記憶過多時 | 1 | Pyramid Memory（按 namespace 分別壓縮） |
| parse_protocol | 每次呼叫 LLM 前 | 1, 2, 3, 4 | NeuroShunter（含 context budget） |
| free_notebooklm | 遇到未知領域需研究時 | 4 | CLI-Anything-Web |
| spec_read | 實作前讀取規格 | 3 | 檔案系統 |
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
處理 Pyramid Memory 深度視覺化：namespace 瀏覽、壓縮狀態、日記輪轉、容量趨勢。從 Hermes Dashboard 加連結跳轉。

開發介面 — Claude Code / OpenCode
平板環境透過 GitHub Codespaces + Claude Code Remote Control。

設計介面 — Stitch（https://stitch.withgoogle.com/）
UI 設計，產出 design.md。平板瀏覽器可直接使用。

驗證介面 — Opal（Phase 0 限定）
流程概念驗證，不納入自動化。

7. weblm-driver 整合架構

7.1 browser-loop.js 與 weblm-driver 的邊界

┌──────────────────────────────────────┐
│  browser-loop.js（HMM 專案）          │
│                                       │
│  職責：                               │
│  · MCP Tool 格式封裝                  │
│  · 三層 fallback 調度                 │
│  · provider 欄位標記                  │
│  · fallback_attempts 記錄             │
│  · Gemini API 呼叫                    │
│  · Ollama 呼叫                        │
│                                       │
│  ┌───────────────────────────────┐    │
│  │  weblm-driver（npm dependency） │    │
│  │                               │    │
│  │  職責：                       │    │
│  │  · Playwright 生命週期        │    │
│  │  · 頁面狀態偵測               │    │
│  │  · Prompt 提交                │    │
│  │  · 輸出擷取（雙階段）         │    │
│  │  · 四級恢復                   │    │
│  │  · Typed errors               │    │
│  │  · Structured logging         │    │
│  │  · Selector fallback          │    │
│  │  · outputKind 分類            │    │
│  │  · 134 unit tests             │    │
│  └───────────────────────────────┘    │
└──────────────────────────────────────┘

7.2 HMM 不需要實作的功能（由 weblm-driver 提供）

| 功能 | weblm-driver 模組 | 省下的工時 |
|------|-------------------|-----------|
| Playwright 瀏覽器啟動/關閉 | BrowserSession | ~45 min |
| Google 登入狀態偵測 | PageStateInspector | ~30 min |
| CAPTCHA / challenge 偵測 | PageStateInspector | ~20 min |
| Prompt 填入 + 送出 | PromptSubmitter | ~20 min |
| 雙階段輸出擷取 | OutputCapture | ~45 min |
| 四級恢復決策矩陣 | RecoveryManager | ~30 min |
| CSS selector 23 候選 fallback | gemini/selectors.ts | ~15 min |
| 8 個 typed error 類別 | errors/index.ts | ~15 min |
| 合計 | | ~3.5 hr |

7.3 HMM 需要自行實作的功能

| 功能 | 檔案 | 說明 |
|------|------|------|
| 三層 fallback 調度邏輯 | browser-loop.js | weblm-driver 失敗 → Gemini API → Ollama |
| MCP Tool 輸入/輸出格式轉換 | browser-loop.js | MCP JSON-RPC ↔ weblm-driver GenerateInput/Output |
| provider 標記 | browser-loop.js | 標記實際使用的後端 |
| fallback_attempts 記錄 | browser-loop.js | 記錄每次嘗試結果 |
| Gemini API 呼叫 | browser-loop.js | @google/generative-ai SDK |
| Ollama 呼叫 | browser-loop.js | HTTP POST to localhost:11434 |
| outputKind 檢查 | browser-loop.js | 非 "normal" 視為失敗進入 fallback |

8. 日常運作總覽

持續運行：閉環 1（學習迴圈）— 每次 Hermes 任務自動觸發。

按需觸發：閉環 2（設計迭代）— UI 模組開發期間；閉環 3（Spec 演化）— 實作發現缺漏時；閉環 4（知識迴圈）— 遇到未知領域時。

自動執行：sync_check — 每次 design.md 或 spec-*.md 被 commit 時由 CI 觸發。memory_compress — 可設為 Hermes Cron 定期執行（如每日凌晨）。

背景監控：browser-loop.js 可定期呼叫 driver.health() 回報 weblm-driver 的五維健康狀態（initialized / browserRunning / pageReady / authenticated / providerReachable），紀錄到 MCP Server 日誌。

版本紀錄
- v1.0（2026-04-15）：初版建立
- v1.1（2026-04-15）：同步 constitution v1.1——namespace、sync_check、fallback、context budget、15 tools
- v1.2（2026-04-15）：整合 weblm-driver——更新 Mermaid 圖標註 weblm-driver、重寫三層 Fallback 為委託模式、新增 §7 weblm-driver 整合架構（邊界圖、省時表、自行實作表）、§8 新增 health() 背景監控
