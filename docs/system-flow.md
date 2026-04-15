# Project Golem — System Flow and Closed-Loop Architecture
# 系統流程與閉環架構圖

---

## 1. 完整系統程序圖

主幹流程：
NotebookLM（研究收斂）
  → Spec Kit（寫規格）
  → Stitch SDK（設計 UI）
  → Claude Code / OpenCode（實作程式碼）
  → MCP Server Core（整合測試）
  → Hermes Agent（自主執行）
  → 使用者（Telegram / Discord / Web Dashboard）

同時 Claude Code → GitHub CI/CD（版控與自動測試）

MCP Server 連接的模組：
  - Pyramid Memory（長期記憶）
  - Browser-in-the-Loop（免費 Gemini）
  - Google Tools（Stitch SDK / NotebookLM CLI）
  - CLI-Anything（桌面軟體 CLI 化）

閉環回饋箭頭：
  - 閉環 1（學習）：Hermes → 經驗存入 → Pyramid Memory → 經驗召回 → MCP → Hermes
  - 閉環 2（設計）：Claude Code → UI 不滿意 → Stitch → 更新 design.md → Claude Code
  - 閉環 3（Spec）：Claude Code → spec 不足 → Spec Kit；MCP → 發現缺陷 → Spec Kit
  - 閉環 4（知識）：Hermes → 需要研究 → NotebookLM → 結果存入 → Pyramid Memory

Opal（選用）：邏輯確認後 → Spec Kit（用完即走）

---

## 2. 四個閉環運轉狀態

系統上線後：
  持續運轉 → 閉環 1（學習）每次互動都在跑
  偶爾觸發 → 閉環 2（設計）新 UI 或改版時
  偶爾觸發 → 閉環 3（Spec）新功能或重構時
  低頻觸發 → 閉環 4（知識）遇到新領域時

---

## 3. 閉環 1：學習迴圈

類型：自動，持續運轉
流程：
  1. Hermes 執行任務
  2. 結果 + metadata 存入 Pyramid Memory（Tier 0 即時層）
  3. 系統定期壓縮（即時→日→週→月→年）
  4. 下次類似任務時 memory_search 召回經驗
  5. MCP 把經驗注入 LLM context
  6. Hermes 產出更精準的結果
  7. 再次存入 Memory，循環繼續

觸發條件：每次任務執行完畢自動觸發
頻率：最高頻
價值：越用越聰明，約 40% 效能提升（Hermes 官方數據）

---

## 4. 閉環 2：設計迭代迴圈

類型：半自動，按需觸發
流程：
  1. Stitch 根據 prompt 產出 design.md + HTML + 截圖
  2. Claude Code 讀取 design.md 生成 UI 程式碼
  3. 預覽或測試發現 UI 不符預期
  4. 回到 Stitch 用 screen.edit("修改指令") 修改
  5. Stitch 更新 design.md
  6. Claude Code 重新生成對應程式碼
  7. 滿意為止，退出迴圈

觸發條件：UI Review 不通過時
頻率：每個 UI 模組 2-5 次迭代
價值：確保設計與實作一致

---

## 5. 閉環 3：Spec 演化迴圈

類型：手動，關鍵節點觸發
流程：
  1. Spec Kit 產出 spec + plan + tasks
  2. Claude Code 開始逐 task 實作
  3. 實作過程中發現問題：
     - spec 遺漏了某個邊界條件
     - API 契約需要調整
     - 模組依賴沒寫清楚
  4. 回到 Spec Kit 更新 spec
  5. 重新執行 /speckit.plan 和 /speckit.tasks
  6. 繼續實作

觸發條件：實作中遇到 spec 與現實不符
頻率：每個模組 1-3 次
價值：Spec 是 living document，保持 spec 與程式碼同步，避免架構漂移

---

## 6. 閉環 4：知識迴圈

類型：按需觸發，跨專案累積
流程：
  1. Hermes 執行任務中遇到未知領域
  2. 觸發 NotebookLM 深度研究（透過 free_notebooklm MCP Tool）
  3. NotebookLM 交叉比對多個來源，產出帶引用的研究摘要
  4. 研究結果存入 Pyramid Memory
  5. 下次遇到相關任務時 memory_search 直接召回
  6. 不需要再次研究

觸發條件：Agent 遇到知識盲區或使用者要求研究
頻率：低頻但高價值
價值：知識永久累積，跨專案可複用

---

## 7. 工具角色定位總表

NotebookLM: 研究收斂，Phase 0 + 閉環 4，參與知識迴圈
Opal: 白板驗證，Phase 0.5 選用，無閉環參與（用完即走）
Spec Kit: 規格管理，Phase 1 + 閉環 3，參與 Spec 演化
Stitch SDK: UI 設計，Phase 2 + 閉環 2，參與設計迭代
Claude Code / OpenCode: 程式碼生成，Phase 3，參與設計迭代 + Spec 演化
MCP Server: 萬用接口，Phase 3+ 永久運行，所有閉環的中樞
Browser-in-the-Loop: 免費 LLM，永久運行，參與學習迴圈
Pyramid Memory: 長期記憶，永久運行，參與學習 + 知識迴圈
Hermes Agent: 自主執行，Phase 5+ 永久運行，參與學習 + 知識迴圈
CLI-Anything-Web: Web App CLI 化，Phase 4，參與知識迴圈（NotebookLM）
GitHub: 版控 CI/CD，全程，參與 Spec 演化（PR review）

---

## 8. 統一介面架構

日常操作 → Hermes Dashboard (localhost:9119)
  對話、下指令、看結果、管理 Skill/MCP Tool、排程 Cron、監控 Token 用量

深入記憶 → Golem Dashboard (localhost:原本的 port)
  Pyramid Memory 視覺化、日記輪轉與備份、記憶壓縮狀態

寫程式碼 → Claude Code / OpenCode（終端機或 Remote Control）
  讀 spec + design.md、逐 task 實作

設計 UI → Stitch（瀏覽器或透過 Hermes 呼叫 design_generate）

概念驗證 → Opal（瀏覽器，手動，用完即走）

90% 的操作在 Hermes Dashboard 一個視窗完成。
