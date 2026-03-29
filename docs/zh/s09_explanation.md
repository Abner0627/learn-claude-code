# s09_agent_teams.py 執行流程詳解

## 問題與解決方案摘要

### 問題

s04 的子 Agent 是**一次性的**：生成、執行、回傳摘要、消亡。沒有身份，沒有跨呼叫的記憶。s08 的後台任務能執行 Shell 指令，但無法做出 LLM 引導的決策。真正的團隊協作需要三樣東西：(1) 能跨多輪對話存活的持久 Agent；(2) 身份與生命週期管理；(3) Agent 之間的通訊管道。

### 解決方案

`TeammateManager` 透過 `config.json` 維護團隊名冊。`spawn()` 建立隊員並在執行緒中啟動 Agent Loop。`MessageBus` 實現 Append-only 的 JSONL 收件匣：`send()` 追加一行，`read_inbox()` 讀取全部並清空。每個隊員在每次 LLM 呼叫前檢查收件匣，將訊息注入上下文，實現異步的多 Agent 通訊。

---

本章展示了 Agent 團隊協作。透過文件系統作為信箱 (Mailbox)，讓多個模型常駐於系統中異步工作。當 Lead Agent 收到 `"Ask Alice to fix my code and Bob to review it."` 時，流程如下：

### 第一輪迴圈 (Iteration 1: 分派工作)

1.  **Lead 發號施令**：
    *   呼叫 `spawn_teammate` 啟動 `Alice` 與 `Bob`。
    *   這會為 `Alice` 與 `Bob` 各自開啟一個獨立的執行緒循環。
2.  **發送訊息**：
    *   Lead 呼叫 `send_message("Alice", "Fix logic in auth.py")`。
    *   這將訊息寫入 `.team/inbox/Alice.jsonl`。

---

### Alice 執行緒工作 (Alice's Loop)

1.  **排空信箱 (Draining Inbox)**：
    *   `Alice` 定期讀取 `Alice.jsonl`。
    *   訊息被載入 `Alice` 的對話歷史中。
2.  **開始工作**：
    *   `Alice` 呼叫 `edit_file` 修改 `auth.py`。
3.  **主動回報**：
    *   完成後，`Alice` 呼叫 `send_message("Lead", "Fixed. Asking Bob for review.")`。
    *   同時發訊息給 `Bob`：`send_message("Bob", "Review auth.py")`。

---

### Bob 執行緒工作 (Bob's Loop)

1.  **接收與執行**：
    *   `Bob` 從信箱收到 `Alice` 的通知。
    *   呼叫 `read_file` 進行程式碼審查。
2.  **給出回報**：`Bob` 向 `Lead` 回報審查結果。

### 核心總結
本章展示了 **「信箱通訊模式（Mailbox Pattern）」** 的力量。不同於子 Agent，團隊成員是常駐且擁有自主性的。透過文件系統解耦（Decoupling），我們實現了一個能跨執行緒、且易於觀測與除錯的異步 Agent 協作架構。

---

## 信箱通訊架構詳解

### 持久隊員 vs 一次性子代理 (s04)

| 維度 | s04 子代理 (Subagent) | s09 持久隊員 (Teammate) |
| :--- | :--- | :--- |
| **生命週期** | 生成 → 完成 → 消亡（一次性） | spawn → WORKING → IDLE → WORKING → ... → SHUTDOWN |
| **固定身份** | 無 | 有名字（name）、角色（role）、狀態 |
| **跨呼叫記憶** | 無 | 有（在同一線程的 `messages` 中持續累積） |
| **通訊能力** | 無（只回傳摘要給父代理） | 可主動發訊息給任何隊員 |
| **適用任務** | 一次性資訊查詢、短暫委派 | 需要多輪協商或長程持續工作的任務 |

### JSONL 信箱的追加-排空 (Append-Drain) 模式

`.team/inbox/alice.jsonl` 是一個追加式日誌檔案，其讀寫操作在不同線程中進行：

- **`send()`**：任何隊員呼叫此方法，訊息會被以 JSON 格式追加（`append`）到目標的 `.jsonl` 檔案末尾。這是非阻塞操作，寫入後立即返回。
- **`read_inbox()`**：讀取所有行後，立即將檔案內容清空（`write_text("")`），確保每條訊息只被處理一次。

這種設計的優點在於：寫入（`send`）與讀取（`read_inbox`）發生在不同線程中，且都只需要基本的檔案操作，不需要複雜的跨線程鎖機制。

### 訊息結構

每條訊息的 JSON 結構如下：

```json
{
  "type": "message",
  "from": "alice",
  "content": "Fixed auth.py. Please review.",
  "timestamp": 1730000000.0
}
```

`type` 欄位決定接收方如何處理這條訊息。預設為 `message`（普通訊息），s10 中會引入 `shutdown_request`、`plan_request` 等協議類型。

### 隊員的 `_teammate_loop` 信箱整合

每個隊員在每次 LLM 呼叫前讀取信箱，確保最新訊息被注入上下文。若信箱不為空，訊息以 `<inbox>` 標籤包裹注入：

```python
inbox = BUS.read_inbox(name)
if inbox != "[]":
    messages.append({"role": "user",
        "content": f"<inbox>{inbox}</inbox>"})
    messages.append({"role": "assistant",
        "content": "Noted inbox messages."})
```

### 團隊名冊（`config.json`）的結構

```json
{
  "members": [
    {"name": "alice", "role": "coder", "status": "working"},
    {"name": "bob",   "role": "tester", "status": "idle"}
  ]
}
```

`status` 欄位（`working` / `idle` / `shutdown`）讓 Lead 可以隨時掌握整個團隊的工作狀態，也是 s10 協議握手與 s11 自主認領的基礎。

---

## 常見問題 Q&A

### 1. 比對 s04 的一次性子 Agent，現代 Agent 是採用哪種架構？還是兩種並行？

現代生產級 Agent 系統**兩種並行使用**，但依任務性質分工，並非二選一。

**兩種模式的定位對比：**

| 維度 | s04 一次性子代理 | s09 持久隊員 |
| :--- | :--- | :--- |
| **核心價值** | Context 隔離、成本可控 | 跨輪記憶、自主協商 |
| **啟動成本** | 低（每次全新建立） | 高（需要 spawn 與狀態管理） |
| **適用任務** | 一次性查詢、資訊提取、批次處理 | 多輪協商、長程持續監控、角色分工 |
| **通訊方式** | 只回傳摘要給父代理 | 可主動發訊息給任何隊員 |

**現代框架的實際做法：**

主流 Agent 框架（CrewAI、AutoGen、LangGraph、Claude Code）幾乎都採用「**Orchestrator + 混合執行層**」的三層架構：

1. **Orchestrator（協調者）**：類似 s09 的 Lead，持久存活，負責決策與任務分派。
2. **持久專家隊員（Persistent Specialists）**：類似 s09 的 Alice / Bob，有固定角色與跨輪記憶，適合需要多輪討論或保持狀態的角色（如 Code Reviewer 持續追蹤品質）。
3. **一次性子代理（Disposable Workers）**：類似 s04，被 Orchestrator 或持久隊員臨時召喚，做完即棄，確保 Context 不會因瑣碎作業而膨脹。

**現實案例：**

*   **Claude Code**：Lead Agent 長駐，但掃描整個 Repo 時會 spawn 一次性子代理（`Agent` 工具）處理搜尋，只把結論回傳，不污染主 Context。
*   **AutoGen（Microsoft）**：定義具名角色（Coder、Critic、Planner）持久存在並互相溝通；同時支援呼叫無狀態函式完成原子操作。
*   **CrewAI**：Agent 與 Task 分離設計，Agent 是持久的角色，Task 是一次性的委派單元——兩者組合即為 s09 × s04 的混搭。

**結論**：兩種模式不是競爭關係，而是**互補的粒度控制工具**。持久隊員解決「誰在做事」的問題，一次性子代理解決「這件事該怎麼做而不拖累全局」的問題。現代系統的成熟度，往往體現在能否精準判斷何時該用哪一種。

### 2. 交辦任務給 Agent 時，要怎麼樣才能讓主 Agent 建立團隊？

觸發 Lead Agent 建立團隊需要**兩個條件同時成立**：系統設計層允許 + 使用者 Prompt 明確觸發。

**條件一：系統設計層（Lead 必須具備相關工具與指示）**

Lead 的 System Prompt 和工具列表決定了它「能不能」建立團隊：

```python
# System Prompt 必須告知 Lead 其角色與能力
SYSTEM = "You are a team lead. Spawn teammates and communicate via inboxes."

# 工具列表必須包含 spawn_teammate
{"name": "spawn_teammate", "description": "Spawn a persistent teammate..."}
```

若 System Prompt 沒有明確指示 Lead 扮演協調者角色，或工具列表缺少 `spawn_teammate`，無論 Prompt 怎麼寫，模型都不會嘗試建立團隊。

**條件二：使用者 Prompt（決定 Lead「要不要」用）**

模型根據任務性質自行判斷是否需要召喚隊員。以下是四種有效的觸發模式：

| 模式 | 範例 Prompt | 為何有效 |
| :--- | :--- | :--- |
| **明確點名** | `"Spawn Alice to fix auth.py and Bob to review it."` | 直接映射到工具呼叫 |
| **並行任務** | `"Fix the bug and review the changes at the same time."` | 「同時」暗示需要平行執行 |
| **角色分工** | `"I need a coder and a tester working on this together."` | 觸發職責切割 |
| **大規模任務** | `"Refactor auth, test every function, and document the API."` | 規模超過單一 Agent 合理範圍 |

**最不可靠的方式**：只描述任務本身、不提分工（如 `"Fix my code"`）。Lead 可能直接自己處理，不一定會 spawn 隊員。

**Lead Agent 收到指令後的決策流程：**

```
使用者 Prompt
    ↓
Lead 判斷：這個任務需要分工嗎？
    ├─ 是（有角色切割、有並行需求）→ 呼叫 spawn_teammate × N → 發訊息 → 等待回報
    └─ 否（簡單任務）→ 直接用 bash / edit_file 自己完成
```

**實際範例（觸發建立 Alice + Bob 團隊）：**

```
Ask Alice to fix the logic error in auth.py,
then have Bob review her changes before reporting back.
```

此 Prompt 包含：兩個具名角色、明確的依賴順序（fix → review → report），Lead 幾乎必然會呼叫兩次 `spawn_teammate` 並透過信箱協調流程。

### 3. 在 Claude Code 中，要用什麼 Prompt 才能讓 Agent 自行判斷是否組建團隊？

這個問題需要先釐清一個重要前提：**Claude Code 的多 Agent 機制跟 s09 不同**。

**Claude Code 的兩種 Agent 模式：**

| 模式 | 對應章節 | 穩定性 | 說明 |
| :--- | :--- | :--- | :--- |
| **Subagent（子代理）** | s04 | 穩定，預設啟用 | 一次性、上下文隔離、做完即棄 |
| **Agent Teams（團隊）** | s09 | 實驗性，預設關閉 | 持久隊員、有名字與狀態、可互相溝通 |

Claude Code 日常使用的是 **s04 模式的 Subagent**，而非 s09 的持久隊員。若要啟用持久 Agent 團隊，需要開啟實驗性功能（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`），且目前功能仍不完整（例如 `/resume` 後隊員狀態會消失）。

**觸發 Claude Code 自動使用 Subagent 的 Prompt 訊號：**

模型的判斷依據是**語意匹配**，不是 Token 數量。以下訊號會提高觸發機率：

| 訊號類型 | 範例 Prompt | 觸發機制 |
| :--- | :--- | :--- |
| **明確點名並行** | `"Research auth, database, and API in parallel"` | "parallel" / "simultaneously" 直接觸發 |
| **角色枚舉** | `"Have 3 reviewers check security, performance, and test coverage"` | 多角色 + 明確職責分工 |
| **探索與行動分離** | `"Use the Explore subagent to understand auth, then implement the fix"` | 讀取（隔離）→ 修改（主 Context） |
| **大範圍枚舉** | `"Fix failing tests in each module"` | "each" 暗示需要逐一平行處理 |
| **競爭假說** | `"Spawn 3 agents to investigate different root causes and debate"` | 多個獨立假說同時驗證 |

**最不可靠的寫法**：只說任務不說分工（`"Fix my code"`）→ Claude 直接自己做，不分派。

**讓 Subagent 自動委派的根本機制（`.claude/agents/`）：**

觸發自動委派的關鍵不是 Prompt，而是 **Subagent 的 `description` 欄位**。Claude Code 會將 Prompt 語意與 `.claude/agents/` 目錄下各 Agent 的描述做匹配，自動決定是否委派：

```yaml
---
name: code-reviewer
description: Use proactively after code changes to review security, performance, and maintainability.
tools: Read, Grep, Glob
---
```

`description` 中的 `"Use proactively"` 是關鍵字，表示這個 Subagent 可以被自動觸發，不需要使用者明確點名。

**CLAUDE.md 對多 Agent 的影響（有限）：**
*   **會繼承**：所有 Subagent 在生成時會自動載入父 Session 的 CLAUDE.md，獲得相同的專案背景。
*   **不會控制**：CLAUDE.md 無法強制 Claude 使用 Subagent，模型依然自主判斷。真正控制觸發行為的是 `.claude/agents/` 目錄下的 Agent 定義。

**實用建議：**
若你想讓 Claude Code 主動組建 Agent 處理複雜任務，最有效的做法是：
1. 在 `.claude/agents/` 定義清楚職責的 Subagent（含 `description`）。
2. Prompt 中明確說明**任務是獨立的、可平行的**，或**直接點名 Subagent**。
3. 對於 s09 式的持久隊員協作，目前最可靠的方式仍是直接執行自訂的 `s09_agent_teams.py`，而非依賴 Claude Code 的實驗性功能。
