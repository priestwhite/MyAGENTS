# AI 工作規範

> 本檔為 AI 助理在此資料夾及子資料夾工作時的強制規範。

---

## 0. 優先序

```
使用者指示 > 子資料夾 AGENTS.md > 本檔 > README.md
```

**變體判斷**：讀取專案 `README.md` 的 Variant Tag（CLI/Web/hybrid）。若無標記，預設 CLI。

**處理問題流程**：解構需求 → 讀取記憶（MCP001） → 蒐集方案（含上網查詢） → 驗證 → 討論確認後執行

---

## 0.1 開發命令速查

| 命令 | 說明 |
|------|------|
| `uv sync` 或 `pip install -r requirements.txt` | 安裝依賴（優先使用 uv） |
| `pytest` / `pytest -v` | 執行測試 |
| `ruff check . --fix && ruff format .` | 檢查並格式化 |
| `mypy src/` | 類型檢查 |

---

## 0.2 程式碼風格

### 命名慣例

| 類型 | 慣例 | 範例 |
|------|------|------|
| 變數 | `snake_case` | `user_list`, `is_valid` |
| 函數 | 動詞開頭 | `get_user()`, `validate_input()` |
| 類別 | `PascalCase` | `MetaClient`, `SafetyManager` |
| 常數 | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| 私有 | 前綴 `_` | `_internal_method()` |
| 時間 | `*_at` 結尾 | `created_at` |

### 匯入順序

標準庫 → 第三方套件 → 本地模組（各區依字母排序）

### 核心原則

- 單檔單功能、簡單勝過複雜
- 程式碼自說明、行為可追溯（預留日誌）
- 跨平台使用 `pathlib.Path`
- 敏感資訊用環境變數，勿硬編碼

---

## 0.3 技術棧標準

> 經 GPT/Gemini/Grok 交叉驗證，反映 2024-2025 業界趨勢。

### 語言分工

| 語言 | 角色 | 使用場景 |
|------|------|----------|
| **Python** | 主力 | 資料處理、AI/ML、後端 API (FastAPI)、MCP 工具 |
| **TypeScript** | 前端 | Web UI (Next.js/React)、Node.js BFF |
| **Go** | 效能層 | 高併發微服務、CLI 工具、長駐背景服務 |

### 資料庫

| 資料庫 | 定位 | 說明 |
|--------|------|------|
| **PostgreSQL** | 新專案標準 | JSONB、分析查詢、pgvector 向量搜尋、PostGIS |
| MySQL | 既有維護 | 現有專案繼續使用 |
| Redis | 快取/Session | 輔助用途 |

### 語言選擇原則

| 情境 | 選擇 |
|------|------|
| 資料處理、ML、快速原型 | Python |
| 所有前端 UI | TypeScript |
| 效能瓶頸、需極致併發 | Go |
| 新資料庫專案 | PostgreSQL |

---

## 1. 強制規則

### 1.1 程式碼修改安全規則

> [!CAUTION]
> **違反此規則將導致不可逆的程式碼損失。**

修改程式碼時，必須遵循「註解化修改流程」：

| 步驟 | 動作 | 標記 |
|------|------|------|
| 1 | 將目標程式碼註解化 | `# [DEPRECATED] 原因 - YYYY-MM-DD` |
| 2 | 在其下方寫入新程式碼 | `# [NEW] 說明 - YYYY-MM-DD` |
| 3 | 驗證通過或用戶允許後，才可移除舊碼 | - |

**例外**：明顯錯字、用戶明確說「直接改」、純新增程式碼

### 1.2 輸出規範

繁體中文 | 結論→步驟→建議（≤3 項） | 命令/路徑用反引號 | 精簡為主

### 1.3 禁止操作

❌ 提交含憑證的 `.env`/`config.ini` | ❌ 未備份即刪檔 | ❌ 硬編碼絕對路徑 | ❌ 依賴 CWD

---

## 2. 標準流程

**任務開始**：讀取 `README.md` 確認變體 → 檢查子資料夾 `AGENTS.md` → 多步任務持續更新進度

**任務進行**：變更時同步更新文檔 → 阻塞時詢問用戶 → 錯誤記錄至 `error.md`

**任務結束**：確認文檔同步 → 未解決問題標記 🔄/❌

---

## 3. 任務邊界（僅 Antigravity）

| 情境 | 使用？ |
|------|--------|
| 單檔快速編輯 | ❌ |
| 多步驟開發、複雜重構 | ✅（預估 3+ 工具呼叫） |

| 模式 | 目的 | 產出 |
|------|------|------|
| PLANNING | 研究、設計 | `implementation_plan.md` |
| EXECUTION | 實作 | 程式碼 |
| VERIFICATION | 測試驗證 | `walkthrough.md` |

任務進行中使用 `notify_user` 與用戶溝通。

---

## 4. 專案結構

### 必備文件

| 檔案 | 用途 |
|------|------|
| `README.md` | 專案文檔（含 Variant Tag、未來規劃章節） |
| `error.md` | 問題追蹤 |
| `requirements.txt` | 依賴 |
| `logs/` | 日誌（加入 .gitignore） |

### 變體結構

| 變體 | 核心目錄 | 驗證指令 |
|------|----------|----------|
| CLI | `src/` | `pytest src/` |
| Web | `routes/`, `templates/` | `pytest data/` |
| hybrid | 混合 | `pytest` |

---

## 5. 刪檔/更名流程

1. 備份原檔 → 2. 搜尋參照 (`Select-String`) → 3. 更新 import → 4. 同步 README → 5. 測試 → 6. 提交

---

## 6. error.md 格式

| 時間 | 錯誤摘要 | 影響範圍 | 處置方式 | 狀態 |
|------|----------|----------|----------|------|
| YYYY-MM-DD | 描述 | 模組名 | 解法 | ✅/🔄/❌ |

定期清理已解決超過 30 天的條目。

---

## 7. MCP 功能（條件觸發）

| MCP | 類型 | 觸發方式 |
|-----|------|----------|
| MCP001 | 記憶 | `memory_add/query/update`，先查 hybrid 模式 |
| MCP002 | SQL | `mysql_dry_run()` → `mysql_execute()` |
| MCP003 | 網搜 | `web_search()` / `web_search_summary()` |
| MCP004 | 系統 | `sys_info()`, `sys_process_list()`, `notify_toast()` 等 |

憑證只存鍵名不存值 | 清理：`memory_cleanup(30 days)`

---

## 8. 檢核清單

- [ ] 繁體中文輸出？
- [ ] 只改相關檔案？
- [ ] 同步 README/error.md？
- [ ] 簡單方案？
- [ ] 遵循註解化流程？
- [ ] pathlib 跨平台？
