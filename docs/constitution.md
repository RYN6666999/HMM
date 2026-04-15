
Project Golem — Constitution v1.1

最後更新：2026-04-15 | 版本：v1.1（NotebookLM 審查修正版）
倉庫：https://github.com/RYN6666999/HMM

1. 使命與願景

使命：打造一個零成本、模組化、Agent 無關的 AI 開發加速器，讓任何人 clone 後即可用 AI 輔助開發。

願景：以 MCP 為膠水、Spec 為護欄、Pyramid Memory 為靈魂，串聯免費 LLM、Google 工具鏈與自主學習 Agent，形成持續進化的開發生態。

核心價值：

- MCP 為核心：所有模組只透過 MCP Tool 介面溝通，Agent 可自由替換。
- 學習迴圈為靈魂：系統從每次任務中累積經驗，越用越強。
- Spec 驅動開發：先寫規格再寫程式碼，杜絕 vibe coding。
- 零成本優先：預設全部免費（Browser-in-the-Loop + Ollama），付費為加速選項。

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
    CODE --> MCP["🔌 MCP Server Core"]
    MCP --> HERMES["🤖 Hermes Agent"]
    HERMES --> USER["👤 使用者"]
    CODE --> GH["📦 GitHub CI/CD"]
    MCP --- MEMORY["🧠 Pyramid Memory"]
    MCP --- BITL["🌐 Browser-in-the-Loop"]
    MCP --- GOOGLE["☁️ Google Tools"]
    MCP --- CLI["🖥️ CLI-Anything"]

    %% 閉環 1：學習迴圈
    HERMES -->|"經驗存入 (namespace: task_experience)"| MEMORY
    MEMORY -->|"經驗召回"| MCP

    %% 閉環 2：設計迭代
    CODE -->|"UI 不滿意"| STITCH

    %% 閉環 3：Spec 演化
    CODE -->|"spec 不足"| SPEC
    MCP -->|"發現缺陷"| SPEC

    %% 閉環 4：知識迴圈
    HERMES -->|"需要研究"| NLM
    NLM -->|"研究結果存入 (namespace: research_knowledge)"| MEMORY

    %% 閉環 2 ↔ 3 同步檢查
    STITCH -.->|"sync_check 觸發"| SPEC
    SPEC -.->|"sync_check 觸發"| STITCH

閉環 1 — 學習迴圈（Learning Loop）
觸發條件：Hermes 完成任何任務。流程：Hermes 執行任務 → 結果 + metadata 存入 Pyramid Memory（namespace: task_experience） → 未來相似任務時 memory_search 召回經驗 → 注入 LLM context → 提升輸出品質。頻率：每次任務自動觸發。預期效益：約 40% 效率提升。

閉環 2 — 設計迭代（Design Iteration Loop）
觸發條件：UI 審查不通過。流程：Stitch 產出 design.md → Claude Code 產生 UI 程式碼 → Agent 或人工審查 → 不滿意則呼叫 screen.edit() 回到 Stitch → 匯出更新後的 design.md → 重新生成程式碼。頻率：每個 UI 模組約 2-5 次迭代。新增同步機制：每次 design.md 被 commit，CI 自動呼叫 sync_check 比對 spec 中的 API 端點與 UI 元件名稱，不一致時標記 warning。

閉環 3 — Spec 演化（Spec Evolution Loop）
觸發條件：實作中發現 spec 缺漏或 API 變更。流程：Claude Code/OpenCode 實作時發現邊界案例或欄位缺失 → 呼叫 spec_update 更新 spec → 任務重新執行 → 程式碼重新實作。頻率：每個模組約 1-3 次更新。新增同步機制：每次 spec-*.md 被 commit，CI 自動呼叫 sync_check 反向比對 design.md，確保雙向一致。

閉環 4 — 知識迴圈（Knowledge Loop）
觸發條件：Hermes 遇到未知領域。流程：Hermes 呼叫 free_notebooklm 進行深度研究 → 研究摘要存入 Pyramid Memory（namespace: research_knowledge） → 未來相同領域查詢時直接召回。頻率：低頻但高價值。

3. MCP Tool Registry

3.1 優先級總表

| 優先級 | Tool 名稱 | 說明 | 核心引擎 |
|--------|-----------|------|----------|
| P0 | free_llm_query | 免費 LLM 查詢 | Browser-in-the-Loop |
| P0 | memory_store | 儲存記憶 | Pyramid Memory |
| P0 | memory_search | 搜尋記憶 | Pyramid Memory |
| P1 | memory_compress | 壓縮記憶 | Pyramid Memory |
| P1 | parse_protocol | 協議解析 + Context 預算管理 | NeuroShunter |
| P1 | free_notebooklm | 免費 NotebookLM 研究 | CLI-Anything-Web |
| P1 | spec_read | 讀取並解析 spec 文件 | 檔案系統 |
| P1 | spec_update | 更新 spec 並自動 commit | 檔案系統 + Git |
| P1 | file_read | 讀取任意允許路徑的檔案 | 檔案系統 |
| P1 | file_write | 寫入任意允許路徑的檔案 | 檔案系統 |
| P1 | file_list | 列出目錄內容 | 檔案系統 |
| P1 | sync_check | 比對 design.md 與 spec 一致性 | 檔案系統 |
| P2 | desktop_control | 桌面操控 | Playwright |
| P2 | design_generate | UI 設計生成 | Stitch SDK |
| P2 | code_generate | 程式碼生成 | Gemini API / OpenCode |

3.2 P0 Tool JSON Schema

free_llm_query

{
  "name": "free_llm_query",
  "description": "透過 Browser-in-the-Loop 免費呼叫 Gemini，含三層 fallback",
  "inputSchema": {
    "type": "object",
    "properties": {
      "prompt": { "type": "string", "description": "查詢內容" },
      "context": { "type": "string", "description": "附加上下文（選填）" },
      "model": { "type": "string", "default": "gemini", "enum": ["gemini", "ollama"], "description": "偏好模型（選填）" }
    },
    "required": ["prompt"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "content": {
        "type": "array",
        "items": { "type": "object", "properties": { "type": { "type": "string" }, "text": { "type": "string" } } }
      },
      "provider": { "type": "string", "enum": ["browser", "api", "ollama"], "description": "實際使用的後端" },
      "latency_ms": { "type": "number" }
    }
  },
  "errors": ["AUTH_EXPIRED", "RATE_LIMITED", "NETWORK_ERROR", "CAPTCHA_DETECTED", "ALL_FALLBACKS_FAILED"]
}

memory_store

{
  "name": "memory_store",
  "description": "將記憶存入 Pyramid Memory，按 namespace 分區",
  "inputSchema": {
    "type": "object",
    "properties": {
      "content": { "type": "string", "description": "記憶內容" },
      "namespace": { "type": "string", "enum": ["task_experience", "research_knowledge", "design_decision"], "description": "記憶分區（必填）" },
      "tags": { "type": "array", "items": { "type": "string" }, "description": "標籤（選填）" },
      "metadata": { "type": "object", "description": "附加 metadata（選填）" }
    },
    "required": ["content", "namespace"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "id": { "type": "string" },
      "stored_at": { "type": "string", "format": "date-time" },
      "namespace": { "type": "string" },
      "token_count": { "type": "number" }
    }
  },
  "errors": ["STORAGE_FULL", "INVALID_NAMESPACE", "WRITE_ERROR"]
}

memory_search

{
  "name": "memory_search",
  "description": "搜尋 Pyramid Memory，支援 namespace 過濾",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "搜尋查詢" },
      "namespace": { "type": "string", "enum": ["task_experience", "research_knowledge", "design_decision", "all"], "default": "all", "description": "限定搜尋分區（選填）" },
      "top_k": { "type": "number", "default": 5, "description": "回傳筆數" },
      "min_relevance": { "type": "number", "default": 0.7, "description": "最低相關性閾值" }
    },
    "required": ["query"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "results": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "id": { "type": "string" },
            "content": { "type": "string" },
            "namespace": { "type": "string" },
            "relevance": { "type": "number" },
            "token_count": { "type": "number" },
            "created_at": { "type": "string", "format": "date-time" },
            "compression_level": { "type": "string", "enum": ["instant", "day", "week", "month", "year"] }
          }
        }
      },
      "total_tokens": { "type": "number" }
    }
  },
  "errors": ["QUERY_TOO_LONG", "INDEX_CORRUPTED", "READ_ERROR"]
}

3.3 P1 Tool JSON Schema

memory_compress

{
  "name": "memory_compress",
  "description": "依時間階層壓縮 Pyramid Memory，按 namespace 分別執行",
  "inputSchema": {
    "type": "object",
    "properties": {
      "namespace": { "type": "string", "enum": ["task_experience", "research_knowledge", "design_decision", "all"], "default": "all" },
      "target_level": { "type": "string", "enum": ["day", "week", "month", "year"], "description": "目標壓縮層級" },
      "dry_run": { "type": "boolean", "default": false, "description": "預覽壓縮結果而不執行" }
    },
    "required": ["target_level"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "compressed_count": { "type": "number" },
      "before_tokens": { "type": "number" },
      "after_tokens": { "type": "number" },
      "compression_ratio": { "type": "number" },
      "namespace": { "type": "string" }
    }
  },
  "errors": ["NOTHING_TO_COMPRESS", "COMPRESSION_FAILED", "INVALID_NAMESPACE"]
}

parse_protocol

{
  "name": "parse_protocol",
  "description": "NeuroShunter 協議解析，含 context budget 管理",
  "inputSchema": {
    "type": "object",
    "properties": {
      "input": { "type": "string", "description": "原始輸入" },
      "context_sources": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "source": { "type": "string", "enum": ["conversation", "task_memory", "research_memory", "spec", "design"] },
            "content": { "type": "string" },
            "token_count": { "type": "number" }
          }
        },
        "description": "可用上下文來源（選填）"
      },
      "context_budget": { "type": "number", "default": 100000, "description": "Context token 預算（預設為模型 window 的 80%）" }
    },
    "required": ["input"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "parsed_intent": { "type": "string" },
      "selected_tools": { "type": "array", "items": { "type": "string" } },
      "assembled_context": { "type": "string", "description": "經裁剪後的合併上下文" },
      "budget_usage": {
        "type": "object",
        "properties": {
          "total_budget": { "type": "number" },
          "used": { "type": "number" },
          "breakdown": { "type": "object", "description": "各來源佔用 token 數" },
          "truncated_sources": { "type": "array", "items": { "type": "string" } }
        }
      }
    }
  },
  "errors": ["PARSE_FAILED", "CONTEXT_OVERFLOW", "UNKNOWN_PROTOCOL"]
}

free_notebooklm

{
  "name": "free_notebooklm",
  "description": "透過 CLI-Anything-Web 呼叫 NotebookLM 進行深度研究",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "研究問題" },
      "sources": { "type": "array", "items": { "type": "string" }, "description": "參考來源 URL 或文字（選填）" },
      "notebook_id": { "type": "string", "description": "既有 notebook ID（選填）" }
    },
    "required": ["query"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "summary": { "type": "string" },
      "key_findings": { "type": "array", "items": { "type": "string" } },
      "notebook_id": { "type": "string" },
      "token_count": { "type": "number" }
    }
  },
  "errors": ["AUTH_EXPIRED", "NOTEBOOK_NOT_FOUND", "RESEARCH_TIMEOUT"]
}

spec_read

{
  "name": "spec_read",
  "description": "讀取並解析 spec 文件，回傳結構化內容",
  "inputSchema": {
    "type": "object",
    "properties": {
      "spec_path": { "type": "string", "description": "spec 檔案路徑（如 docs/spec-mcp-server.md）" },
      "section": { "type": "string", "description": "指定章節名稱（選填，不填則回傳全部）" }
    },
    "required": ["spec_path"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "title": { "type": "string" },
      "sections": { "type": "array", "items": { "type": "object", "properties": { "heading": { "type": "string" }, "content": { "type": "string" }, "token_count": { "type": "number" } } } },
      "tools_defined": { "type": "array", "items": { "type": "string" } },
      "acceptance_criteria": { "type": "array", "items": { "type": "string" } },
      "total_token_count": { "type": "number" }
    }
  },
  "errors": ["FILE_NOT_FOUND", "PARSE_ERROR", "SECTION_NOT_FOUND"]
}

spec_update

{
  "name": "spec_update",
  "description": "更新 spec 文件指定章節並自動 commit",
  "inputSchema": {
    "type": "object",
    "properties": {
      "spec_path": { "type": "string", "description": "spec 檔案路徑" },
      "section": { "type": "string", "description": "要更新的章節名稱" },
      "new_content": { "type": "string", "description": "新內容" },
      "commit_message": { "type": "string", "description": "commit 訊息（選填，自動生成）" }
    },
    "required": ["spec_path", "section", "new_content"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "updated": { "type": "boolean" },
      "commit_sha": { "type": "string" },
      "diff_summary": { "type": "string" }
    }
  },
  "errors": ["FILE_NOT_FOUND", "SECTION_NOT_FOUND", "GIT_COMMIT_FAILED", "PERMISSION_DENIED"]
}

file_read

{
  "name": "file_read",
  "description": "讀取允許路徑內的檔案",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "檔案路徑（相對於 repo 根目錄）" },
      "encoding": { "type": "string", "default": "utf-8" }
    },
    "required": ["path"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "content": { "type": "string" },
      "size_bytes": { "type": "number" },
      "token_count": { "type": "number" }
    }
  },
  "errors": ["FILE_NOT_FOUND", "PERMISSION_DENIED", "PATH_NOT_ALLOWED", "FILE_TOO_LARGE"]
}

file_write

{
  "name": "file_write",
  "description": "寫入允許路徑內的檔案",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "檔案路徑（相對於 repo 根目錄）" },
      "content": { "type": "string", "description": "檔案內容" },
      "create_dirs": { "type": "boolean", "default": true, "description": "自動建立父目錄" },
      "overwrite": { "type": "boolean", "default": false, "description": "是否覆蓋既有檔案" }
    },
    "required": ["path", "content"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "written": { "type": "boolean" },
      "size_bytes": { "type": "number" },
      "path": { "type": "string" }
    }
  },
  "errors": ["PERMISSION_DENIED", "PATH_NOT_ALLOWED", "FILE_EXISTS", "WRITE_ERROR"]
}

file_list

{
  "name": "file_list",
  "description": "列出允許路徑內的目錄內容",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "default": ".", "description": "目錄路徑（相對於 repo 根目錄）" },
      "recursive": { "type": "boolean", "default": false },
      "pattern": { "type": "string", "description": "glob 過濾模式（選填，如 *.md）" }
    },
    "required": []
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "entries": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name": { "type": "string" },
            "type": { "type": "string", "enum": ["file", "directory"] },
            "size_bytes": { "type": "number" },
            "modified_at": { "type": "string", "format": "date-time" }
          }
        }
      },
      "total_count": { "type": "number" }
    }
  },
  "errors": ["DIRECTORY_NOT_FOUND", "PERMISSION_DENIED", "PATH_NOT_ALLOWED"]
}

sync_check

{
  "name": "sync_check",
  "description": "比對 design.md 與 spec 文件的一致性，回報差異",
  "inputSchema": {
    "type": "object",
    "properties": {
      "design_path": { "type": "string", "default": "design/design.md" },
      "spec_path": { "type": "string", "default": "docs/spec-mcp-server.md" }
    },
    "required": []
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "in_sync": { "type": "boolean" },
      "mismatches": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "item": { "type": "string", "description": "不一致的端點或元件名稱" },
            "in_design": { "type": "string" },
            "in_spec": { "type": "string" },
            "suggestion": { "type": "string" }
          }
        }
      },
      "design_only": { "type": "array", "items": { "type": "string" }, "description": "只在 design 中出現的項目" },
      "spec_only": { "type": "array", "items": { "type": "string" }, "description": "只在 spec 中出現的項目" }
    }
  },
  "errors": ["FILE_NOT_FOUND", "PARSE_ERROR"]
}

3.4 安全性設定

所有 file_* 工具在 mcp-config.json 中設定允許路徑：

{
  "file_security": {
    "allowed_paths": [
      "./docs/",
      "./mcp-server/",
      "./design/",
      "./knowledge/",
      "./agent-config/",
      "./scripts/",
      "./src/",
      "./packages/",
      "./test/"
    ],
    "denied_patterns": [
      "**/.env",
      "*/.key",
      "*/.pem",
      "**/node_modules/",
      "**/.git/"
    ],
    "max_file_size_bytes": 1048576
  }
}

4. 架構決策記錄（ADR）

ADR-001：混合架構 — Golem Engine + Hermes Agent

狀態：已採納

背景：需要決定 Hermes Agent 的角色——完全取代 Golem Engine，還是作為補強。

備選方案：
A) 完全用 Hermes 取代 Golem — 簡化架構但失去 BitL 免費 LLM 與 Pyramid Memory 壓縮。
B) 混合架構：Golem 負責核心引擎（BitL、Memory、NeuroShunter），Hermes 負責任務排程與自學習迴圈 — 各取所長。
C) 只用 Golem 不加 Hermes — 缺少成熟的 Agent 框架（session 管理、cron、dashboard）。

決策：選擇 B。

理由：Golem BitL 提供免費 Gemini（$0），Pyramid Memory 壓縮比極高（~3 MB / 50 年），這兩個能力 Hermes 沒有。Hermes 的 Dashboard、Cron、Session 管理、150+ 設定項則是 Golem 缺乏的。透過 MCP 介面組合兩者，每個都只做自己最強的事。

風險與緩解：

風險 A — Browser-in-the-Loop 延遲與脆弱性：多層鏈路（MCP → Node.js → Playwright → 瀏覽器 → Google → 擷取結果）任何一環斷裂都導致失敗。緩解：free_llm_query 實作三層 fallback。第一層：Playwright 失敗時自動重試一次（含重新載入頁面）。第二層：切換到 Gemini API 免費額度（每分鐘 15 次、每日 1500 次）。第三層：切換到本地 Ollama。回應中附帶 provider 欄位標示實際使用的後端。

風險 B — Context window 溢出：同時載入對話、記憶、spec、研究報告可能超過模型限制。緩解：parse_protocol（NeuroShunter）內建 context budget 管理。每次呼叫 LLM 前計算各來源 token 數，按優先級分配（當前對話 > task_memory > spec 摘要 > research_knowledge），超過預算（模型 window 的 80%）時低優先級來源被截斷。memory_search 回傳 token_count 供 NeuroShunter 計算。

**風險 C —「clone 到可用   → 功能開發（如 feature/free-llm-query）
spec/     → 規格撰寫（如 spec/mcp-server）
hotfix/    → 緊急修復

6.2 Commit 訊息格式

(): 

type: feat | fix | docs | spec | design | refactor | test | ci | chore
scope: mcp-server | golem-engine | dashboard | memory | hermes | stitch | ci

範例：
feat(mcp-server): implement free_llm_query with 3-tier fallback
spec(mcp-server): add memory_store JSON schema
docs(constitution): v1.1 add namespace and sync_check
design(dashboard): initial Stitch-generated layout
ci: add GitHub Actions workflow

6.3 CI Pipeline

每次 push 或 PR 到 main / develop 時自動執行：

1. npm ci — 安裝依賴
2. npm run lint — 程式碼檢查
3. npm run test:unit — 單元測試
4. npm run test:integration — 整合測試（含 MCP Tool 呼叫）
5. npm run arch:check — 架構邊界檢查（infra/）
6. sync_check — 比對 design.md 與 spec 一致性（Phase 2 後啟用）

7. 非功能性需求（NFR）

| 項目 | 指標 |
|------|------|
| 啟動時間（Docker） | clone → docker compose up → 可用  80% |
| 長期記憶容量 | ≈ 3 MB / 50 年（Pyramid Memory 壓縮後） |
| 文件語言 | 繁體中文（主）+ 英文（README、程式碼註解） |
| 授權 | MIT |
| 月成本（免費方案） | $0 |

8. 路線圖

Phase 0 — 基礎建設
- [x] Golem 引擎拆解與移植
- [x] MCP 萬用接口基礎實作
- [x] GitHub repo 建立
- [x] constitution.md v1.0
- [x] system-flow.md
- [x] NotebookLM 架構審查
- [x] constitution.md v1.1（本次修正）
- [ ] 推送 v1.1 至 GitHub ← 目前此處

Phase 1 — 規格撰寫
- [ ] Spec Kit 初始化（specify init . --ai claude）
- [ ] docs/spec-mcp-server.md 撰寫
- [ ] 實作計畫與任務清單產出

Phase 2 — UI 設計
- [ ] Stitch Dashboard 設計
- [ ] design/design.md 匯出
- [ ] 視覺稿確認

Phase 3 — MCP Server Core 實作
- [ ] free_llm_query（含三層 fallback）
- [ ] memory_store / memory_search（含 namespace）
- [ ] memory_compress（按 namespace 分別壓縮）
- [ ] parse_protocol（含 context budget）
- [ ] file_read / file_write / file_list
- [ ] spec_read / spec_update
- [ ] sync_check
- [ ] 主入口 index.js
- [ ] Dockerfile 初版
- [ ] 單元測試 > 80%

Phase 4 — Google 工具鏈整合
- [ ] design_generate（Stitch SDK）
- [ ] free_notebooklm（CLI-Anything-Web）
- [ ] code_generate（Gemini API）

Phase 5 — Agent 增強
- [ ] Hermes Agent 接入 MCP Server
- [ ] Hermes Skill 定義（golem-engine/SKILL.md）
- [ ] 學習迴圈驗證（完成任務 → 記憶 → 召回 → 改善）

Phase 6 — 開源釋出
- [ ] docker-compose.yml 完善
- [ ] scripts/init.sh 一鍵啟動
- [ ] README.md 重寫（含快速入門）
- [ ] CONTRIBUTING.md
- [ ] .env.example
- [ ] GitHub Actions CI 完整配置

9. 成本矩陣

| 項目 | 免費方案 | 付費方案 |
|------|---------|---------|
| LLM（Gemini） | Browser-in-the-Loop $0 | Gemini API ~$7/M tokens |
| LLM（Claude） | — | Claude API ~$15/M tokens |
| LLM（本地） | Ollama $0 | — |
| UI 設計 | Stitch 免費額度 $0 | — |
| 知識庫 | NotebookLM 免費 $0 | NotebookLM Plus ~$20/月 |
| 程式碼託管 | GitHub Free $0 | GitHub Pro $4/月 |
| Agent | Hermes 免費 $0 | — |
| 雲端運行 | GitHub Codespaces 120h/月 $0 | VPS $5-20/月 |
| 合計 | $0/月 | $24-51/月 |

10. 文件層級與衝突規則

constitution.md（最高層級 — 設計原則與架構決策）
  ├── system-flow.md（系統流程與閉環說明）
  ├── spec-*.md（各模組規格，遵循 constitution 的 Tool Registry）
  ├── design.md（UI 設計，須通過 sync_check 與 spec 一致）
  ├── CLAUDE.md（Claude Code 的 context 規則，引用 constitution）
  ├── SKILL.md（Hermes 技能定義，引用 constitution 的 Tool 列表）
  └── README.md（面向外部使用者的快速入門）

當任何文件與 constitution.md 衝突時，以 constitution.md 為準。更新 constitution 需經 PR 審查並更新版本號。

11. 詞彙表

| 術語 | 說明 |
|------|------|
| MCP | Model Context Protocol — Agent 與工具間的通用通訊協議 |
| Browser-in-the-Loop（BitL） | 透過 Playwright 操控瀏覽器存取免費 Gemini |
| Pyramid Memory | 分層壓縮記憶系統（即時→日→週→月→年），含 namespace 分區 |
| NeuroShunter | 協議解析與 context budget 管理引擎 |
| Reflex Shunting | NeuroShunter 的快速路由機制，跳過不必要的處理步驟 |
| GOLEM_PROTOCOL | Golem 引擎內部通訊格式 |
| SDD | Spec-Driven Development — 先寫規格再寫程式碼 |
| Namespace | Pyramid Memory 的記憶分區（task_experience / research_knowledge / design_decision） |
| Context Budget | 每次 LLM 呼叫的 token 分配上限，由 NeuroShunter 管理 |
| sync_check | 自動比對 design.md 與 spec 文件一致性的 MCP Tool |
| 三層 Fallback | free_llm_query 的容錯機制：Browser → Gemini API → Ollama |

12. 參考連結

| 資源 | 連結 |
|------|------|
| Project Golem（原始引擎） | https://github.com/Arvincreator/project-golem |
| Project Golem（新框架） | https://github.com/RYN6666999/HMM |
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
- v1.1（2026-04-15）：依 NotebookLM 審查結果修正——新增 Pyramid Memory namespace 分區、sync_check 閉環同步機制、6 支新 MCP Tool（spec_read / spec_update / file_read / file_write / file_list / sync_check）、ADR-001 三項風險緩解措施、context budget 管理、Docker 啟動時間 NFR、檔案依賴路徑標註、文件層級衝突規則
