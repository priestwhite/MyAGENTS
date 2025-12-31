# MERA 記憶治理規範 v0.1

> **狀態**：草案  
> **最後更新**：2025-12-26  
> **討論紀錄**：[MERA-DISCUSSION.md](./MERA-DISCUSSION.md)

---

## 1. 設計原則

| 面向 | 共識 |
|------|------|
| **記憶工具職責** | 純儲存/檢索，如同大腦的「記憶」功能 |
| **判斷/推理** | LLM 負責，不在記憶層處理 |
| **記憶本質** | 持久化的上下文擴展 |
| **觸發時機** | 每次對話開始時，由 Agent 主動呼叫 |
| **優先級** | 準確 > 快速 |

---

## 2. 記憶分層架構

```
┌─────────────────────────────────────────────────────────┐
│  Layer 0: 核心規則 (Always)                              │
│  - 篩選: is_core=true 或 user_weight >= 9               │
│  - 觸發: 對話開始時自動注入                              │
│  - 上限: 10 條                                          │
├─────────────────────────────────────────────────────────┤
│  Layer 1: 專案/Scope 記憶 (By Context)                  │
│  - 篩選: scope 匹配當前工作區                            │
│  - 觸發: 識別到專案路徑/語言/框架後                      │
│  - 上限: 5 條                                           │
├─────────────────────────────────────────────────────────┤
│  Layer 2: 任務相關記憶 (By Query)                        │
│  - 篩選: 語意/關鍵字檢索                                 │
│  - 觸發: 見下方觸發條件                                  │
│  - 上限: 5 條（可多次檢索）                              │
└─────────────────────────────────────────────────────────┘
```

### Layer 2 觸發條件（啟發式規則）

| 條件 | 說明 |
|------|------|
| 歷史關鍵字偵測 | 問題包含「之前」「上次」「歷史」「過去」等關鍵字 |
| 未解析 Entity | 偵測到專案相關 entity（如技術名詞）但上下文無相關記憶 |
| 決策類問題 | 問題類型判定為「選擇」「比較」「決策」 |
| 週期性強制檢索 | 每 5 輪對話強制檢索一次（防呆機制） |

> [!NOTE]
> 目前不使用 LLM 自評信心度，因缺乏穩定的計算方式。未來若有可靠算法再行導入。

---

## 3. Schema 欄位規範

### 3.1 必填欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `summary` | string | 50 字以內結論 |
| `memory_class` | enum | `policy` / `episodic` / `mixed` |
| `scope` | string | `global` / `project:<name>` / `lang:<name>` |

### 3.2 可選欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `canonical_topic` | string | 主題分類（如 `database:choice`） |
| `supersedes` | array | 取代的 memory_id 列表 |
| `status` | enum | `active` / `deprecated` |
| `claims` | array | 結構化的事實主張列表 |
| `is_core` | boolean | 核心記憶標記 |
| `enforcement_level` | enum | `soft` / `hard`（僅 policy 類） |

### 3.3 記憶分類說明

| 類型 | 特性 | 範例 |
|------|------|------|
| **policy** | 穩定、可版本化、難被覆蓋 | 「輸出語言必須是繁體中文」 |
| **episodic** | 可追溯、允許過期、可被取代 | 「專案改用 PostgreSQL」 |
| **mixed** | 混合特性 | 視情況處理 |

---

## 4. 記憶生命週期

### 4.1 健康度公式

#### 場景 A：檢索排序（有查詢語句）

```python
retrieval_score = 0.4 × relevance + 0.25 × recency_decay + 0.2 × access_frequency + 0.15 × user_weight
```

#### 場景 B：清理評估（無查詢語句）

```python
health_score = 0.4 × recency_decay + 0.35 × access_frequency + 0.25 × user_weight
```

### 4.2 各變數計算方式

| 變數 | 計算公式 | 實作狀態 |
|------|----------|----------|
| `relevance` | Hybrid Search 回傳的 score（0~1） | ✅ 已實作 |
| `recency_decay` | `exp(-0.05 × days_since_created)` | ✅ 可計算（用 `created_at`） |
| `user_weight` | `metadata.user_weight / 10`（正規化為 0~1） | ✅ 已實作 |
| `access_frequency` | `min(metadata.access_count / 20, 1)` | ✅ 已實作 |

### 4.3 recency_decay 計算範例

```python
import math
from datetime import datetime, timezone

def recency_decay(created_at: str, half_life_days: int = 14) -> float:
    """計算時間衰減分數，半衰期預設 14 天"""
    created = datetime.fromisoformat(created_at)
    now = datetime.now(timezone.utc)
    days_old = (now - created).days
    decay_rate = 0.693 / half_life_days  # ln(2) / half_life
    return math.exp(-decay_rate * days_old)

# 範例：
# 今天建立: 1.0
# 14 天前: 0.5
# 28 天前: 0.25
# 60 天前: 0.05
```

### 4.4 清理流程

```
降權 → 停止檢索 → 封存 → 定期刪除
```

| 階段 | 條件 | 動作 |
|------|------|------|
| 降權 | health_score < 0.3 | 標記 `status: low_priority` |
| 停止檢索 | health_score < 0.15 | 標記 `status: archived` |
| 定期刪除 | archived 超過 60 天 | 執行 `memory_delete` |

> [!CAUTION]
> 不建議直接 hard delete，避免誤殺重要記憶。

---

## 5. 衝突處理

### 5.1 版本鏈機制

- 使用 `supersedes: [memory_id]` 標記取代關係
- 使用 `status: active | deprecated` 標記生效狀態
- 檢索時優先返回 `active` 且鏈尾（最新版本）

### 5.2 衝突檢測

- 寫入前做「相似記憶 Top-K」檢查
- 同 `canonical_topic` 內有多筆 `active` 時觸發警告
- 提示用戶選擇「新增」或「取代」

---

## 6. Token 預算管理（Agent 端責任）

> [!NOTE]
> Token 預算管理由 **Agent/LLM 端** 負責，非 MCP 實作範圍。
> MCP 遵循「純儲存/檢索」原則，不進行 token 計算或載入決策。

### 6.1 Agent 端建議策略

| 步驟 | 說明 |
|------|------|
| 1. 設定預算 | 對話開始時設定記憶注入預算（如 2000 tokens） |
| 2. 依序載入 | Layer 0 → Layer 1 → Layer 2，預算不足時停止 |
| 3. 動態壓縮 | 預算緊張時只載入 `summary`，不載入完整 `content` |

### 6.2 載入優先級

```
Layer 0（核心規則）→ 完整載入
Layer 1（專案記憶）→ 預算允許時載入
Layer 2（任務記憶）→ 依剩餘預算決定
```

---

## 7. 實作路線圖

| 階段 | 改動內容 | 優先級 |
|------|----------|--------|
| Phase 1 | 透過 `metadata` 欄位模擬（`topic`, `supersedes`, `status`, `access_count`） | ✅ 已完成 |
| Phase 2 | 正式新增 `canonical_topic`, `memory_class` 欄位到 schema | ✅ 已完成 |
| Phase 3 | 支援 `supersedes` 鏈的遞迴查詢與 `memory_archaeology` 指令 | ✅ 已完成 |

---

## 8. 共識總覽

| 議題 | 共識內容 | 提出者 |
|------|----------|--------|
| 記憶層與推理層分離 | MCP 只負責儲存/檢索，決策交給 Agent/LLM | 多方共識 |
| 衝突處理機制 | 使用 `supersedes` + `status` 建立版本鏈 | 多方共識 |
| 記憶分類 | 引入 `memory_class` 區分規則類與事件類 | GPT、Grok |
| 回饋閉環 | 追蹤 `access_count` / `utility_score` | GEMINI、Grok |
| 核心記憶標記 | `is_core` 硬開關 + `user_weight` 排序雙軌制 | GPT、GEMINI |
| 跨專案隔離 | 預設隔離，`scope: global` 才共享 | GPT、GEMINI |
| Token 預算 | 主動式配額，Layer 0 原文優先 | Grok、GPT |
