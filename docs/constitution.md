docs/constitution.md v1.2

最後更新：2026-04-15 | 版本：v1.2（整合 super-engine / weblm-driver）
倉庫：https://github.com/RYN6666999/HMM

1. 使命與願景

使命：打造一個零成本、模組化、Agent 無關的 AI 開發加速器，讓任何人 clone 後即可用 AI 輔助開發。

願景：以 MCP 為膠水、Spec 為護欄、Pyramid Memory 為靈魂，串聯免費 LLM、Google 工具鏈與自主學習 Agent，形成持續進化的開發生態。

核心價值：

- MCP 為核心：所有模組只透過 MCP Tool 介面溝通，Agent 可自由替換。
- 學習迴圈為靈魂：系統從每次任務中累積經驗，越用越強。
- Spec 驅動開發：先寫規格再寫程式碼，杜絕 vibe coding。
- 零成本優先：預設全部免費（Browser-in-the-Loop + Ollama），付費為加速選項。
- 複用優先：已驗證的模組直接引入，不重複造輪子。

2. 系統架構

2.1 三層架構

┌─────────────────────────────────────────────┐
│              Agent Layer                     │
│  Hermes ∙ Claude Code ∙ OpenCode ∙ Gemini   │
└──────────────────┬──────────────────────────┘
                   │ MCP Protocol (JSON-RPC)
┌──────────────────▼──────────────────────────┐
│           MCP Universal Server               │
│  ┌──────────┐ ┌───────────┐ ┌─────────────┐ │
│  │  Golem   │ │  Google   │ │  External   │ │
│  │  Engine  │ │  Tools    │ │  Tools      │ │
│  └──────────┘ └───────────┘ └─────────────┘ │
│       ▲                                      │
│       │ npm dependency                       │
│  ┌────┴─────┐                                │
│  │ weblm-   │ ← super-engine repo            │
│  │ driver   │   (v0.1.5, 134 tests)          │
│  └──────────┘                                │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           Infrastructure                     │
│  GitHub CI ∙ Docker ∙ Firebase (optional)    │
└─────────────────────────────────────────────┘

2.2 四大閉環架構

flowchart TB
    NLM["🔬 NotebookLM"] --> SPEC["📋 Spec Kit"]
    SPEC --> STITCH["🎨 Stitch SDK"]
    STITCH --> CODE["⌨️ Claude Code / OpenCode"]
    CODE --> MCP["🔌 MCP Server Core15 支 Tool"]
    MCP --> HERMES["🤖 Hermes Agent"]
    HERMES --> USER["👤 使用者"]
    CODE --> GH["📦 GitHub CI/CD"]

    MCP --- MEMORY["🧠 Pyramid Memory3 namespaces"]
    MCP --- BITL["🌐 weblm-driver v0.1.5Browser-in-the-Loop"]
    MCP --- GOOGLE["☁️ Google Tools"]
    MCP --- CLI["🖥️ CLI-Anything"]
    MCP --- FS["📁 檔案系統"]

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
    BITL -->|"四級恢復皆失敗"| API["⚡ Gemini API免費額度"]
    API -->|"失敗"| OLLAMA["🏠 Ollama本地模型"]

    %% Context Budget
    MCP -->|"呼叫 LLM 前"| NS["🧮 NeuroShuntercontext budget 管理"]
    NS -->|"裁剪後的 context"| BITL

閉環 1 — 學習迴圈（Learning Loop）
觸發條件：Hermes 完成任何任務。流程：Hermes 執行任務 → 結果 + metadata 存入 Pyramid Memory（namespace: task_experience） → 未來相似任務時 memory_search 召回經驗 → 經驗附帶 token_count 傳入 NeuroShunter → NeuroShunter 按 context budget 裁剪後注入 LLM prompt → 輸出品質提升。壓縮機制按 namespace 分別執行。頻率：每次任務自動觸發。預期效益：約 40% 效率提升。

閉環 2 — 設計迭代（Design Iteration Loop）
觸發條件：UI 審查不通過。流程：Stitch → design.md → Claude Code 生成 UI → 審查 → 不滿意回 Stitch。每次 design.md commit 時 CI 自動呼叫 sync_check。頻率：每個 UI 模組約 2-5 次迭代。

閉環 3 — Spec 演化（Spec Evolution Loop）
觸發條件：實作中發現 spec 缺漏。流程：Claude Code 呼叫 spec_update → spec commit → CI 呼叫 sync_check 反向比對 design.md。頻率：每個模組約 1-3 次更新。

閉環 4 — 知識迴圈（Knowledge Loop）
觸發條件：Hermes 遇到未知領域。流程：free_notebooklm → 研究摘要 → memory_store（namespace: research_knowledge）。頻率：低頻但高價值。

3. MCP Tool Registry

3.1 優先級總表

| 優先級 | Tool 名稱 | 說明 | 核心引擎 |
|--------|-----------|------|----------|
| P0 | free_llm_query | 免費 LLM 查詢（三層 fallback） | weblm-driver + Gemini API + Ollama |
| P0 | memory_store | 儲存記憶（含 namespace） | Pyramid Memory |
| P0 | memory_search | 搜尋記憶（含 token_count） | Pyramid Memory |
| P1 | memory_compress | 壓縮記憶（按 namespace 分別） | Pyramid Memory |
| P1 | parse_protocol | 協議解析 + Context 預算管理 | NeuroShunter |
| P1 | free_notebooklm | 免費 NotebookLM 研究 | CLI-Anything-Web |
| P1 | spec_read | 讀取並解析 spec 文件 | 檔案系統 |
| P1 | spec_update | 更新 spec 並自動 commit | 檔案系統 + Git |
| P1 | file_read | 讀取允許路徑的檔案 | 檔案系統 |
| P1 | file_write | 寫入允許路徑的檔案 | 檔案系統 |
| P1 | file_list | 列出目錄內容 | 檔案系統 |
| P1 | sync_check | 比對 design.md 與 spec 一致性 | 檔案系統 |
| P2 | desktop_control | 桌面操控 | Playwright |
| P2 | design_generate | UI 設計生成 | Stitch SDK |
| P2 | code_generate | 程式碼生成 | Gemini API / OpenCode |

3.2 Tool JSON Schema

完整 JSON Schema 定義見 docs/spec-mcp-server.md 第 4-6 章。

3.3 安全性設定

所有 file_* 工具在 mcp-config.json 中設定允許路徑：

{
  "file_security": {
    "allowed_paths": [
      "./docs/", "./mcp-server/", "./design/", "./knowledge/",
      "./agent-config/", "./scripts/", "./src/", "./packages/", "./test/"
    ],
    "denied_patterns": [
      "/.env", "/.key", "/.pem", "/node_modules/", "/.git/"
    ],
    "max_file_size_bytes": 1048576
  }
}

4. 架構決策記錄（ADR）

ADR-001：混合架構 — Golem Engine + Hermes Agent

狀態：已採納

背景：需要決定 Hermes Agent 的角色。

備選方案：A) 完全用 Hermes 取代 Golem。B) 混合架構。C) 只用 Golem 不加 Hermes。

決策：選擇 B。Golem 負責核心引擎（BitL、Memory、NeuroShunter），Hermes 負責任務排程與自學習迴圈。

風險與緩解：

風險 A — Browser-in-the-Loop 延遲與脆弱性：weblm-driver 已內建四級恢復（refresh → reopen → restart → rebuild）與 reason-aware 路由。在此之上，free_llm_query 再疊加兩層 fallback（Gemini API → Ollama），形成總共六級容錯。

風險 B — Context window 溢出：parse_protocol（NeuroShunter）內建 context budget 管理，按優先級分配 token。memory_search 回傳 token_count 供計算。

**風險 C —「clone 到可用   → 功能開發
spec/     → 規格撰寫
hotfix/    → 緊急修復

6.2 Commit 訊息格式

(): 

type: feat | fix | docs | spec | design | refactor | test | ci | chore
scope: mcp-server | golem-engine | dashboard | memory | hermes | stitch | ci | weblm

範例：
feat(golem-engine): integrate weblm-driver as browser-loop backend
feat(mcp-server): implement free_llm_query with 3-tier fallback
docs(constitution): v1.2 add ADR-005 weblm-driver integration

6.3 CI Pipeline

每次 push 或 PR 到 main / develop：npm ci → lint → test:unit → test:integration → arch:check → sync_check（Phase 2 後啟用）。

7. 非功能性需求（NFR）

| 項目 | 指標 |
|------|------|
| 啟動時間（Docker） |  80%（weblm-driver 自帶 134 個測試另計） |
| 長期記憶容量 | ≈ 3 MB / 50 年 |
| 文件語言 | 繁體中文（主）+ 英文 |
| 授權 | MIT |
| 月成本（免費方案） | $0 |

8. 路線圖

Phase 0 — 基礎建設
- [x] Golem 引擎拆解與移植
- [x] MCP 萬用接口基礎實作
- [x] GitHub repo 建立
- [x] constitution.md v1.0
- [x] system-flow.md v1.0
- [x] NotebookLM 架構審查
- [x] constitution.md v1.1（NotebookLM 修正）
- [x] system-flow.md v1.1
- [x] spec-mcp-server.md v1.0
- [x] 發現 super-engine 可複用，更新架構
- [x] constitution.md v1.2 ← 目前此處

Phase 1 — 規格撰寫
- [x] docs/spec-mcp-server.md v1.0
- [ ] docs/spec-mcp-server.md v1.1（整合 weblm-driver）← 下一步
- [ ] Spec Kit 初始化（有電腦時）

Phase 2 — UI 設計
- [ ] Stitch Dashboard 設計
- [ ] design/design.md 匯出

Phase 3 — MCP Server Core 實作（~21 小時，省 7 小時）
- [ ] 目錄結構 + mcp-config.json
- [ ] security-manager.js
- [ ] memory_store / memory_search / memory_compress
- [ ] browser-loop.js（封裝 weblm-driver + 三層 fallback）← 省 3 小時
- [ ] neuro-shunter.js
- [ ] file tools + spec tools + sync_check
- [ ] index.js 主入口
- [ ] 測試（weblm-driver 134 測試另計）← 省 3 小時
- [ ] Dockerfile 初版

Phase 4 — Google 工具鏈整合
- [ ] design_generate / free_notebooklm / code_generate / desktop_control

Phase 5 — Agent 增強
- [ ] Hermes Agent 接入 + Skill 定義 + 學習迴圈驗證

Phase 6 — 開源釋出
- [ ] docker-compose.yml + init.sh + README + CONTRIBUTING

9. 成本矩陣

| 項目 | 免費方案 | 付費方案 |
|------|---------|---------|
| LLM（Gemini） | weblm-driver Browser-in-the-Loop $0 | Gemini API ~$7/M tokens |
| LLM（Claude） | — | Claude API ~$15/M tokens |
| LLM（本地） | Ollama $0 | — |
| UI 設計 | Stitch 免費額度 $0 | — |
| 知識庫 | NotebookLM 免費 $0 | NotebookLM Plus ~$20/月 |
| 程式碼託管 | GitHub Free $0 | GitHub Pro $4/月 |
| Agent | Hermes 免費 $0 | — |
| 雲端運行 | Codespaces 120h/月 $0 | VPS $5-20/月 |
| 合計 | $0/月 | $24-51/月 |

10. 文件層級與衝突規則

constitution.md（最高層級）
  ├── system-flow.md
  ├── spec-*.md
  ├── design.md（須通過 sync_check）
  ├── CLAUDE.md
  ├── SKILL.md
  └── README.md

衝突時以 constitution.md 為準。更新需 PR 審查並更新版本號。

11. 詞彙表

| 術語 | 說明 |
|------|------|
| MCP | Model Context Protocol — Agent 與工具間的通用通訊協議 |
| weblm-driver | super-engine repo 的 npm 套件名，提供 Browser-in-the-Loop 驅動 |
| GeminiWebDriver | weblm-driver 的核心類別，實作 WebLLMDriver 介面 |
| Browser-in-the-Loop（BitL） | 透過 Playwright 操控瀏覽器存取免費 Gemini |
| Pyramid Memory | 分層壓縮記憶系統（即時→日→週→月→年），含 namespace 分區 |
| NeuroShunter | 協議解析與 context budget 管理引擎 |
| Reflex Shunting | NeuroShunter 的快速路由機制 |
| GOLEM_PROTOCOL | Golem 引擎內部通訊格式 |
| SDD | Spec-Driven Development |
| Namespace | Pyramid Memory 記憶分區（task_experience / research_knowledge / design_decision） |
| Context Budget | 每次 LLM 呼叫的 token 分配上限 |
| sync_check | 自動比對 design.md 與 spec 一致性的 MCP Tool |
| 三層 Fallback | free_llm_query 容錯：weblm-driver（含四級恢復）→ Gemini API → Ollama |
| 四級恢復 | weblm-driver 內建：refresh-page → reopen-page → restart-browser → rebuild-session |

12. 參考連結

| 資源 | 連結 |
|------|------|
| Project Golem（原始引擎） | https://github.com/Arvincreator/project-golem |
| Project Golem（新框架 HMM） | https://github.com/RYN6666999/HMM |
| weblm-driver（super-engine） | https://github.com/RYN6666999/super-engine |
| GitHub Spec Kit | https://github.com/github/spec-kit |
| Google Stitch SDK | https://www.npmjs.com/package/@google/stitch-sdk |
| Google Opal | https://opal.withgoogle.com/ |
| Google AI Studio | https://aistudio.google.com/ |
| Google NotebookLM | https://notebooklm.google.com/ |
| Hermes Agent | https://github.com/NousResearch/hermes-agent |
| OpenCode CLI | https://github.com/nicepkg/opencode |
| CLI-Anything | https://github.com/nicepkg/cli-anything |
| MCP Specification | https://spec.modelcontextprotocol.io/ |

版本紀錄
- v1.0（2026-04-15）：初版建立
- v1.1（2026-04-15）：NotebookLM 審查修正——namespace、sync_check、6 新 Tool、ADR 風險緩解
- v1.2（2026-04-15）：整合 super-engine/weblm-driver——新增 ADR-005、更新架構圖與依賴路徑、更新路線圖預估時間、新增複用優先原則
