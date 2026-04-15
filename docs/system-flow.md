
docs/system-flow.md v1.3

最後更新：2026-04-15 | 對應：constitution.md v1.3
倉庫：https://github.com/RYN6666999/HMM

1. 完整系統流程圖mermaid
flowchart TB
    NLM["🔬 NotebookLM\n知識研究"] --> SPEC["📋 Spec Kit\n規格撰寫"]
    SPEC --> STITCH["🎨 Stitch SDK\nUI 設計"]
    STITCH --> CODE["⌨️ Claude Code / OpenCode\n程式碼實作"]
    CODE --> MCP["🔌 FastMCP Server\n15 支 Tool"]
    MCP --> HERMES["🤖 Hermes Agent\n任務執行 + 自學習"]
    HERMES --> USER["👤 使用者\nTelegram / Discord / Web"]
    CODE --> GH["📦 GitHub CI/CD\n測試 + sync_check"]

    MCP --- MEMORY["🧠 Pyramid Memory\n3 namespaces\n（LanceDB 自建）"]
    MCP --- GOOGLE["☁️ Google Tools\nStitch · NotebookLM · AI Studio"]
    MCP --- CLI["🖥️ CLI-Anything"]
    MCP --- FS["📁 @mcp/server-filesystem\n（Anthropic 官方套件）\nread · write · list · search · tree"]

    %% weblm-driver 與三層 Fallback
    MCP --- BL["🔌 browser-loop.ts\n三層 Fallback 調度器"]
    BL --> WEBLM["🌐 weblm-driver v0.1.5\nGeminiWebDriver\n四級恢復 · 134 tests"]
    BL -->|"weblm-driver 全部失敗"| API["⚡ Gemini API\n免費額度 15次/分"]
    BL -->|"API 也失敗"| OLLAMA["🏠 Ollama\n本地模型"]

    %% weblm-driver 內部四級恢復
    WEBLM -.->|"refresh → reopen → restart → rebuild"| WEBLM

    %% 閉環 1：學習迴圈
    HERMES -->|"經驗存入\nnamespace: task_experience"| MEMORY
    MEMORY -->|"經驗召回\nmemory_search + token_count"| MCP

    %% 閉環 2：設計迭代
    CODE -->|"UI 不滿意 → screen.edit()"| STITCH

    %% 閉環 3：Spec 演化
    CODE -->|"發現 spec 缺漏 → spec_update"| SPEC
    MCP -->|"發現邊界缺陷 → spec_update"| SPEC

    %% 閉環 4：知識迴圈
    HERMES -->|"未知領域 → free_notebooklm"| NLM
    NLM -->|"研究結果存入\nnamespace: research_knowledge"| MEMORY

    %% 閉環 2 ↔ 3 同步檢查
    GH -->|"design.md 變更 → sync_check"| SPEC
    GH -->|"spec-*.md 變更 → sync_check"| STITCH

    %% Context Budget
    MCP -->|"呼叫 LLM 前"| NS["🧮 NeuroShunter\ncontext budget 管理"]
    NS -->|"裁剪後的 context"| BL
2. 四大閉環詳述

2.1 閉環 1 — 學習迴圈（Learning Loop）

觸發條件：Hermes 完成任何任務。

資料流向：Hermes 執行任務 → 結果 + metadata 呼叫 memory_store（namespace: task_experience）存入 Pyramid Memory（LanceDB 自建）→ 未來相似任務時 memory_search（namespace: task_experience）召回經驗 → 經驗附帶 token_count 傳入 NeuroShunter → NeuroShunter 按 context budget 裁剪後注入 LLM prompt → 輸出品質提升。

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

資料流向：Claude Code/OpenCode 實作時發現問題 → 呼叫 spec_update（指定 spec 路徑、章節、新內容）→ spec_update 底層委託 @modelcontextprotocol/server-filesystem 讀寫檔案，上層處理 section 解析與 git commit → 任務清單重新生成 → 程式碼重新實作。MCP Server 運行時發現缺陷也可觸發 spec_update。

同步機制：每次 spec-*.md 被 commit，CI 自動呼叫 sync_check 反向比對 design.md。

頻率：每個模組約 1-3 次更新。

2.4 閉環 4 — 知識迴圈（Knowledge Loop）

觸發條件：Hermes 遇到未知領域或需要深度研究。

資料流向：Hermes 呼叫 free_notebooklm（query + 參考來源）→ NotebookLM 進行研究 → 回傳 summary + key_findings → 研究結果呼叫 memory_store（namespace: research_knowledge）存入 Pyramid Memory → 未來相同領域查詢時 memory_search（namespace: research_knowledge）直接召回。

頻率：低頻但高價值。

3. 三層 Fallback 機制（free_llm_query）

free_llm_query 被呼叫時，由 browser-loop.ts 調度三個層級。第一層委託 weblm-driver（super-engine）的 GeminiWebDriver，不自己管 Playwright。

3.1 第一層 — weblm-driver（GeminiWebDriver）

browser-loop.ts 呼叫 GeminiWebDriver.generate({ prompt, timeoutMs }) → weblm-driver 內部處理完整的 Playwright 生命週期：啟動瀏覽器 → 偵測登入狀態 → 提交 prompt → 雙階段輪詢擷取輸出（first-token timeout + stability check + stop-button 偵測）→ 回傳 GenerateOutput。

若 generate() 拋出 recoverable DriverError，browser-loop.ts 呼叫 driver.recover(error.code) 進入 weblm-driver 的四級恢復：
refresh-page    → 重新整理頁面，清除卡住的 generation 狀態
reopen-page     → 關閉再重開頁面，重新載入 SPA
restart-browser → 關閉整個瀏覽器 context，重新啟動
rebuild-session → 完全重建 session（最後手段）
恢復成功後重試一次 generate()。若恢復失敗（recovery.ok === false）或第二次 generate() 仍然拋錯，才進入第二層。

weblm-driver 提供的額外保障（HMM 不需要自己實作）：
- outputKind 分類（normal / provider-error / unknown）— browser-loop.ts 檢查此欄位，非 normal 視為失敗進入 fallback。
- ConcurrentGenerationError — 防止同時多次呼叫。
- health() — 非拋出式健康檢查（5 個維度），browser-loop.ts 可定期呼叫回報到 FastMCP Server 日誌。
- Structured logging — JSON-Lines 記錄到 stderr，15 種事件涵蓋完整生命週期。
- 23 個 CSS selector fallback — 應對 Gemini UI 改版。

延遲目標： task_memory（任務經驗）> spec（規格摘要）> research_knowledge（研究知識）> design（設計片段）。

第三步，從最低優先級開始，超出預算的來源被截斷或只保留摘要。

第四步，budget_usage 欄位回報各來源佔用的 token 數與被截斷的來源名稱，供除錯用。

設計參考：SimpleMem 論文（arXiv 2601.02553）的三階段 pipeline（semantic compression → adaptive retrieval → budget allocation）。

5. MCP Tool 完整角色表（15 支）

| Tool | 使用時機 | 參與閉環 | 核心引擎 | 來源 |
|------|---------|---------|----------|------|
| free_llm_query | 需要 LLM 回應時 | 1, 4 | weblm-driver + Gemini API + Ollama | 自建 |
| memory_store | 任務完成 / 研究完成 / 設計決策時 | 1, 4 | Pyramid Memory（LanceDB 自建，含 namespace） | 自建 |
| memory_search | 開始新任務 / 需要歷史經驗時 | 1, 4 | Pyramid Memory（LanceDB 自建，含 namespace + token_count） | 自建 |
| memory_compress | 定期維護 / 記憶過多時 | 1 | Pyramid Memory（LanceDB 自建，按 namespace 分別壓縮） | 自建 |
| parse_protocol | 每次呼叫 LLM 前 | 1, 2, 3, 4 | NeuroShunter（自建，含 context budget） | 自建 |
| free_notebooklm | 遇到未知領域需研究時 | 4 | CLI-Anything-Web | 自建 |
| spec_read | 實作前讀取規格 | 3 | filesystem server + section 解析 wrapper | 混合 |
| spec_update | 實作中發現 spec 缺漏時 | 3 | filesystem server + git commit wrapper | 混合 |
| read_text_file | 讀取 log / config / 任意檔案 | — | @modelcontextprotocol/server-filesystem（官方） | 借力 |
| write_file | 寫入 config / 生成檔案 | — | @modelcontextprotocol/server-filesystem（官方） | 借力 |
| list_directory | 列出目錄結構 | — | @modelcontextprotocol/server-filesystem（官方） | 借力 |
| sync_check | design.md 或 spec 被 commit 時 | 2, 3 | 自建純邏輯（CI 自動觸發） | 自建 |
| desktop_control | 需要操控桌面應用時 | — | Playwright | 自建 |
| design_generate | 需要 UI 設計時 | 2 | Stitch SDK | 自建 |
| code_generate | 需要程式碼生成時 | — | Gemini API / OpenCode | 自建 |

注意：v1.3 起，file_read / file_write / file_list 已改為 @modelcontextprotocol/server-filesystem 的原生工具名稱 read_text_file / write_file / list_directory。額外可用的官方工具包含 search_files、directory_tree、edit_file、get_file_info、move_file、create_directory、read_media_file、read_multiple_files、list_directory_with_sizes、list_allowed_directories，共 13 個檔案操作工具，無需自建。

6. 借力架構圖（v1.3 新增）
┌──────────────────────────────────────────────────────────┐
│  FastMCP 框架（npm: fastmcp）                             │
│  · server.addTool() 註冊所有 Tool                         │
│  · JSON-RPC over stdio                                    │
│  · Session 管理 · Health endpoint · Error handling         │
│  · Progress · Streaming · CLI 測試                         │
│                                                           │
│  ┌─────────────────────────┐  ┌────────────────────────┐  │
│  │  Golem Engine（自建）    │  │  官方 Filesystem Server │  │
│  │                         │  │  （npm 借力）            │  │
│  │  ┌───────────────────┐  │  │                        │  │
│  │  │ browser-loop.ts   │  │  │  read_text_file        │  │
│  │  │ · weblm-driver    │  │  │  write_file            │  │
│  │  │ · Gemini API      │  │  │  list_directory        │  │
│  │  │ · Ollama          │  │  │  search_files          │  │
│  │  └───────────────────┘  │  │  directory_tree        │  │
│  │                         │  │  edit_file             │  │
│  │  ┌───────────────────┐  │  │  get_file_info         │  │
│  │  │ pyramid-memory.ts │  │  │  move_file             │  │
│  │  │ · LanceDB 自建    │  │  │  create_directory      │  │
│  │  │ · namespace 分區  │  │  │  ... 共 13 個工具       │  │
│  │  │ · 分層壓縮        │  │  │                        │  │
│  │  └───────────────────┘  │  │  allowed-path sandbox  │  │
│  │                         │  │  （命令列參數控制）      │  │
│  │  ┌───────────────────┐  │  └────────────────────────┘  │
│  │  │ neuro-shunter.ts  │  │                              │
│  │  │ · context budget  │  │  ┌────────────────────────┐  │
│  │  │ · SimpleMem 參考  │  │  │  薄 Wrapper 層          │  │
│  │  └───────────────────┘  │  │                        │  │
│  │                         │  │  spec-read.ts          │  │
│  │  ┌───────────────────┐  │  │  · filesystem → 讀檔   │  │
│  │  │ security-manager  │  │  │  · section 解析        │  │
│  │  │ · denied patterns │  │  │                        │  │
│  │  │ · .env/.key 過濾  │  │  │  spec-update.ts        │  │
│  │  └───────────────────┘  │  │  · filesystem → 寫檔   │  │
│  │                         │  │  · git commit          │  │
│  └─────────────────────────┘  │                        │  │
│                               │  sync-check.ts         │  │
│                               │  · 純邏輯比對          │  │
│                               └────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
7. weblm-driver 整合架構

7.1 browser-loop.ts 與 weblm-driver 的邊界
┌──────────────────────────────────────┐
│  browser-loop.ts（HMM 專案）          │
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

7.3 HMM 不需要實作的功能（由 FastMCP 提供，v1.3 新增）

| 功能 | FastMCP 模組 | 省下的工時 |
|------|-------------|-----------|
| JSON-RPC over stdio 路由 | FastMCP core | ~45 min |
| Tool 註冊 + inputSchema 驗證 | server.addTool() | ~30 min |
| Session 管理 | FastMCPSession | ~20 min |
| Error 格式化（MCP 標準） | UserError | ~15 min |
| Health endpoint | health config | ~10 min |
| Progress reporting | reportProgress | ~10 min |
| Shutdown hook | FastMCP lifecycle | ~10 min |
| CLI 測試/除錯 | fastmcp dev / inspect | ~20 min |
| 合計 | | ~2.5 hr |

7.4 HMM 不需要實作的功能（由 @modelcontextprotocol/server-filesystem 提供，v1.3 新增）

| 功能 | filesystem server 工具 | 省下的工時 |
|------|----------------------|-----------|
| 檔案讀取 + UTF-8 處理 | read_text_file | ~30 min |
| 檔案寫入 + 覆蓋保護 | write_file | ~30 min |
| 目錄列表 | list_directory | ~20 min |
| 遞迴搜尋 | search_files | ~15 min |
| 目錄樹 | directory_tree | ~15 min |
| allowed-path sandbox | 命令列參數 | ~30 min |
| 檔案 metadata | get_file_info | ~10 min |
| 檔案搬移 | move_file | ~10 min |
| 目錄建立 | create_directory | ~5 min |
| 合計 | | ~2.75 hr |

7.5 HMM 需要自行實作的功能

| 功能 | 檔案 | 說明 |
|------|------|------|
| FastMCP 初始化 + Tool 註冊 | index.ts | new FastMCP() + server.addTool() × N |
| 三層 fallback 調度邏輯 | browser-loop.ts | weblm-driver 失敗 → Gemini API → Ollama |
| MCP Tool 輸入/輸出格式轉換 | browser-loop.ts | FastMCP args ↔ weblm-driver GenerateInput/Output |
| provider 標記 | browser-loop.ts | 標記實際使用的後端 |
| fallback_attempts 記錄 | browser-loop.ts | 記錄每次嘗試結果 |
| Gemini API 呼叫 | browser-loop.ts | @google/generative-ai SDK |
| Ollama 呼叫 | browser-loop.ts | HTTP POST to localhost:11434 |
| outputKind 檢查 | browser-loop.ts | 非 "normal" 視為失敗進入 fallback |
| Pyramid Memory 全部邏輯 | pyramid-memory.ts | LanceDB 自建：store / search / compress / namespace |
| NeuroShunter 全部邏輯 | neuro-shunter.ts | context budget 分配 + 裁剪 |
| denied-pattern 補充 | security-manager.ts | .env / .key / .pem / .git/ 過濾 |
| spec section 解析 | spec-read.ts | 委託 filesystem server 讀檔 + 解析 markdown section |
| spec 更新 + git commit | spec-update.ts | 委託 filesystem server 寫檔 + child_process git |
| sync_check 比對邏輯 | sync-check.ts | 純邏輯：解析 design.md 與 spec 比對 |

8. 統一介面架構

主介面 — Hermes Dashboard（http://127.0.0.1:9119）
處理 90% 的日常操作：對話、指令發送、MCP Tool 呼叫、Cron 排程、Session 管理、Token 用量監控、Skill 啟停、API Key 管理。平板可透過 --host 0.0.0.0 遠端存取（建議搭配 VPN）。

輔助介面 — Golem Dashboard
處理 Pyramid Memory 深度視覺化：namespace 瀏覽、壓縮狀態、日記輪轉、容量趨勢。從 Hermes Dashboard 加連結跳轉。

開發介面 — Claude Code / OpenCode
平板環境透過 GitHub Codespaces + Claude Code Remote Control。

設計介面 — Stitch（https://stitch.withgoogle.com/）
UI 設計，產出 design.md。平板瀏覽器可直接使用。

測試介面 — FastMCP CLI（v1.3 新增）
開發期間使用 npx fastmcp dev index.ts 在終端機互動測試每支 Tool，或 npx fastmcp inspect index.ts 開啟 MCP Inspector Web UI 除錯。

驗證介面 — Opal（Phase 0 限定）
流程概念驗證，不納入自動化。

9. 日常運作總覽

持續運行：閉環 1（學習迴圈）— 每次 Hermes 任務自動觸發。

按需觸發：閉環 2（設計迭代）— UI 模組開發期間；閉環 3（Spec 演化）— 實作發現缺漏時；閉環 4（知識迴圈）— 遇到未知領域時。

自動執行：sync_check — 每次 design.md 或 spec-*.md 被 commit 時由 CI 觸發。memory_compress — 可設為 Hermes Cron 定期執行（如每日凌晨）。

背景監控：browser-loop.ts 可定期呼叫 driver.health() 回報 weblm-driver 的五維健康狀態（initialized / browserRunning / pageReady / authenticated / providerReachable），紀錄到 FastMCP Server 日誌。FastMCP 自帶的 health endpoint 可供外部監控存取。

10. 借力節省總覽（v1.3 新增）

| 借力來源 | 省下工時 | 省去的自建項目 |
|---------|---------|---------------|
| weblm-driver v0.1.5 | ~3.5 hr | Playwright 全流程、恢復、selector fallback、typed errors |
| FastMCP | ~2.5 hr | JSON-RPC 路由、Tool 註冊、Session、Error、Health、CLI |
| @modelcontextprotocol/server-filesystem | ~2.75 hr | file_read、file_write、file_list、search、tree、sandbox |
| 合計 | ~8.75 hr | Phase 3 從 28h → 14h（省 50%） |

自建保留項目（核心差異化）：Pyramid Memory（LanceDB 分層壓縮 + namespace）、NeuroShunter（context budget）、三層 Fallback 調度、sync_check、spec wrapper。

版本紀錄

- v1.0（2026-04-15）：初版建立
- v1.1（2026-04-15）：同步 constitution v1.1——namespace、sync_check、fallback、context budget、15 tools
- v1.2（2026-04-15）：整合 weblm-driver——更新 Mermaid 圖標註 weblm-driver、重寫三層 Fallback 為委託模式、新增 §7 weblm-driver 整合架構（邊界圖、省時表、自行實作表）、§8 新增 health() 背景監控
- v1.3（2026-04-15）：混合借力策略——更新 Mermaid 圖（FastMCP Server、@mcp/server-filesystem）、新增 §6 借力架構圖、§7.3 FastMCP 省時表、§7.4 filesystem server 省時表、更新 §5 Tool 表（工具名稱改用官方名、新增來源欄位）、更新 §8 統一介面（新增 FastMCP CLI 測試介面）、新增 §10 借力節省總覽、file 工具改為官方套件名稱（read_text_file / write_file / list_directory）
