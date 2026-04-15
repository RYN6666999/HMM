
第三份：docs/spec-mcp-server.md v1.1

最後更新：2026-04-15 | 版本：v1.1（整合 weblm-driver）
對應：constitution.md v1.2、system-flow.md v1.2
倉庫：https://github.com/RYN6666999/HMM

1. 概述

1.1 本文件目的

定義 MCP Universal Server 中所有 15 支 Tool 的完整實作規格，作為 Phase 3-4 開發的 single source of truth。任何 Agent（Hermes、Claude Code、OpenCode）讀取本文件即可理解每支工具的行為、邊界與驗收標準。

1.2 適用範圍

本 spec 涵蓋 mcp-server/ 目錄下所有模組，包括主入口 index.js、Golem Engine 包裝層（golem-engine/）、獨立工具（tools/）與統一設定檔（mcp-config.json）。

1.3 v1.1 變更摘要

本版本的核心變更是將 free_llm_query 第一層 Browser-in-the-Loop 從「自行實作 Playwright 全流程」改為「委託 weblm-driver（super-engine repo）的 GeminiWebDriver」。這省去了瀏覽器生命週期管理、頁面狀態偵測、Prompt 提交、輸出擷取、恢復機制等全部底層工作（已有 134 個測試覆蓋），browser-loop.js 只需負責三層 fallback 調度與 MCP Tool 格式封裝。Task List 因此從 25 項減少到 23 項，Phase 3 預估從 28 小時降至 21 小時。

1.4 術語

所有術語定義見 constitution.md 第 11 章詞彙表。本文件額外定義：

- Tool：MCP 協議中的一個可呼叫函式，接受 JSON 輸入、回傳 JSON 輸出。
- 驗收標準（AC）：該工具被視為「完成」所必須通過的條件清單。
- Task：開發者可獨立執行的最小工作單位，每個 task 對應一次 commit。
- weblm-driver：super-engine repo 的 npm 套件，提供 GeminiWebDriver 類別。
- GeminiWebDriver：weblm-driver 的核心類別，實作 WebLLMDriver 介面（init / generate / health / recover / shutdown）。

2. 系統架構

2.1 MCP Server 內部結構

mcp-server/
├── index.js                  ← 主入口：註冊所有 Tool、啟動 JSON-RPC server
├── mcp-config.json           ← 統一設定：Tool 列表、安全規則、fallback 設定
├── package.json              ← 包含 weblm-driver dependency
├── golem-engine/
│   ├── browser-loop.js       ← free_llm_query（import weblm-driver + 三層 fallback）
│   ├── pyramid-memory.js     ← memory_store / memory_search / memory_compress
│   ├── neuro-shunter.js      ← parse_protocol（context budget）
│   └── security-manager.js   ← 路徑驗證、權限檢查
└── tools/
    ├── spec-read.js
    ├── spec-update.js
    ├── file-read.js
    ├── file-write.js
    ├── file-list.js
    ├── sync-check.js
    ├── free-notebooklm.js
    ├── design-generate.js
    ├── code-generate.js
    └── desktop-control.js

2.2 依賴關係

index.js
├── imports golem-engine/browser-loop.js
│   ├── requires weblm-driver (GeminiWebDriver, DriverError)  ← super-engine
│   ├── requires @google/generative-ai                         ← Gemini API fallback
│   └── requires ollama HTTP client                            ← Ollama fallback
├── imports golem-engine/pyramid-memory.js
│   └── requires lancedb
├── imports golem-engine/neuro-shunter.js
│   └── 自行實作
├── imports golem-engine/security-manager.js
│   └── 自行實作
├── imports tools/*.js
│   ├── spec-read.js / spec-update.js → requires fs + child_process (git)
│   ├── file-read.js / file-write.js / file-list.js → requires fs
│   │   └── all file ops → validated by security-manager.js
│   ├── sync-check.js → requires spec-read.js + file-read.js
│   ├── free-notebooklm.js → requires child_process (cli-web-notebooklm)
│   ├── design-generate.js → requires @google/stitch-sdk
│   ├── code-generate.js → requires @google/generative-ai
│   └── desktop-control.js → requires playwright
└── reads mcp-config.json

2.3 package.json 依賴

{
  "name": "golem-mcp-server",
  "version": "0.1.0",
  "dependencies": {
    "weblm-driver": "github:RYN6666999/super-engine#v0.1.5",
    "@google/generative-ai": "^0.21.0",
    "@google/stitch-sdk": "^0.1.0",
    "lancedb": "^0.4.0",
    "playwright": "^1.43.0"
  },
  "devDependencies": {
    "vitest": "^1.6.0",
    "@types/node": "^25.5.0",
    "typescript": "^5.4.0"
  }
}

注意：playwright 在 weblm-driver 中已是 dependency，此處再列是因為 desktop-control.js 也需要。npm 會自動 dedupe。

2.4 通訊協議

所有 Tool 遵循 MCP 規範（JSON-RPC 2.0 over stdio）。Agent 送出 tools/call 請求，Server 回傳 content 陣列。錯誤以 MCP 標準 error response 回傳。

3. 統一設定檔 — mcp-config.json

{
  "server": {
    "name": "golem-mcp-server",
    "version": "0.1.0",
    "description": "Project Golem MCP Universal Server"
  },
  "tools": {
    "enabled": [
      "free_llm_query", "memory_store", "memory_search",
      "memory_compress", "parse_protocol", "free_notebooklm",
      "spec_read", "spec_update", "file_read", "file_write",
      "file_list", "sync_check", "desktop_control",
      "design_generate", "code_generate"
    ]
  },
  "weblm_driver": {
    "provider_url": "https://gemini.google.com/app",
    "headless": true,
    "first_token_timeout_ms": 30000,
    "stability_timeout_ms": 120000,
    "stability_interval_ms": 1500,
    "args": ["--no-first-run", "--disable-session-crashed-bubble"]
  },
  "fallback": {
    "free_llm_query": {
      "order": ["browser", "api", "ollama"],
      "api": {
        "provider": "gemini",
        "model": "gemini-2.0-flash",
        "rate_limit_per_minute": 15,
        "rate_limit_per_day": 1500
      },
      "ollama": {
        "model": "llama3.2",
        "host": "http://localhost:11434",
        "timeout_ms": 15000
      }
    }
  },
  "memory": {
    "namespaces": ["task_experience", "research_knowledge", "design_decision"],
    "compression_schedule": {
      "day": "0 0 * * *",
      "week": "0 0 * * 0",
      "month": "0 0 1 * *",
      "year": "0 0 1 1 *"
    },
    "max_storage_mb": 100
  },
  "context_budget": {
    "default_budget_tokens": 100000,
    "budget_ratio": 0.8,
    "priority_order": ["conversation", "task_memory", "spec", "research_memory", "design"]
  },
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

4. P0 工具規格

4.1 free_llm_query

檔案：mcp-server/golem-engine/browser-loop.js

功能說明：透過三層 fallback 提供免費 LLM 查詢。第一層委託 weblm-driver 的 GeminiWebDriver（含四級恢復），第二層呼叫 Gemini API 免費額度，第三層呼叫本地 Ollama。browser-loop.js 不自己管理 Playwright，只做調度與格式轉換。

輸入 Schema：

{
  "type": "object",
  "properties": {
    "prompt": { "type": "string", "maxLength": 30000, "description": "查詢內容（必填）" },
    "context": { "type": "string", "maxLength": 50000, "description": "附加上下文（選填）" },
    "model": { "type": "string", "default": "gemini", "enum": ["gemini", "ollama"], "description": "偏好模型（選填）" }
  },
  "required": ["prompt"]
}

輸出 Schema：

{
  "type": "object",
  "properties": {
    "content": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["text"] },
          "text": { "type": "string" }
        }
      }
    },
    "provider": { "type": "string", "enum": ["browser", "api", "ollama"] },
    "latency_ms": { "type": "number" },
    "model_used": { "type": "string" },
    "output_kind": { "type": "string", "enum": ["normal", "provider-error", "unknown"], "description": "weblm-driver 的輸出分類（僅 provider=browser 時有意義）" },
    "fallback_attempts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "provider": { "type": "string" },
          "success": { "type": "boolean" },
          "error": { "type": "string" },
          "recovery_action": { "type": "string", "description": "weblm-driver 的恢復動作（僅 browser 層）" }
        }
      }
    },
    "driver_health": {
      "type": "object",
      "description": "weblm-driver 的健康快照（選填，定期回報用）",
      "properties": {
        "ok": { "type": "boolean" },
        "initialized": { "type": "boolean" },
        "browserRunning": { "type": "boolean" },
        "pageReady": { "type": "boolean" },
        "authenticated": { "type": "boolean" },
        "providerReachable": { "type": "boolean" },
        "mode": { "type": "string" }
      }
    }
  }
}

錯誤碼：

| 錯誤碼 | 觸發條件 | 處理方式 |
|--------|---------|---------|
| AUTH_EXPIRED | weblm-driver 拋出 AuthenticationRequiredError | 嘗試 recover → 重試 → 失敗則 fallback 到 api |
| CAPTCHA_DETECTED | weblm-driver health 顯示 challenge | recover → 重試 → fallback |
| BROWSER_CRASH | weblm-driver 拋出不可恢復的錯誤 | 直接 fallback 到 api |
| RATE_LIMITED | Gemini API 額度用盡 | fallback 到 ollama |
| NETWORK_ERROR | 網路不可用 | 若 ollama 可用則 fallback |
| ALL_FALLBACKS_FAILED | 三層全部失敗 | 回傳完整 fallback_attempts |
| PROMPT_TOO_LONG | prompt 超過 maxLength | 立即回傳錯誤 |

內部流程：

收到 MCP tools/call 請求
  │
  ├── prompt 長度檢查 → 超過 30000 → 回傳 PROMPT_TOO_LONG
  │
  ├── model = "ollama" → 直接跳到第三層
  │
  ├── 第一層：weblm-driver
  │   │
  │   ├── 組裝 GenerateInput
  │   │   prompt = input.prompt + input.context（合併）
  │   │   timeoutMs = 30000
  │   │   newConversation = false
  │   │
  │   ├── driver.generate(generateInput)
  │   │   ├── 成功
  │   │   │   ├── output.outputKind === "normal"
  │   │   │   │   → 轉換為 MCP 輸出格式
  │   │   │   │   → provider = "browser"
  │   │   │   │   → 回傳
  │   │   │   └── output.outputKind !== "normal"
  │   │   │       → 記錄為 fallback_attempt (success: false, error: outputKind)
  │   │   │       → 進入第二層
  │   │   │
  │   │   └── 拋出 DriverError
  │   │       ├── err.recoverable === true
  │   │       │   ├── driver.recover(err.code)
  │   │       │   │   ├── recovery.ok === true
  │   │       │   │   │   ├── driver.generate(generateInput)  ← 重試
  │   │       │   │   │   │   ├── 成功 + normal → 回傳 (provider: "browser")
  │   │       │   │   │   │   └── 失敗 → 記錄，進入第二層
  │   │       │   │   │   └──
  │   │       │   │   └── recovery.ok === false
  │   │       │   │       → 記錄，進入第二層
  │   │       │   └──
  │   │       └── err.recoverable === false
  │   │           → 記錄 (BROWSER_CRASH)，進入第二層
  │   │
  │   └── 記錄 fallback_attempt { provider: "browser", success, error, recovery_action }
  │
  ├── 第二層：Gemini API
  │   ├── 檢查 process.env.GEMINI_API_KEY
  │   │   └── 不存在 → 跳過，進入第三層
  │   ├── new GoogleGenerativeAI(apiKey)
  │   ├── model.generateContent(prompt)
  │   ├── timeout: 10s
  │   ├── 成功 → 轉換為 MCP 格式，provider = "api"，回傳
  │   └── 失敗 → 記錄 fallback_attempt，進入第三層
  │
  └── 第三層：Ollama
      ├── 檢查 Ollama 服務（GET http://localhost:11434/api/tags）
      │   └── 不可用 → ALL_FALLBACKS_FAILED
      ├── POST http://localhost:11434/api/generate
      │   body: { model, prompt, stream: false }
      ├── timeout: 15s
      ├── 成功 → 轉換為 MCP 格式，provider = "ollama"，回傳
      └── 失敗 → ALL_FALLBACKS_FAILED + 完整 fallback_attempts

browser-loop.js 的 weblm-driver 生命週期管理：

MCP Server 啟動 (index.js)
  │
  ├── 建立 GeminiWebDriver 實例
  │   const driver = new GeminiWebDriver({
  │     providerUrl: config.weblm_driver.provider_url,
  │     profileDir: process.env.SMOKE_PROFILE_DIR,
  │     executablePath: process.env.CHROME_EXECUTABLE,  // 選填
  │     headless: config.weblm_driver.headless,
  │     firstTokenTimeoutMs: config.weblm_driver.first_token_timeout_ms,
  │     stabilityTimeoutMs: config.weblm_driver.stability_timeout_ms,
  │     stabilityIntervalMs: config.weblm_driver.stability_interval_ms,
  │     args: config.weblm_driver.args,
  │   });
  │
  ├── driver.init()
  │   └── 失敗（AuthenticationRequiredError）→ 記錄 warning，browser 層標記為不可用
  │
  ├── 每次 free_llm_query 呼叫 → driver.generate() / recover()
  │
  ├── 定期（每 60 秒）→ driver.health() → 記錄到 MCP Server 日誌
  │
  └── MCP Server 關閉 → driver.shutdown()

呼叫範例：

請求：
{
  "method": "tools/call",
  "params": {
    "name": "free_llm_query",
    "arguments": {
      "prompt": "解釋 MCP 協議的核心概念",
      "context": "使用者是初學者，需要簡單易懂的說明"
    }
  }
}

成功回應（weblm-driver 層）：
{
  "content": [{ "type": "text", "text": "MCP（Model Context Protocol）是一個開放標準..." }],
  "provider": "browser",
  "latency_ms": 8432,
  "model_used": "gemini-web",
  "output_kind": "normal",
  "fallback_attempts": [
    { "provider": "browser", "success": true, "error": null, "recovery_action": null }
  ]
}

成功回應（經 fallback 到 API 層）：
{
  "content": [{ "type": "text", "text": "MCP 是一個通用協議..." }],
  "provider": "api",
  "latency_ms": 2105,
  "model_used": "gemini-2.0-flash",
  "output_kind": null,
  "fallback_attempts": [
    { "provider": "browser", "success": false, "error": "AUTH_EXPIRED", "recovery_action": "refresh-page" },
    { "provider": "browser", "success": false, "error": "AUTH_EXPIRED (retry after recover)", "recovery_action": null },
    { "provider": "api", "success": true, "error": null, "recovery_action": null }
  ]
}

驗收標準（AC）：

- AC-1：weblm-driver 正常時，回傳 provider: "browser"、output_kind: "normal"、content[0].text 非空。
- AC-2：weblm-driver generate() 拋出 recoverable error → 呼叫 recover() → 恢復成功後重試 → 成功回傳 provider: "browser"。
- AC-3：weblm-driver outputKind 為 provider-error → 視為失敗 → fallback 到 API 層。
- AC-4：weblm-driver 恢復失敗 → fallback 到 API 層，回傳 provider: "api"。
- AC-5：weblm-driver 拋出不可恢復錯誤（recoverable === false）→ 不嘗試 recover → 直接 fallback。
- AC-6：API 層也失敗 → fallback 到 Ollama，回傳 provider: "ollama"。
- AC-7：三層全部失敗 → 回傳 ALL_FALLBACKS_FAILED + 完整 fallback_attempts。
- AC-8：prompt 超過 30000 字元 → 立即回傳 PROMPT_TOO_LONG。
- AC-9：model: "ollama" → 跳過前兩層。
- AC-10：fallback_attempts 中 browser 層的記錄包含 recovery_action（如 "refresh-page"）。
- AC-11：MCP Server 啟動時呼叫 driver.init()，失敗不導致 Server crash，而是標記 browser 層不可用。
- AC-12：MCP Server 關閉時呼叫 driver.shutdown()，確保瀏覽器正常關閉。

4.2 memory_store

檔案：mcp-server/golem-engine/pyramid-memory.js

功能說明：將記憶存入 Pyramid Memory，按 namespace 分區儲存。每筆記憶附帶自動生成的向量嵌入與 token 計數。

輸入 Schema：

{
  "type": "object",
  "properties": {
    "content": { "type": "string", "maxLength": 10000, "description": "記憶內容（必填）" },
    "namespace": { "type": "string", "enum": ["task_experience", "research_knowledge", "design_decision"], "description": "記憶分區（必填）" },
    "tags": { "type": "array", "items": { "type": "string", "maxLength": 50 }, "maxItems": 10, "description": "標籤（選填）" },
    "metadata": {
      "type": "object",
      "properties": {
        "source_tool": { "type": "string" },
        "task_id": { "type": "string" },
        "importance": { "type": "number", "minimum": 0, "maximum": 1, "default": 0.5 }
      },
      "description": "附加 metadata（選填）"
    }
  },
  "required": ["content", "namespace"]
}

輸出 Schema：

{
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "UUID v4" },
    "stored_at": { "type": "string", "format": "date-time" },
    "namespace": { "type": "string" },
    "token_count": { "type": "number" },
    "compression_level": { "type": "string", "enum": ["instant"] },
    "vector_dimensions": { "type": "number" }
  }
}

錯誤碼：INVALID_NAMESPACE、CONTENT_TOO_LONG、STORAGE_FULL（觸發自動壓縮後重試）、WRITE_ERROR、EMBEDDING_FAILED。

內部流程：驗證 namespace → 驗證 content 長度 → 計算 token_count → 生成向量嵌入（優先 Gemini embedding API，fallback 本地 sentence-transformers）→ 組裝記憶物件（id: UUID v4, compression_level: "instant"）→ 寫入 LanceDB（table: memory_{namespace}）→ STORAGE_FULL 時觸發 memory_compress 後重試。

驗收標準：
- AC-1：有效 content + namespace → 回傳含 UUID id、stored_at、正確 namespace。
- AC-2：namespace 不在 enum → INVALID_NAMESPACE。
- AC-3：content 超過 10000 字元 → CONTENT_TOO_LONG。
- AC-4：存入後 memory_search 以語意相近 query 可找到。
- AC-5：不同 namespace 儲存在不同 LanceDB table。
- AC-6：token_count 為正整數。
- AC-7：儲存空間超限時自動壓縮後重試。

4.3 memory_search

檔案：mcp-server/golem-engine/pyramid-memory.js

功能說明：透過語意搜尋查詢 Pyramid Memory，支援 namespace 過濾。回傳 token_count 供 NeuroShunter 計算 context budget。

輸入 Schema：

{
  "type": "object",
  "properties": {
    "query": { "type": "string", "maxLength": 5000 },
    "namespace": { "type": "string", "enum": ["task_experience", "research_knowledge", "design_decision", "all"], "default": "all" },
    "top_k": { "type": "number", "default": 5, "minimum": 1, "maximum": 20 },
    "min_relevance": { "type": "number", "default": 0.7, "minimum": 0, "maximum": 1 },
    "include_compressed": { "type": "boolean", "default": true }
  },
  "required": ["query"]
}

輸出 Schema：

{
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
          "tags": { "type": "array", "items": { "type": "string" } },
          "compression_level": { "type": "string", "enum": ["instant", "day", "week", "month", "year"] },
          "created_at": { "type": "string", "format": "date-time" },
          "metadata": { "type": "object" }
        }
      }
    },
    "total_tokens": { "type": "number" },
    "searched_namespaces": { "type": "array", "items": { "type": "string" } },
    "total_matches": { "type": "number" }
  }
}

錯誤碼：QUERY_TOO_LONG、INDEX_CORRUPTED（嘗試重建）、READ_ERROR、EMBEDDING_FAILED。

內部流程：驗證 query 長度 → 生成 query 向量嵌入 → 決定搜尋範圍（all = 三個 table，指定 = 一個）→ 各 table 向量相似度搜尋 → 篩選 relevance ≥ min_relevance → include_compressed 過濾 → 合併排序取 top_k → 計算 total_tokens。

驗收標準：
- AC-1：存入後以語意相近 query 搜尋，relevance > 0.7。
- AC-2：指定 namespace 時不回傳其他 namespace 記憶。
- AC-3：namespace = "all" 時跨三個 table 搜尋。
- AC-4：min_relevance = 0.95 時可能回傳空陣列。
- AC-5：total_tokens = sum(result.token_count)。
- AC-6：結果按 relevance 降序。
- AC-7：延遲  1 MB → FILE_TOO_LARGE。

5.7 file_write

檔案：mcp-server/tools/file-write.js

輸入：path（必填）、content（必填）、create_dirs（預設 true）、overwrite（預設 false）。

輸出：written、size_bytes、path。

錯誤碼：PERMISSION_DENIED、PATH_NOT_ALLOWED、FILE_EXISTS、WRITE_ERROR。

驗收標準：AC-1 新檔案 written: true。AC-2 overwrite:false + 已存在 → FILE_EXISTS。AC-3 create_dirs 自動建立。AC-4 .git/ → PATH_NOT_ALLOWED。

5.8 file_list

檔案：mcp-server/tools/file-list.js

輸入：path（預設 "."）、recursive（預設 false）、pattern（glob，選填）。

輸出：entries（name/type/size_bytes/modified_at）、total_count。

錯誤碼：DIRECTORY_NOT_FOUND、PERMISSION_DENIED、PATH_NOT_ALLOWED。

驗收標準：AC-1 docs/ 含 constitution.md。AC-2 recursive 含子目錄。AC-3 *.md 只回傳 markdown。AC-4 node_modules/ → PATH_NOT_ALLOWED。

5.9 sync_check

檔案：mcp-server/tools/sync-check.js

輸入：design_path（預設 "design/design.md"）、spec_path（預設 "docs/spec-mcp-server.md"）。

輸出：in_sync、mismatches（item/in_design/in_spec/suggestion）、design_only、spec_only。

錯誤碼：FILE_NOT_FOUND、PARSE_ERROR。

驗收標準：AC-1 命名不匹配回報 mismatch。AC-2 未定義頁面出現在 design_only。AC-3 完全一致時 in_sync: true。

6. P2 工具規格

6.1 design_generate

檔案：mcp-server/tools/design-generate.js

輸入：prompt（必填）、project_id（選填）、design_system（選填）。

輸出：screen_id、html_url、image_url、project_id。

錯誤碼：AUTH_FAILED、STITCH_ERROR、QUOTA_EXCEEDED。

驗收標準：AC-1 有效 prompt 回傳 html_url 和 image_url。AC-2 無 API Key → AUTH_FAILED。AC-3 指定 project_id 在既有專案新增 screen。

6.2 code_generate

檔案：mcp-server/tools/code-generate.js

輸入：prompt（必填）、language（預設 "javascript"）、spec_context（選填）、provider（"gemini"/"opencode"，預設 "gemini"）。

輸出：code、language、explanation、provider。

錯誤碼：API_ERROR、PROVIDER_UNAVAILABLE、INVALID_LANGUAGE。

驗收標準：AC-1 回傳有效程式碼。AC-2 含 spec_context 時符合 I/O 定義。AC-3 provider 不可用 → PROVIDER_UNAVAILABLE。

6.3 desktop_control

檔案：mcp-server/tools/desktop-control.js

輸入：action（navigate/click/type/screenshot/evaluate，必填）、target（URL 或 selector）、value（type 動作的輸入）。

輸出：success、screenshot_url（screenshot 時）、result（evaluate 時）。

錯誤碼：BROWSER_LAUNCH_FAILED、ELEMENT_NOT_FOUND、NAVIGATION_TIMEOUT、ACTION_FAILED。

注意：desktop_control 使用自己的 Playwright 實例，與 weblm-driver 的瀏覽器完全獨立，避免干擾 LLM 會話。

驗收標準：AC-1 navigate 成功。AC-2 screenshot 回傳有效路徑。AC-3 不存在 selector → ELEMENT_NOT_FOUND。

7. 主入口規格 — index.js

檔案：mcp-server/index.js

啟動流程：

node mcp-server/index.js
  │
  ├── 讀取 mcp-config.json
  │
  ├── 初始化共用元件
  │   ├── SecurityManager
  │   ├── PyramidMemory (LanceDB 連接，確認三個 namespace table)
  │   ├── NeuroShunter
  │   └── GeminiWebDriver (weblm-driver)
  │       ├── driver.init()
  │       └── 失敗 → warning log，browser 層標記不可用（不 crash）
  │
  ├── 註冊 Tool handlers
  │   └── 對每個 enabled tool：import handler → 註冊 name + inputSchema + handler
  │
  ├── 啟動定期健康檢查
  │   └── setInterval(() => driver.health(), 60000)
  │
  ├── 啟動 MCP Server (JSON-RPC over stdio)
  │
  ├── 註冊 shutdown hook
  │   └── process.on('SIGTERM', async () => { await driver.shutdown(); process.exit(0); })
  │
  └── 就緒 log: "Golem MCP Server started with N tools (browser: ok/degraded)"

驗收標準：
- AC-1：無錯誤啟動。
- AC-2：tools/list 回傳正確數量。
- AC-3：tools/call 呼叫已註冊 tool 回傳有效回應。
- AC-4：呼叫未註冊 tool 回傳 MCP error。
- AC-5：mcp-config.json 語法錯誤時 log 並 exit(1)。
- AC-6：記憶體 .test.js`

9.2 整合測試

啟動 index.js → stdin 送 JSON-RPC → 驗證 stdout。測試跨工具場景：memory_store → memory_search → memory_compress → memory_search。測試 weblm-driver 降級場景：init 失敗 → Server 仍啟動 → free_llm_query fallback 到 API/Ollama。

9.3 覆蓋率目標

golem-engine/ > 80%（browser-loop.js 的 fallback 邏輯需 100% 路徑覆蓋）。tools/ > 70%。index.js > 60%。weblm-driver 自帶 134 個測試另計。

10. 環境變數

.env.example

=== weblm-driver（Browser-in-the-Loop）===
SMOKE_PROFILE_DIR=               # Chrome profile 目錄（啟用 browser 層必填）
CHROME_EXECUTABLE=               # Chrome 執行檔路徑（選填，profile 不相容時需要）

=== Gemini API（第二層 fallback）===
GEMINI_API_KEY=                  # Gemini API Key（選填，啟用 API fallback）

=== Stitch SDK ===
STITCH_API_KEY=                  # Stitch API Key（選填，啟用 design_generate）
STITCH_ACCESS_TOKEN=             # Stitch OAuth token（與 API Key 二選一）
GOOGLE_CLOUD_PROJECT=            # Google Cloud 專案 ID（選填）

=== Ollama（第三層 fallback）===
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=llama3.2

=== 開發 ===
LOG_LEVEL=info                   # debug | info | warn | error
MCP_CONFIG_PATH=./mcp-config.json
NODE_ENV=development

11. 部署設定

11.1 開發環境啟動

cd mcp-server
npm install          # 自動安裝 weblm-driver from GitHub
cp .env.example .env # 填入 SMOKE_PROFILE_DIR
node index.js

11.2 作為 Hermes Agent 的 MCP Server

~/.hermes/config.yaml
mcp:
  servers:
    golem-engine:
      command: "node"
      args: ["/mcp-server/index.js"]
      description: "Golem Engine MCP Server — weblm-driver + memory + tools"

11.3 作為 Claude Code 的 MCP Server

{
  "golem-engine": {
    "command": "node",
    "args": ["./mcp-server/index.js"],
    "description": "Golem Engine MCP Server"
  }
}

11.4 Docker

FROM node:20-slim
WORKDIR /app
COPY mcp-server/ ./mcp-server/
COPY packages/ ./packages/
RUN cd mcp-server && npm ci --production
weblm-driver 的 playwright dependency
RUN npx playwright install --with-deps chromium
CMD ["node", "mcp-server/index.js"]

12. 驗收總表

| # | 標準 | 驗證方式 |
|---|------|---------|
| 1 | node mcp-server/index.js 無錯誤啟動 | 手動執行 |
| 2 | tools/list 回傳 15 支工具 | JSON-RPC |
| 3 | free_llm_query weblm-driver 層可用 | 整合測試 |
| 4 | free_llm_query fallback 到 API 可用 | 單元測試 + mock |
| 5 | free_llm_query fallback 到 Ollama 可用 | 單元測試 + mock |
| 6 | free_llm_query ALL_FALLBACKS_FAILED 正確回報 | 單元測試 |
| 7 | weblm-driver init 失敗 → Server 仍啟動 | 整合測試 |
| 8 | memory_store → memory_search 端對端可用 | 整合測試 |
| 9 | memory_compress 不混合 namespace | 單元測試 |
| 10 | parse_protocol context budget 裁剪正確 | 單元測試 |
| 11 | file_* 安全性驗證（路徑穿越） | 單元測試 |
| 12 | spec_read / spec_update 可操作本文件 | 整合測試 |
| 13 | sync_check 偵測不一致 | 單元測試 |
| 14 | 核心模組測試覆蓋率 > 80% | vitest --coverage |
| 15 | 記憶體  版本紀錄
- v1.0（2026-04-15）：初版——15 支 Tool 完整規格、25 項任務、測試策略
- v1.1（2026-04-15）：整合 weblm-driver——free_llm_query 第一層改為委託 GeminiWebDriver（四級恢復 + outputKind 檢查）、browser-loop.js 只負責 fallback 調度與 MCP 封裝、新增 weblm_driver 設定區塊於 mcp-config.json、新增 driver 生命週期管理於 index.js（init/health/shutdown）、Task List 從 25→23 項（刪除重複 Playwright 實作）、Phase 3 預估從 28→21 小時、新增 weblm-driver 節省明細表、新增驗收項 #7 #18 #19、新增 package.json 依賴定義、.env.example 新增 SMOKE_PROFILE_DIR 和 CHROME_EXECUTABLE
