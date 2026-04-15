
docs/spec-mcp-server.md v1.2

最後更新：2026-04-15 | 版本：v1.2（混合借力——FastMCP + 官方 Filesystem Server）
對應：constitution.md v1.3、system-flow.md v1.3
倉庫：https://github.com/RYN6666999/HMM

1. 概述

1.1 本文件目的

定義 MCP Universal Server 中所有 15 支 Tool 的完整實作規格，作為 Phase 3-4 開發的 single source of truth。任何 Agent（Hermes、Claude Code、OpenCode）讀取本文件即可理解每支工具的行為、邊界與驗收標準。

1.2 適用範圍

本 spec 涵蓋 mcp-server/ 目錄下所有模組，包括主入口 index.ts（基於 FastMCP 框架）、Golem Engine 自建層（golem-engine/）、薄 wrapper 工具（tools/）、統一設定檔（mcp-config.json），以及借力的外部套件（fastmcp、@modelcontextprotocol/server-filesystem、weblm-driver）。

1.3 v1.2 變更摘要

本版本的核心變更：

1. MCP Server 框架遷移：從手寫 index.js JSON-RPC 路由改為使用 FastMCP 框架（npm install fastmcp）。index.ts 只需呼叫 new FastMCP() + server.addTool() 註冊每支 Tool handler，框架自動處理 JSON-RPC、Session、Health、Error 格式化。
2. 檔案系統工具借力：file_read / file_write / file_list 三支自建工具改為引入 @modelcontextprotocol/server-filesystem（Anthropic 官方）。官方套件提供 13 個檔案操作工具，含 allowed-path sandbox。spec 中的工具名稱更新為官方名稱（read_text_file、write_file、list_directory）。
3. Pyramid Memory 保留自建：LanceDB + 分層壓縮 + namespace 分區為核心差異化，不替換為 Mem0。
4. Task List 更新：從 23 項減少到 18 項，Phase 3 預估從 21 小時降至 ~14 小時。

1.4 術語

所有術語定義見 constitution.md v1.3 第 11 章詞彙表。本文件額外定義：

- Tool：MCP 協議中的一個可呼叫函式，接受 JSON 輸入、回傳 JSON 輸出。
- 驗收標準（AC）：該工具被視為「完成」所必須通過的條件清單。
- Task：開發者可獨立執行的最小工作單位，每個 task 對應一次 commit。
- FastMCP：TypeScript MCP server 框架（npm: fastmcp），提供 Tool 註冊、JSON-RPC、Session、Health 等基建。
- @modelcontextprotocol/server-filesystem：Anthropic 官方 MCP filesystem server，提供 13 個檔案操作工具。
- weblm-driver：super-engine repo 的 npm 套件，提供 GeminiWebDriver 類別。
- GeminiWebDriver：weblm-driver 的核心類別，實作 WebLLMDriver 介面（init / generate / health / recover / shutdown）。

2. 系統架構

2.1 MCP Server 內部結構
mcp-server/
├── index.ts                  ← 主入口：FastMCP 初始化 + 註冊所有 Tool
├── mcp-config.json           ← 統一設定：Tool 列表、安全規則、fallback 設定
├── package.json              ← 含 fastmcp、weblm-driver、filesystem server 依賴
├── tsconfig.json
├── golem-engine/
│   ├── browser-loop.ts       ← free_llm_query（import weblm-driver + 三層 fallback）
│   ├── pyramid-memory.ts     ← memory_store / memory_search / memory_compress（LanceDB 自建）
│   ├── neuro-shunter.ts      ← parse_protocol（context budget）
│   └── security-manager.ts   ← denied-pattern 補充驗證（.env/.key/.pem/.git）
└── tools/
    ├── spec-read.ts           ← 委託 filesystem server 讀檔 + section 解析
    ├── spec-update.ts         ← 委託 filesystem server 寫檔 + git commit
    ├── sync-check.ts          ← 純邏輯比對
    ├── free-notebooklm.ts
    ├── design-generate.ts
    ├── code-generate.ts
    └── desktop-control.ts
注意：v1.2 起不再有 file-read.ts、file-write.ts、file-list.ts — 由 @modelcontextprotocol/server-filesystem 直接提供。

2.2 依賴關係
index.ts (FastMCP)
├── imports fastmcp (FastMCP, UserError, z)                     ← MCP 框架
├── imports golem-engine/browser-loop.ts
│   ├── requires weblm-driver (GeminiWebDriver, DriverError)    ← super-engine
│   ├── requires @google/generative-ai                          ← Gemini API fallback
│   └── requires ollama HTTP client                             ← Ollama fallback
├── imports golem-engine/pyramid-memory.ts
│   └── requires lancedb                                        ← 自建核心
├── imports golem-engine/neuro-shunter.ts
│   └── 自行實作（參考 SimpleMem 演算法）
├── imports golem-engine/security-manager.ts
│   └── 自行實作
├── imports tools/*.ts
│   ├── spec-read.ts / spec-update.ts → 委託 filesystem server + child_process (git)
│   ├── sync-check.ts → requires spec-read.ts
│   ├── free-notebooklm.ts → requires child_process (cli-web-notebooklm)
│   ├── design-generate.ts → requires @google/stitch-sdk
│   ├── code-generate.ts → requires @google/generative-ai
│   └── desktop-control.ts → requires playwright
├── spawns @modelcontextprotocol/server-filesystem               ← 官方 filesystem
│   └── 啟動參數：allowed paths（./docs、./mcp-server 等）
│   └── 提供 13 個檔案工具（read_text_file、write_file、list_directory 等）
└── reads mcp-config.json
2.3 package.json 依賴json
{
  "name": "golem-mcp-server",
  "version": "0.2.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "npx fastmcp dev src/index.ts",
    "inspect": "npx fastmcp inspect src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint src/"
  },
  "dependencies": {
    "fastmcp": "^1.0.0",
    "weblm-driver": "github:RYN6666999/super-engine#v0.1.5",
    "@modelcontextprotocol/server-filesystem": "^2.0.0",
    "@google/generative-ai": "^0.21.0",
    "@google/stitch-sdk": "^0.1.0",
    "lancedb": "^0.4.0",
    "playwright": "^1.43.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "vitest": "^1.6.0",
    "@types/node": "^25.5.0",
    "typescript": "^5.4.0",
    "@typescript-eslint/eslint-plugin": "^7.8.0",
    "@typescript-eslint/parser": "^7.8.0",
    "eslint": "^8.57.0"
  }
}
注意：playwright 在 weblm-driver 中已是 dependency，此處再列是因為 desktop-control.ts 也需要。npm 會自動 dedupe。zod 為 FastMCP 的 Tool schema 定義所需。

2.4 通訊協議

所有 Tool 遵循 MCP 規範（JSON-RPC 2.0 over stdio），由 FastMCP 框架自動處理。Agent 送出 tools/call 請求，FastMCP 路由到對應 handler，回傳 content 陣列。錯誤以 FastMCP 的 UserError 拋出，框架自動轉為 MCP 標準 error response。

3. 統一設定檔 — mcp-config.jsonjson
{
  "server": {
    "name": "golem-mcp-server",
    "version": "0.2.0",
    "description": "Project Golem MCP Universal Server (FastMCP + Filesystem Server)"
  },
  "tools": {
    "enabled": [
      "free_llm_query", "memory_store", "memory_search",
      "memory_compress", "parse_protocol", "free_notebooklm",
      "spec_read", "spec_update", "sync_check",
      "desktop_control", "design_generate", "code_generate"
    ],
    "filesystem_server": {
      "enabled": true,
      "allowed_paths": [
        "./docs/", "./mcp-server/", "./design/", "./knowledge/",
        "./agent-config/", "./scripts/", "./src/", "./packages/", "./test/"
      ],
      "note": "filesystem server 提供 read_text_file、write_file、list_directory 等 13 個工具"
    }
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
    "denied_patterns": [
      "/.env", "/.key", "/.pem", "/node_modules/", "/.git/"
    ],
    "max_file_size_bytes": 1048576,
    "note": "allowed_paths 由 filesystem server 的命令列參數控制；denied_patterns 由 security-manager.ts 在 wrapper 層補充"
  }
}
4. P0 工具規格

4.1 free_llm_query

檔案：mcp-server/golem-engine/browser-loop.ts

功能說明：透過三層 fallback 提供免費 LLM 查詢。第一層委託 weblm-driver 的 GeminiWebDriver（含四級恢復），第二層呼叫 Gemini API 免費額度，第三層呼叫本地 Ollama。browser-loop.ts 不自己管理 Playwright，只做調度與格式轉換。

FastMCP 註冊方式：typescript
import { z } from "zod";

server.addTool({
  name: "free_llm_query",
  description: "免費 LLM 查詢（三層 fallback：weblm-driver → Gemini API → Ollama）",
  parameters: z.object({
    prompt: z.string().max(30000).describe("查詢內容（必填）"),
    context: z.string().max(50000).optional().describe("附加上下文（選填）"),
    model: z.enum(["gemini", "ollama"]).default("gemini").optional().describe("偏好模型（選填）"),
  }),
  execute: async (args, { log, reportProgress }) => {
    return await browserLoop.query(args, { log, reportProgress });
  },
});
輸出 Schema：json
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
| ALL_FALLBACKS_FAILED | 三層全部失敗 | 回傳完整 fallback_attempts，拋出 UserError |
| PROMPT_TOO_LONG | prompt 超過 maxLength | 立即拋出 UserError |

內部流程：
收到 FastMCP tools/call 請求（args 已由 zod 驗證）
  │
  ├── prompt 長度檢查 → 超過 30000 → throw new UserError("PROMPT_TOO_LONG")
  │
  ├── model = "ollama" → 直接跳到第三層
  │
  ├── 第一層：weblm-driver
  │   ├── 組裝 GenerateInput { prompt + context, timeoutMs: 30000 }
  │   ├── driver.generate(generateInput)
  │   │   ├── 成功 + outputKind === "normal" → 回傳 provider: "browser"
  │   │   ├── 成功 + outputKind !== "normal" → 記錄，進入第二層
  │   │   └── 拋出 DriverError
  │   │       ├── recoverable → driver.recover() → 重試 → 成功/失敗
  │   │       └── 不可恢復 → BROWSER_CRASH，進入第二層
  │   └── 記錄 fallback_attempt
  │
  ├── 第二層：Gemini API
  │   ├── 檢查 GEMINI_API_KEY → 不存在則跳過
  │   ├── model.generateContent(prompt)，timeout 10s
  │   ├── 成功 → provider: "api"
  │   └── 失敗 → 記錄，進入第三層
  │
  └── 第三層：Ollama
      ├── GET http://localhost:11434/api/tags → 不可用則 ALL_FALLBACKS_FAILED
      ├── POST /api/generate，timeout 15s
      ├── 成功 → provider: "ollama"
      └── 失敗 → throw new UserError("ALL_FALLBACKS_FAILED")
browser-loop.ts 的 weblm-driver 生命週期管理：
FastMCP Server 啟動 (index.ts)
  │
  ├── 建立 GeminiWebDriver 實例（從 mcp-config.json 讀取設定）
  ├── driver.init()
  │   └── 失敗（AuthenticationRequiredError）→ log.warn()，browser 層標記不可用
  │
  ├── 每次 free_llm_query 呼叫 → driver.generate() / recover()
  │
  ├── 定期（每 60 秒）→ driver.health() → log.info() 記錄到 FastMCP 日誌
  │
  └── FastMCP shutdown → driver.shutdown()
呼叫範例：

請求：json
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
成功回應（weblm-driver 層）：json
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
成功回應（經 fallback 到 API 層）：json
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
- AC-7：三層全部失敗 → 拋出 UserError("ALL_FALLBACKS_FAILED") + 完整 fallback_attempts。
- AC-8：prompt 超過 30000 字元 → 立即拋出 UserError("PROMPT_TOO_LONG")。
- AC-9：model: "ollama" → 跳過前兩層。
- AC-10：fallback_attempts 中 browser 層的記錄包含 recovery_action（如 "refresh-page"）。
- AC-11：FastMCP Server 啟動時呼叫 driver.init()，失敗不導致 Server crash，而是標記 browser 層不可用。
- AC-12：FastMCP Server 關閉時呼叫 driver.shutdown()，確保瀏覽器正常關閉。

4.2 memory_store

檔案：mcp-server/golem-engine/pyramid-memory.ts

功能說明：將記憶存入 Pyramid Memory（LanceDB 自建），按 namespace 分區儲存。每筆記憶附帶自動生成的向量嵌入與 token 計數。

FastMCP 註冊方式：typescript
server.addTool({
  name: "memory_store",
  description: "儲存記憶至 Pyramid Memory（含 namespace 分區、向量嵌入、token 計數）",
  parameters: z.object({
    content: z.string().max(10000).describe("記憶內容（必填）"),
    namespace: z.enum(["task_experience", "research_knowledge", "design_decision"]).describe("記憶分區（必填）"),
    tags: z.array(z.string().max(50)).max(10).optional().describe("標籤（選填）"),
    metadata: z.object({
      source_tool: z.string().optional(),
      task_id: z.string().optional(),
      importance: z.number().min(0).max(1).default(0.5).optional(),
    }).optional().describe("附加 metadata（選填）"),
  }),
  execute: async (args) => {
    return await pyramidMemory.store(args);
  },
});
輸出 Schema：json
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
錯誤碼：

| 錯誤碼 | 觸發條件 | 處理方式 |
|--------|---------|---------|
| INVALID_NAMESPACE | namespace 不在 enum 中 | 拋出 UserError（zod 已擋，此為防禦性檢查） |
| CONTENT_TOO_LONG | content 超過 10000 字元 | 拋出 UserError（zod 已擋） |
| STORAGE_FULL | LanceDB 儲存空間超限 | 觸發自動 memory_compress 後重試一次 |
| WRITE_ERROR | LanceDB 寫入失敗 | 拋出 UserError |
| EMBEDDING_FAILED | 向量嵌入生成失敗 | 拋出 UserError |

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

檔案：mcp-server/golem-engine/pyramid-memory.ts

功能說明：透過語意搜尋查詢 Pyramid Memory（LanceDB 自建），支援 namespace 過濾。回傳 token_count 供 NeuroShunter 計算 context budget。

FastMCP 註冊方式：typescript
server.addTool({
  name: "memory_search",
  description: "搜尋 Pyramid Memory（語意搜尋、namespace 過濾、token_count 回傳）",
  parameters: z.object({
    query: z.string().max(5000).describe("搜尋查詢（必填）"),
    namespace: z.enum(["task_experience", "research_knowledge", "design_decision", "all"]).default("all").optional(),
    top_k: z.number().min(1).max(20).default(5).optional(),
    min_relevance: z.number().min(0).max(1).default(0.7).optional(),
    include_compressed: z.boolean().default(true).optional(),
  }),
  execute: async (args) => {
    return await pyramidMemory.search(args);
  },
});
輸出 Schema：json
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
- AC-7：延遲  {
    return await pyramidMemory.compress(args);
  },
});
輸出 Schema：json
{
  "type": "object",
  "properties": {
    "compressed_count": { "type": "number" },
    "original_tokens": { "type": "number" },
    "compressed_tokens": { "type": "number" },
    "compression_ratio": { "type": "number" },
    "namespaces_processed": { "type": "array", "items": { "type": "string" } }
  }
}
錯誤碼：NOTHING_TO_COMPRESS、COMPRESSION_FAILED、INVALID_NAMESPACE、LLM_UNAVAILABLE（壓縮需呼叫 LLM 做摘要）。

驗收標準：

- AC-1：壓縮後 compressed_tokens  {
    return await neuroShunter.parse(args);
  },
});
輸出 Schema：json
{
  "type": "object",
  "properties": {
    "intent": { "type": "string" },
    "suggested_tool": { "type": "string" },
    "trimmed_context": { "type": "string" },
    "budget_usage": {
      "type": "object",
      "properties": {
        "total_tokens": { "type": "number" },
        "budget_tokens": { "type": "number" },
        "sources": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": { "type": "string" },
              "tokens": { "type": "number" },
              "trimmed": { "type": "boolean" }
            }
          }
        }
      }
    }
  }
}
錯誤碼：PARSE_FAILED、CONTEXT_OVERFLOW、UNKNOWN_PROTOCOL。

驗收標準：

- AC-1：有效輸入 → 回傳 intent + suggested_tool。
- AC-2：context 超過 budget → 低優先級來源被截斷，budget_usage 標記 trimmed: true。
- AC-3：優先級順序：conversation > task_memory > spec > research_memory > design。
- AC-4：budget_usage.total_tokens ≤ budget_tokens。

5.3 free_notebooklm

檔案：mcp-server/tools/free-notebooklm.ts

功能說明：透過 CLI-Anything-Web 呼叫免費 NotebookLM 進行研究。

FastMCP 註冊方式：typescript
server.addTool({
  name: "free_notebooklm",
  description: "免費 NotebookLM 研究（CLI-Anything-Web）",
  parameters: z.object({
    query: z.string().max(5000).describe("研究問題"),
    sources: z.array(z.string()).max(5).optional().describe("參考來源 URL"),
  }),
  execute: async (args, { log }) => {
    return await notebookLM.research(args, log);
  },
});
輸出：summary、key_findings（陣列）、sources_used。

錯誤碼：CLI_NOT_FOUND、NOTEBOOKLM_ERROR、TIMEOUT、AUTH_REQUIRED。

驗收標準：AC-1 有效 query 回傳 summary 非空。AC-2 CLI 不存在 → CLI_NOT_FOUND。AC-3 超時 → TIMEOUT。

5.4 spec_read

檔案：mcp-server/tools/spec-read.ts

功能說明：讀取並解析 spec 文件。底層委託 @modelcontextprotocol/server-filesystem 的 read_text_file 讀取檔案內容，上層自行解析 markdown section。

FastMCP 註冊方式：typescript
server.addTool({
  name: "spec_read",
  description: "讀取並解析 spec 文件（委託 filesystem server 讀檔 + section 解析）",
  parameters: z.object({
    path: z.string().default("docs/spec-mcp-server.md").describe("spec 檔案路徑"),
    section: z.string().optional().describe("指定章節標題（選填，不填回傳全文）"),
  }),
  execute: async (args) => {
    return await specRead.read(args);
  },
});
輸出：content（全文或指定 section）、sections（章節列表）、path、token_count。

錯誤碼：FILE_NOT_FOUND、SECTION_NOT_FOUND、PERMISSION_DENIED。

驗收標準：AC-1 有效路徑 → 回傳 content 非空。AC-2 指定 section → 只回傳該章節。AC-3 不存在的 section → SECTION_NOT_FOUND。AC-4 docs/ 外的路徑 → PERMISSION_DENIED。

5.5 spec_update

檔案：mcp-server/tools/spec-update.ts

功能說明：更新 spec 文件並自動 git commit。底層委託 @modelcontextprotocol/server-filesystem 的 write_file 寫入，上層處理 section 定位、內容插入/替換、git commit。

FastMCP 註冊方式：typescript
server.addTool({
  name: "spec_update",
  description: "更新 spec 並自動 git commit（委託 filesystem server 寫檔 + git）",
  parameters: z.object({
    path: z.string().default("docs/spec-mcp-server.md"),
    section: z.string().describe("要更新的章節標題"),
    content: z.string().max(10000).describe("新內容（Markdown）"),
    commit_message: z.string().max(200).optional().describe("commit 訊息（選填，自動生成）"),
  }),
  execute: async (args) => {
    return await specUpdate.update(args);
  },
});
輸出：updated（boolean）、path、section、commit_sha、diff_summary。

錯誤碼：FILE_NOT_FOUND、SECTION_NOT_FOUND、GIT_COMMIT_FAILED、PERMISSION_DENIED、CONTENT_TOO_LONG。

驗收標準：AC-1 有效 section + content → updated: true + commit_sha 非空。AC-2 不存在的 section → SECTION_NOT_FOUND。AC-3 更新後 spec_read 可讀到新內容。AC-4 git log 有對應 commit。

5.6 檔案系統工具（借力 @modelcontextprotocol/server-filesystem）

來源：npm @modelcontextprotocol/server-filesystem（Anthropic 官方）

不需自建。由 index.ts 在啟動時以子程序方式啟動 filesystem server，或以 MCP proxy 方式整合到 FastMCP server。

啟動方式：typescript
// 方式一：作為獨立子程序（推薦）
// Agent 的 MCP 設定中直接加入 filesystem server
// {
//   "filesystem": {
//     "command": "npx",
//     "args": ["-y", "@modelcontextprotocol/server-filesystem",
//              "./docs", "./mcp-server", "./design", "./knowledge",
//              "./agent-config", "./scripts", "./src", "./packages", "./test"]
//   }
// }

// 方式二：在 index.ts 中以 proxy 方式整合
// 將 filesystem server 的 13 個工具轉發到 FastMCP server
提供的工具（13 個）：

| 工具名稱 | 說明 | 對應原 spec 工具 |
|---------|------|-----------------|
| read_text_file | 讀取檔案內容（UTF-8） | file_read |
| write_file | 建立或覆寫檔案 | file_write |
| list_directory | 列出目錄內容 | file_list |
| edit_file | 選擇性編輯（pattern match） | — 新增 |
| search_files | 遞迴搜尋檔案 | — 新增 |
| directory_tree | 遞迴目錄樹（JSON） | — 新增 |
| get_file_info | 檔案 metadata | — 新增 |
| move_file | 搬移/重新命名 | — 新增 |
| create_directory | 建立目錄 | — 新增 |
| read_media_file | 讀取圖片/音訊 | — 新增 |
| read_multiple_files | 批次讀取 | — 新增 |
| list_directory_with_sizes | 含檔案大小的目錄列表 | — 新增 |
| list_allowed_directories | 列出允許的目錄 | — 新增 |

安全性：allowed-path 由命令列參數控制（只允許上述 9 個目錄）。denied_patterns（.env、.key、.pem、.git/）由 security-manager.ts 在 spec_read / spec_update 的 wrapper 層補充檢查。

驗收標準：

- AC-1：read_text_file 可讀取 docs/constitution.md。
- AC-2：write_file 可寫入 test/ 目錄下的檔案。
- AC-3：list_directory 可列出 docs/ 目錄。
- AC-4：嘗試讀取 /etc/passwd → 被 allowed-path sandbox 擋下。
- AC-5：search_files 可在 docs/ 下找到含 "MCP" 的檔案。
- AC-6：list_allowed_directories 回傳正確的 9 個目錄。

5.7 sync_check

檔案：mcp-server/tools/sync-check.ts

功能說明：比對 design.md 與 spec 文件的一致性。純邏輯比對，不依賴外部服務。

FastMCP 註冊方式：typescript
server.addTool({
  name: "sync_check",
  description: "比對 design.md 與 spec 一致性（CI 自動觸發）",
  parameters: z.object({
    design_path: z.string().default("design/design.md"),
    spec_path: z.string().default("docs/spec-mcp-server.md"),
  }),
  execute: async (args) => {
    return await syncCheck.check(args);
  },
});
輸出：in_sync（boolean）、mismatches（item / in_design / in_spec / suggestion 陣列）、design_only、spec_only。

錯誤碼：FILE_NOT_FOUND、PARSE_ERROR。

驗收標準：AC-1 命名不匹配回報 mismatch。AC-2 未定義頁面出現在 design_only。AC-3 完全一致時 in_sync: true。

6. P2 工具規格

6.1 design_generate

檔案：mcp-server/tools/design-generate.ts

FastMCP 註冊方式：typescript
server.addTool({
  name: "design_generate",
  description: "UI 設計生成（Stitch SDK）",
  parameters: z.object({
    prompt: z.string().max(5000).describe("設計需求描述"),
    project_id: z.string().optional(),
    design_system: z.string().optional(),
  }),
  execute: async (args) => {
    return await designGenerate.generate(args);
  },
});
輸出：screen_id、html_url、image_url、project_id。

錯誤碼：AUTH_FAILED、STITCH_ERROR、QUOTA_EXCEEDED。

驗收標準：AC-1 有效 prompt 回傳 html_url 和 image_url。AC-2 無 API Key → AUTH_FAILED。AC-3 指定 project_id 在既有專案新增 screen。

6.2 code_generate

檔案：mcp-server/tools/code-generate.ts

FastMCP 註冊方式：typescript
server.addTool({
  name: "code_generate",
  description: "程式碼生成（Gemini API / OpenCode）",
  parameters: z.object({
    prompt: z.string().max(10000).describe("程式碼需求描述"),
    language: z.string().default("javascript").optional(),
    spec_context: z.string().optional().describe("相關 spec 片段"),
    provider: z.enum(["gemini", "opencode"]).default("gemini").optional(),
  }),
  execute: async (args) => {
    return await codeGenerate.generate(args);
  },
});
輸出：code、language、explanation、provider。

錯誤碼：API_ERROR、PROVIDER_UNAVAILABLE、INVALID_LANGUAGE。

驗收標準：AC-1 回傳有效程式碼。AC-2 含 spec_context 時符合 I/O 定義。AC-3 provider 不可用 → PROVIDER_UNAVAILABLE。

6.3 desktop_control

檔案：mcp-server/tools/desktop-control.ts

FastMCP 註冊方式：typescript
server.addTool({
  name: "desktop_control",
  description: "桌面操控（Playwright，獨立於 weblm-driver）",
  parameters: z.object({
    action: z.enum(["navigate", "click", "type", "screenshot", "evaluate"]),
    target: z.string().optional().describe("URL 或 CSS selector"),
    value: z.string().optional().describe("type 動作的輸入值"),
  }),
  execute: async (args) => {
    return await desktopControl.execute(args);
  },
});
輸出：success、screenshot_url（screenshot 時）、result（evaluate 時）。

錯誤碼：BROWSER_LAUNCH_FAILED、ELEMENT_NOT_FOUND、NAVIGATION_TIMEOUT、ACTION_FAILED。

注意：desktop_control 使用自己的 Playwright 實例，與 weblm-driver 的瀏覽器完全獨立，避免干擾 LLM 會話。

驗收標準：AC-1 navigate 成功。AC-2 screenshot 回傳有效路徑。AC-3 不存在 selector → ELEMENT_NOT_FOUND。

7. 主入口規格 — index.ts（FastMCP 版）

檔案：mcp-server/index.ts

FastMCP 初始化：typescript
import { FastMCP } from "fastmcp";
import { z } from "zod";
import { readFileSync } from "fs";

// 讀取設定
const config = JSON.parse(readFileSync("./mcp-config.json", "utf-8"));

// 建立 FastMCP Server
const server = new FastMCP({
  name: config.server.name,
  version: config.server.version,
  instructions: "Project Golem MCP Universal Server — 15 支 Tool，支援免費 LLM、Pyramid Memory、Spec 管理、檔案操作。",
  health: {
    enabled: true,
    path: "/health",
    message: "golem-mcp-server ok",
  },
});
啟動流程：
node dist/index.js（或 npx fastmcp dev src/index.ts）
  │
  ├── 讀取 mcp-config.json
  │
  ├── 初始化共用元件
  │   ├── SecurityManager（denied-pattern 補充）
  │   ├── PyramidMemory（LanceDB 連接，確認三個 namespace table）
  │   ├── NeuroShunter
  │   └── GeminiWebDriver（weblm-driver）
  │       ├── driver.init()
  │       └── 失敗 → log.warn()，browser 層標記不可用（不 crash）
  │
  ├── 註冊 Tool handlers（server.addTool() × 12 支自建/混合工具）
  │   ├── free_llm_query
  │   ├── memory_store / memory_search / memory_compress
  │   ├── parse_protocol
  │   ├── free_notebooklm
  │   ├── spec_read / spec_update
  │   ├── sync_check
  │   ├── design_generate / code_generate / desktop_control
  │   └── filesystem server 的 13 個工具由獨立子程序或 proxy 提供
  │
  ├── 啟動定期健康檢查
  │   └── setInterval(() => driver.health(), 60000) → log.info()
  │
  ├── 啟動 FastMCP Server
  │   └── server.start({ transportType: "stdio" })
  │
  └── 就緒 log: "Golem MCP Server v0.2.0 started with N tools (browser: ok/degraded)"
驗收標準：

- AC-1：無錯誤啟動，log 顯示 tool 數量與 browser 狀態。
- AC-2：tools/list 回傳正確數量（12 支自建 + filesystem server 的 13 支）。
- AC-3：tools/call 呼叫已註冊 tool 回傳有效回應。
- AC-4：呼叫未註冊 tool 回傳 MCP error（FastMCP 自動處理）。
- AC-5：mcp-config.json 語法錯誤時 log 並 exit(1)。
- AC-6：記憶體  80%（借力套件自帶測試另計） |
| T-013 | 整合測試 | test/integration/*.test.ts | T-010, T-011 | 0.5h | JSON-RPC 端對端、跨工具場景 |
| T-014 | Dockerfile + docker-compose.yml | Dockerfile, docker-compose.yml | T-010 | 0.5h | 含 Playwright chromium 安裝 |

Phase 3 合計：~14 小時

8.2 Phase 4 任務（~9.5 小時）

| ID | 任務 | 檔案 | 依賴 | 預估 |
|----|------|------|------|------|
| T-015 | design-generate.ts（Stitch SDK） | tools/design-generate.ts | T-010 | 2h |
| T-016 | free-notebooklm.ts（CLI-Anything-Web） | tools/free-notebooklm.ts | T-010 | 2h |
| T-017 | code-generate.ts（Gemini API / OpenCode） | tools/code-generate.ts | T-010 | 2h |
| T-018 | desktop-control.ts（Playwright） | tools/desktop-control.ts | T-010 | 2h |
| T-019 | Phase 4 整合測試 | test/integration/phase4.test.ts | T-015~T-018 | 1.5h |

Phase 4 合計：~9.5 小時

Phase 3 + 4 總計：~23.5 小時（約 8 天 × 3h/天，或用 Claude Code 約 4-5 天）

8.3 任務並行化建議
並行組 A（無依賴，可同時開始）：
  T-001, T-002

並行組 B（依賴 T-001）：
  T-003, T-006, T-007, T-008, T-011

並行組 C（依賴 T-003）：
  T-004, T-005

並行組 D（依賴 T-008）：
  T-009

串行：T-010 → T-012 → T-013 → T-014
8.4 v1.2 省時明細

| 借力來源 | 刪除/簡化的任務 | 省下工時 |
|---------|---------------|---------|
| FastMCP | 刪除 T-016（原 index.js 手寫 JSON-RPC，2h）；T-010 從 2h 降到 0.5h | ~3.5h |
| @modelcontextprotocol/server-filesystem | 刪除 T-010T-012（原 file-read/write/list，2.25h）；T-002 從 1h 降到 0.5h | 2.75h |
| weblm-driver（v1.1 已計入） | 刪除原 Playwright 全流程實作（3.5h）；測試減半（1.5h） | ~5h |
| 合計 | | ~11.25h（從原始 28h → 23 項 21h → 18 項 14h） |

9. 測試策略

9.1 單元測試

每支自建工具一個測試檔案：test/golem-engine/.test.ts、test/tools/.test.ts。

使用 vitest。測試 happy path + 所有 error code。FastMCP 的 UserError 拋出需在測試中驗證。

weblm-driver 自帶 134 個測試，不重複測試其內部邏輯，只測試 browser-loop.ts 的 fallback 路徑。

@modelcontextprotocol/server-filesystem 自帶測試，不重複測試其內部邏輯，只測試整合層（確認 13 個工具可用）。

9.2 整合測試

啟動 FastMCP server（npx fastmcp dev）→ stdin 送 JSON-RPC → 驗證 stdout。

測試跨工具場景：memory_store → memory_search → memory_compress → memory_search。
測試 weblm-driver 降級場景：init 失敗 → Server 仍啟動 → free_llm_query fallback 到 API/Ollama。
測試 filesystem server 整合：read_text_file → write_file → list_directory。

9.3 覆蓋率目標

golem-engine/ > 80%（browser-loop.ts 的 fallback 邏輯需 100% 路徑覆蓋）。
tools/ > 70%。
index.ts > 60%。
weblm-driver 自帶 134 個測試另計。
filesystem server 自帶測試另計。

10. 環境變數env
.env.example

=== weblm-driver（Browser-in-the-Loop）===
SMOKE_PROFILE_DIR=               # Chrome profile 目錄（啟用 browser 層必填）
CHROME_EXECUTABLE=               # Chrome 執行檔路徑（選填）

=== Gemini API（第二層 fallback）===
GEMINI_API_KEY=                  # Gemini API Key（選填）

=== Stitch SDK ===
STITCH_API_KEY=                  # Stitch API Key（選填）
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

11.1 開發環境啟動bash
cd mcp-server
npm install          # 自動安裝 fastmcp、weblm-driver、filesystem server
cp .env.example .env # 填入 SMOKE_PROFILE_DIR
npm run build        # TypeScript 編譯
npm run start        # 正式啟動
或
npm run dev          # 開發模式（fastmcp dev，互動測試）
npm run inspect      # MCP Inspector Web UI 除錯
11.2 作為 Hermes Agent 的 MCP Serveryaml
~/.hermes/config.yaml
mcp:
  servers:
    golem-engine:
      command: "node"
      args: ["/path/to/mcp-server/dist/index.js"]
      description: "Golem Engine MCP Server (FastMCP)"
    filesystem:
      command: "npx"
      args: ["-y", "@modelcontextprotocol/server-filesystem",
             "./docs", "./mcp-server", "./design", "./knowledge",
             "./agent-config", "./scripts", "./src", "./packages", "./test"]
      description: "Anthropic Filesystem Server"
11.3 作為 Claude Code 的 MCP Serverjson
{
  "golem-engine": {
    "command": "node",
    "args": ["./mcp-server/dist/index.js"],
    "description": "Golem Engine MCP Server (FastMCP)"
  },
  "filesystem": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem",
             "./docs", "./mcp-server", "./design", "./knowledge",
             "./agent-config", "./scripts", "./src", "./packages", "./test"],
    "description": "Anthropic Filesystem Server"
  }
}
11.4 Dockerdockerfile
FROM node:20-slim
WORKDIR /app

複製專案
COPY mcp-server/ ./mcp-server/
COPY packages/ ./packages/

安裝依賴
RUN cd mcp-server && npm ci --production

weblm-driver 的 playwright dependency
RUN npx playwright install --with-deps chromium

filesystem server 全域安裝
RUN npm install -g @modelcontextprotocol/server-filesystem

CMD ["node", "mcp-server/dist/index.js"]
yaml
docker-compose.yml
version: "3.8"
services:
  golem-mcp:
    build: .
    volumes:
      - ./docs:/app/docs
      - ./design:/app/design
      - ./knowledge:/app/knowledge
      - ./agent-config:/app/agent-config
      - ./data:/app/data
    env_file: ./mcp-server/.env
    stdin_open: true
    tty: true
12. 驗收總表

| # | 標準 | 驗證方式 |
|---|------|---------|
| 1 | npm run start 無錯誤啟動，log 顯示 tool 數量 | 手動執行 |
| 2 | tools/list 回傳自建 12 支工具 | JSON-RPC |
| 3 | filesystem server 回傳 13 支工具 | JSON-RPC |
| 4 | free_llm_query weblm-driver 層可用 | 整合測試 |
| 5 | free_llm_query fallback 到 API 可用 | 單元測試 + mock |
| 6 | free_llm_query fallback 到 Ollama 可用 | 單元測試 + mock |
| 7 | free_llm_query ALL_FALLBACKS_FAILED 正確拋出 UserError | 單元測試 |
| 8 | weblm-driver init 失敗 → FastMCP Server 仍啟動 | 整合測試 |
| 9 | memory_store → memory_search 端對端可用 | 整合測試 |
| 10 | memory_compress 不混合 namespace | 單元測試 |
| 11 | parse_protocol context budget 裁剪正確 | 單元測試 |
| 12 | read_text_file 可讀 docs/constitution.md | 整合測試 |
| 13 | write_file 被 allowed-path sandbox 限制 | 整合測試 |
| 14 | spec_read / spec_update 可操作 spec 文件 | 整合測試 |
| 15 | sync_check 偵測不一致 | 單元測試 |
| 16 | 核心模組測試覆蓋率 > 80% | vitest --coverage |
| 17 | 記憶體 < 512 MiB | 手動檢查 |
| 18 | SIGTERM → driver.shutdown() → 正常退出 | 整合測試 |
| 19 | npm run dev（fastmcp dev）可互動測試 | 手動執行 |
| 20 | npm run inspect（fastmcp inspect）可開啟 Web UI | 手動執行 |
| 21 | Docker build 成功 + 容器內啟動正常 | docker build + run |

版本紀錄

- v1.0（2026-04-15）：初版——15 支 Tool 完整規格、25 項任務、測試策略
- v1.1（2026-04-15）：整合 weblm-driver——free_llm_query 第一層改為委託 GeminiWebDriver（四級恢復 + outputKind 檢查）、browser-loop.js 只負責 fallback 調度與 MCP 封裝、新增 weblm_driver 設定區塊於 mcp-config.json、新增 driver 生命週期管理於 index.js（init/health/shutdown）、Task List 從 25→23 項（刪除重複 Playwright 實作）、Phase 3 預估從 28→21 小時、新增 weblm-driver 節省明細表、新增驗收項 #7 #18 #19、新增 package.json 依賴定義、.env.example 新增 SMOKE_PROFILE_DIR 和 CHROME_EXECUTABLE
- v1.2（2026-04-15）：混合借力策略——MCP 框架從手寫 JSON-RPC 遷移至 FastMCP（npm: fastmcp）、檔案工具從自建改為 @modelcontextprotocol/server-filesystem（Anthropic 官方 13 個工具）、所有 Tool 改用 FastMCP server.addTool() + zod schema 註冊、錯誤處理改用 FastMCP UserError、新增 fastmcp dev / inspect 開發工具、檔案從 .js 改為 .ts、index.js 改為 index.ts（FastMCP 初始化）、刪除 file-read/write/list 自建檔案、package.json 新增 fastmcp + zod + filesystem server 依賴、工具名稱 file_read/file_write/file_list 改為官方名 read_text_file/write_file/list_directory、Task List 從 23→18 項、Phase 3 預估從 21→14 小時、新增 §8.4 省時明細、驗收總表從 19→21 項、Agent 部署設定新增 filesystem server 獨立配置
