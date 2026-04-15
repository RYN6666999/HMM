# Project Golem: Constitution
# 專案憲章 v1.0
# Last updated: 2026-04-15
# Owner: Arvincreator
# Repo: https://github.com/Arvincreator/project-golem

---

## 1. 使命與願景 (Mission)

Project Golem 是一個零成本運行的 AI 開發加速器框架。
透過 MCP (Model Context Protocol) 萬用接口為中樞，
整合 Google 工具鏈與自學習 Agent，
建立一個能隨時間成長、知識累積且 Agent 無關的開發環境。

核心價值觀：
- MCP 是膠水：所有組件透過標準協議溝通，更換任何 Agent 不需改動引擎。
- Spec 是防腐劑：拒絕 Vibe Coding，一切開發以規格為準。
- Memory 是靈魂：系統必須具備長效記憶與自我進化能力。
- 免費是底線：預設不使用付費 API，優先採用 Browser-in-the-Loop。

---

## 2. 系統架構 (Architecture)

### 2.1 三層架構總覽

Agent Layer（Hermes / Claude Code / OpenCode / Gemini CLI）
        |
        | MCP Protocol
        v
MCP Universal Server
  - Golem Engine（Browser-in-the-Loop, Pyramid Memory, NeuroShunter）
  - Google Tools（Stitch SDK, NotebookLM CLI, Firebase）
  - External Tools（CLI-Anything, OpenCode, GitHub, File System）
        |
        v
Infrastructure（GitHub CI/CD, Docker, Firebase）

### 2.2 四個閉環

閉環 1 - 學習迴圈（自動，最高頻）：
Hermes 執行任務 → 結果存入 Pyramid Memory → 壓縮（即時到日到週到月到年）→ 下次任務召回經驗 → MCP 注入 LLM context → 產出更好的結果 → 再次存入。
觸發：每次任務完畢自動觸發。
價值：越用越聰明，約 40% 效能提升。

閉環 2 - 設計迭代（半自動，按需）：
Stitch 產出 design.md → Claude Code 生成 UI 程式碼 → 預覽不滿意 → 回 Stitch screen.edit() 修改 → 更新 design.md → 重新生成 → 滿意為止。
觸發：UI Review 不通過。
頻率：每個 UI 模組 2-5 次迭代。

閉環 3 - Spec 演化（手動，關鍵節點）：
Spec Kit 產出 spec → 實作中發現遺漏或矛盾 → 回 Spec Kit 更新 → 重新 plan 和 tasks → 繼續實作。
觸發：實作中遇到 spec 與現實不符。
價值：Spec 是 living document，保持與程式碼同步。

閉環 4 - 知識迴圈（按需，跨專案）：
遇到未知領域 → 觸發 NotebookLM 研究 → 結果存入 Pyramid Memory → 下次直接召回 → 不需再次研究。
觸發：Agent 遇到知識盲區。
價值：知識永久累積，跨專案複用。

### 2.3 日常運作模式

使用者提需求
  → Hermes 接收 → memory_search 找過去經驗
    → 有經驗 → 直接執行（閉環 1 快速路徑）
    → 沒經驗 → free_llm_query 問 Gemini
      → 需要研究 → NotebookLM（閉環 4）
      → 需要 UI → Stitch SDK（閉環 2）
      → 直接回答 → 存入 Memory → 推送給使用者

---

## 3. MCP Tool Registry

### 3.1 工具總表

P0（最優先）：
- free_llm_query：透過 Browser-in-the-Loop 存取免費 Gemini，對應 Browser-in-the-Loop 模組
- memory_store：儲存任務經驗與知識片段，對應 Pyramid Memory
- memory_search：召回相關經驗與 context，對應 Pyramid Memory

P1（次優先）：
- memory_compress：執行記憶層級壓縮，對應 Pyramid Memory
- parse_protocol：解析 GOLEM_PROTOCOL 結構化協議，對應 NeuroShunter
- free_notebooklm：自動化操作 NotebookLM 進行研究，對應 CLI-Anything-Web

P2（後續）：
- desktop_control：控制 GIMP/Blender 等桌面軟體，對應 CLI-Anything Desktop
- design_generate：呼叫 Stitch 產出 UI 設計，對應 Stitch SDK
- code_generate：驅動編碼引擎實作任務，對應 OpenCode / Gemini API

### 3.2 Tool JSON Schema

free_llm_query:
  input:
    prompt: string, required, 要問 Gemini 的問題
    context: string, optional, 額外 context（如 Memory 召回結果）
    model: string, optional, default "gemini", 模型選擇 gemini 或 ollama
  output:
    content: [{ type: "text", text: string }]
  errors: AUTH_EXPIRED, RATE_LIMITED, NETWORK_ERROR, BROWSER_CRASH

memory_store:
  input:
    data: string, required, 要儲存的內容
    metadata: object, optional, 標籤、來源、時間戳等
    tier: number, optional, default 0, 存入層級 0-4
  output:
    id: string, 儲存後的記憶 ID
    tier: number, 實際存入的層級
  errors: STORAGE_FULL, WRITE_ERROR

memory_search:
  input:
    query: string, required, 搜尋關鍵詞或語意查詢
    limit: number, optional, default 5, 回傳筆數上限
    tier: number, optional, 限定搜尋層級，不填則全層搜尋
  output:
    results: [{ id, data, score, tier, metadata }]
  errors: SEARCH_ERROR, INDEX_CORRUPT

memory_compress:
  input:
    target_tier: number, optional, 壓縮到目標層級，不填則自動判斷
  output:
    compressed: number, 被壓縮的記憶筆數
    from_tier: number
    to_tier: number
  errors: COMPRESS_ERROR, NOTHING_TO_COMPRESS

parse_protocol:
  input:
    raw: string, required, 原始 GOLEM_PROTOCOL 字串
  output:
    type: string, REPLY 或 MEMORY 或 ACTION 或 OBSERVE
    payload: object, 解析後的結構化資料
  errors: PARSE_ERROR, UNKNOWN_PROTOCOL

free_notebooklm:
  input:
    action: string, required, create_notebook 或 add_source 或 query 或 generate_mindmap 或 generate_audio
    query: string, optional, 研究問題（action=query 時必填）
    sources: array, optional, 來源 URL 列表（action=add_source 時必填）
  output:
    result: string, 研究摘要或 mindmap 或 audio URL
    citations: array, 引用來源列表
  errors: AUTH_EXPIRED, NOTEBOOK_NOT_FOUND, SOURCE_LIMIT_EXCEEDED

desktop_control:
  input:
    app: string, required, 目標軟體 gimp 或 blender 或 libreoffice 或 drawio
    action: string, required, 執行動作
    params: object, optional, 動作參數
  output:
    result: string, 執行結果或產出檔案路徑
  errors: APP_NOT_FOUND, ACTION_FAILED, CLI_NOT_INSTALLED

design_generate:
  input:
    prompt: string, required, UI 設計描述
    device: string, optional, default DESKTOP, 可選 MOBILE DESKTOP TABLET AGNOSTIC
    project_id: string, optional, Stitch 專案 ID 不填則建新專案
  output:
    html_url: string, HTML 下載 URL
    image_url: string, 截圖下載 URL
    screen_id: string, Stitch Screen ID
  errors: AUTH_FAILED, RATE_LIMITED, GENERATION_FAILED

code_generate:
  input:
    spec_path: string, required, spec 文件路徑
    task_id: string, required, 要實作的 task 編號
    design_path: string, optional, design.md 路徑
  output:
    files: array, 產出或修改的檔案路徑列表
    summary: string, 實作摘要
  errors: SPEC_NOT_FOUND, TASK_AMBIGUOUS, GENERATION_FAILED

---

## 4. 建築決策紀錄 (ADR)

### ADR-001：Hermes 作為補強而非取代 Golem

背景：
Hermes Agent（42K stars, Python）擁有自學習迴圈、200+ 模型、Sub-agent、排程。
Golem Engine（Node.js）擁有 Browser-in-the-Loop、Pyramid Memory、NeuroShunter、Dashboard。

考慮過的選項：
A. 完全用 Hermes 取代 Golem — 單一系統、社群大，但失去免費 Gemini、Pyramid Memory、Dashboard
B. 混合架構（MCP 橋接）— 互補各取所長，但雙系統維護成本
C. 只用 Golem 加 Hermes Skill — 單一系統，但工程量巨大、失去 Hermes 社群更新

決策：選項 B — 混合架構。

理由：
- Golem 的 BitL 提供免費 Gemini（1M+ tokens），Hermes 無此能力，這是零成本底線的關鍵。
- Pyramid Memory（50 年 / 3MB）是 Hermes SQLite FTS5 無法替代的。
- Hermes 的自學習迴圈（40% 效能提升）是 Golem 手動 /learn 無法比擬的。
- MCP 統一介面降低雙系統耦合。

風險與緩解：
- 雙系統版本不同步 → MCP 介面版本鎖定，兩側獨立升級。
- 記憶系統重疊 → Hermes FTS5 負責短期快速搜尋，Golem Pyramid 負責長期壓縮，職責切分。

### ADR-002：Stitch 優先選用官方 SDK

考慮過的選項：
A. 官方 @google/stitch-sdk — 穩定性高，API Key 認證
B. CLI-Anything-Web cli-web-stitch — 穩定性中，逆向工程可能變動
C. Stitch MCP Proxy — 穩定性高，API Key 認證

決策：A 為主要路徑，B 為 fallback。

理由：官方 SDK 有版本管理、錯誤碼定義、原生 JSON 輸出。逆向工程隨時可能壞掉。

### ADR-003：Spec-Driven Development (SDD)

決策：使用 GitHub Spec Kit，每個模組先寫 spec 再寫 code。

理由：
- Spec 是 AI Agent 最好的 context。
- 提供結構化流程（constitution → specify → plan → tasks → implement）。
- 開源社群貢獻有明確依據。
- 閉環 3 確保 spec 不會過時。

### ADR-004：零成本優先 + Opal 排除

決策：
- Browser-in-the-Loop 作為免費 LLM 主要來源。
- Ollama 本地模型作為離線備援。
- Opal 定位為 Phase 0 白板工具，不進入自動化 pipeline。

Opal 排除理由：
- 沒有公開 API。
- 產出為 Google 託管連結，無法匯出。
- 僅限美國地區。
- 其價值在「快速驗證邏輯」，手動 30 分鐘的事，不需要自動化。

---

## 5. 檔案結構映射

### 5.1 現況與目標

已存在：
- apps/runtime/ — Golem Engine 核心運行環境
- apps/dashboard/ — Dashboard 啟動器（Phase 4 用 Stitch 重新設計）
- src/ — Brain / Memory / Skills / Managers 核心邏輯
- web-dashboard/ — Web UI 與 API 路由（Phase 4 重構）
- packages/ — security / memory / protocol facade（被 MCP Server 引用）
- infra/ — 架構邊界檢查 arch:check（CI 繼續使用）
- docs/ — 既有文件（AGENTS.md、架構藍圖等）
- scripts/ — setup.sh 等
- personas/ — 多人格管理
- data/ — 資料檔
- test/bridges/ — 測試檔

待建立：
- mcp-server/ — MCP 核心中樞
- mcp-server/index.js — MCP Server 進入點
- mcp-server/golem-engine/ — browser-loop.js, pyramid-memory.js, neuro-shunter.js
- mcp-server/tools/ — stitch.js, notebooklm.js, cli-anything.js, opencode-bridge.js
- mcp-server/mcp-config.json — MCP Server 統一組態
- design/ — Stitch 產出的 design.md 與截圖
- design/stitch-prompts/ — Stitch 提示詞模板
- knowledge/ — NotebookLM 產出的研究報告
- knowledge/research-template.md — 研究模板
- agent-config/ — Agent 設定檔目錄
- agent-config/hermes-skill/ — Hermes 技能定義
- agent-config/CLAUDE.md — Claude Code context 規則
- .github/workflows/ci.yml — CI pipeline（如需更新）
- .env.example — 環境變數模板

### 5.2 關鍵依賴關係

mcp-server/golem-engine/browser-loop.js
  呼叫 → packages/protocol/ (ProtocolFormatter)
  呼叫 → apps/runtime/ (Playwright session)

mcp-server/golem-engine/pyramid-memory.js
  呼叫 → packages/memory/ (LanceDBProDriver, ExperienceMemory)

mcp-server/golem-engine/neuro-shunter.js
  呼叫 → packages/protocol/ (NeuroShunter, ResponseExtractor)

mcp-server/tools/stitch.js
  呼叫 → @google/stitch-sdk (外部 npm 套件)

mcp-server/tools/notebooklm.js
  呼叫 → cli-web-notebooklm (外部 Python CLI)

---

## 6. 開發流程與標準

### 6.1 Git 分支策略

- main — 穩定版，僅透過 PR 合入
- develop — 整合分支
- feature/<module-name> — 功能分支
- spec/<module-name> — spec 撰寫分支

### 6.2 Commit 訊息規範

格式：<type>(<scope>): <description>
type: feat, fix, spec, docs, refactor, test, ci
scope: mcp-server, browser-loop, pyramid-memory, neuro-shunter, dashboard, stitch 等

範例：
- spec(mcp-server): add free_llm_query tool definition
- feat(browser-loop): implement Gemini web auto-login
- test(pyramid-memory): add tier compression unit tests
- docs(constitution): v1.0 finalized with 4 closed-loops

### 6.3 CI Pipeline

觸發條件：PR 合併至 develop 或 main。
執行內容：npm run arch:check（架構邊界）→ npm test（單元測試）→ npm run test:integration（整合測試）。

---

## 7. 非功能性需求 (NFR)

- 啟動時間：clone 到可用 < 5 分鐘
- MCP Server 記憶體占用：< 512 MB RAM
- free_llm_query 回應延遲：< 30 秒（含 Browser-in-the-Loop 頁面載入）
- memory_search 回應延遲：< 100 毫秒（10K+ 筆記憶）
- 核心模組測試覆蓋率：> 80%
- 長期記憶容量：約 3 MB / 50 年（Pyramid Memory 壓縮後）
- 文件語言：繁體中文為主，程式碼註解英文
- 授權：MIT License
- 月運行成本（預設）：$0

---

## 8. 路線圖與進度

Phase 0 — 基礎建設：
- [x] Golem 引擎拆解與移植（BitL, Pyramid Memory, NeuroShunter, Security, Skills）
- [x] MCP 萬用接口基礎實作
- [x] GitHub repo 建立
- [x] Web Dashboard 可運行
- [x] Pyramid Memory 5 層壓縮系統可運行
- [x] packages/ facade 建立（security, memory, protocol）
- [x] 架構邊界檢查 npm run arch:check 可運行
- [ ] constitution.md 定稿 ← 目前在這裡
- [ ] system-flow.md 推上 repo

Phase 1 — 規格撰寫：
- [ ] Spec Kit 初始化
- [ ] /speckit.constitution 讀取 constitution.md
- [ ] docs/spec-mcp-server.md 撰寫
- [ ] /speckit.plan + /speckit.tasks 產出

Phase 2 — UI 設計：
- [ ] Stitch API Key 取得
- [ ] Dashboard UI 重新設計
- [ ] design/design.md 產出
- [ ] agent-config/CLAUDE.md 建立

Phase 3 — MCP Server Core 實作：
- [ ] free_llm_query 完整實作與測試
- [ ] memory_store / memory_search 完整實作與測試
- [ ] parse_protocol 實作
- [ ] mcp-server/index.js 可被外部 Agent 呼叫
- [ ] CI pipeline 建立

Phase 4 — Google 工具鏈整合：
- [ ] design_generate（Stitch SDK）整合
- [ ] free_notebooklm（CLI-Anything-Web）整合
- [ ] code_generate（Gemini API / OpenCode）整合
- [ ] desktop_control（CLI-Anything Desktop）整合

Phase 5 — Agent 補強：
- [ ] Hermes Agent 接入 MCP Server
- [ ] 雙記憶體系統啟用（Hermes FTS5 短期 + Golem Pyramid 長期）
- [ ] 自學習迴圈驗證
- [ ] 自然語言排程啟用

Phase 6 — 開源與社群：
- [ ] scripts/init.sh 一鍵初始化腳本
- [ ] README.md 重寫
- [ ] .env.example 環境變數模板
- [ ] CONTRIBUTING.md 貢獻指南
- [ ] Starter Template 可被 clone 並 5 分鐘內跑起來

---

## 9. 成本預算

免費方案（預設）：
- Golem Engine: $0 (MIT)
- Hermes Agent: $0 (MIT)
- LLM Gemini: $0 (Browser-in-the-Loop)
- LLM 本地: $0 (Ollama)
- NotebookLM: $0 (50 sources)
- Stitch: $0 (Beta)
- AI Studio: $0 (每日配額)
- GitHub: $0 (Public repo)
- Spec Kit: $0 (MIT)
- CLI-Anything-Web: $0 (MIT)
- OpenCode CLI: $0 (MIT)
- 伺服器: $0 (本機)
- 總計: $0/月

付費方案（選配）：
- Gemini API: $7/M tokens
- NotebookLM Plus: $20/月
- GitHub Pro: $4/月
- VPS: $5-20/月
- 總計: $24-51/月

---

## 10. 文件層級與既有文件關係

constitution.md（本文件）← 最高層級
  |
  +-- system-flow.md — 系統程序圖與閉環機制詳解
  |
  +-- spec-*.md — 各模組功能規格（Phase 1 產出）
  |
  +-- 既有文件（Phase 0 前已存在）：
      +-- AGENTS.md — AI Agent 使用指南
      +-- 大型產品架構藍圖.md — apps + packages + infra 結構說明
      +-- infra/architecture/README.md — 架構治理與 arch:check 規則
      +-- MCP-使用與開發指南.md — MCP Server 開發細節
      +-- 記憶系統架構說明.md — Pyramid Memory 技術規格
      +-- Web-Dashboard-使用說明.md — Dashboard UI 操作手冊
      +-- 開發者實作指南.md — Skill 與 Golem Protocol 開發
      +-- 指令說明一覽表.md — Telegram/Discord 指令參考

衝突解決原則：當既有文件與 constitution.md 不一致時，以 constitution.md 為準。

---

## 11. 術語對照表

- MCP (Model Context Protocol): Agent 與工具間的標準通訊協議
- BitL (Browser-in-the-Loop): 透過 Playwright 操控瀏覽器存取免費 Web AI 服務
- Pyramid Memory: Golem 的 5 層壓縮記憶系統（即時→日→週→月→年）
- NeuroShunter: Golem 的結構化協議路由器，解析 GOLEM_PROTOCOL
- GOLEM_PROTOCOL: Golem 自定義的 LLM 回應格式（REPLY/MEMORY/ACTION/OBSERVE）
- Reflex Shunting: NeuroShunter 的核心機制，根據協議類型自動分流回應
- SDD (Spec-Driven Development): 規格驅動開發，先寫 spec 再寫 code
- Spec Kit: GitHub 開源的 SDD 工具組
- Stitch: Google AI UI 設計工具，產出 design.md 和 HTML
- Opal: Google AI 無程式碼工作流建構器（Phase 0 白板工具）
- CLI-Anything-Web: 將任意網頁應用逆向工程為 Python CLI 的工具
- FTS5 (Full-Text Search 5): SQLite 全文搜索引擎，Hermes 用於短期記憶檢索

---

## 附錄：參考連結

- Project Golem: https://github.com/Arvincreator/project-golem
- GitHub Spec Kit: https://github.com/github/spec-kit
- Google Stitch: https://stitch.withgoogle.com/
- Stitch SDK: https://github.com/google-labs-code/stitch-sdk
- Google Opal: https://opal.google/
- Google AI Studio: https://aistudio.google.com/
- Google NotebookLM: https://notebooklm.google.com/
- Hermes Agent: https://github.com/nousresearch/hermes-agent
- OpenCode CLI: https://github.com/opencode-ai/opencode
- CLI-Anything: https://github.com/HKUDS/CLI-Anything
- CLI-Anything-Web: https://github.com/ItamarZand88/CLI-Anything-WEB
- MCP Specification: https://spec.modelcontextprotocol.io/
