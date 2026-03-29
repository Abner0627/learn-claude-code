# s12_worktree_task_isolation.py 執行流程詳解

## 問題與解決方案摘要

### 問題

到了 s11，Agent 已能自主認領並完成任務，但所有任務共享同一個工作目錄。當兩個 Agent 同時重構不同模組時，若都修改了 `config.py`，**未提交的變更會互相污染**，誰也無法乾淨地回滾。任務看板管理「做什麼」，但不管理「在哪做」。

### 解決方案

為每個任務建立獨立的 Git Worktree 目錄，以任務 ID 關聯兩者。控制平面（`.tasks/`）記錄「做什麼」；執行平面（`.worktrees/`）記錄「在哪做」。`worktree_create()` 將任務推進至 `in_progress` 並建立 Git Worktree；`worktree_remove()` 刪除目錄、完成任務、觸發事件。所有生命週期步驟寫入 `.worktrees/events.jsonl`，供外部系統訂閱與審計。

---

本章介紹了「物理隔離（Physical Isolation）」。利用 Git Worktree 讓 Agent 可以在並行的、互不干擾的目錄中執行多項任務。當輸入 Prompt 為 `"Work on auth refactor and UI redesign in parallel."` 時，流程如下：

### 第一階段 (建立平行通道)

1.  **使用者請求**：模型需要平行處理兩個大任務。
2.  **分配工作通道 (Lane Allocation)**：
    *   模型呼叫 `worktree_create("auth-lane", task_id=12)`。
    *   Git 在 `.worktrees/auth-lane/` 建立一個全新的簽出目錄。
3.  **重複動作**：為 UI 重構建立另一個通道 `worktree_create("ui-lane", task_id=13)`。

---

### 第二階段 (平行執行)

1.  **隔離執行**：模型利用 `worktree_run` 指令。
    *   `worktree_run("auth-lane", "npm run refactor")`：該指令只在 `auth-lane` 目錄下執行，完全不影響主目錄。
2.  **狀態觀察**：模型呼叫 `worktree_status` 觀察各個通道的檔案變動情況。
3.  **事件記錄**：所有的建立與執行動作都會被自動記錄在 `events.jsonl` 中，供外部審計或 UI 顯示。

---

### 第三階段 (結案與清理)

1.  **任務完成**：在兩個通道中的工作皆完成。
2.  **選擇結案方式**：
    *   模型呼叫 `worktree_remove("auth-lane", complete_task=True)`。
    *   Git 移除該 Worktree，並自動更新 `.tasks/task_12.json` 為 `completed`。
3.  **保留研究**：模型可以呼叫 `worktree_keep("ui-lane")`，在 index 中標記為 `kept`，這表示該通道不會被移除，供之後查閱。

### 核心總結
本章展示了 **「目錄即通道（Directory as a Lane）」** 的實作。透過將任務綁定到獨立的 Worktree，我們解決了多任務環境下的檔案衝突與狀態混亂問題，實現了生產等級的並行任務執行能力。

---

## Worktree 隔離機制詳解

### 控制平面 vs 執行平面

s12 將任務管理分為兩個平面：

| 平面 | 目錄 | 職責 |
| :--- | :--- | :--- |
| **控制平面** | `.tasks/` | 記錄「做什麼」：任務的 ID、狀態、依賴、擁有者 |
| **執行平面** | `.worktrees/` | 記錄「在哪做」：每個任務對應的獨立 Git Worktree 目錄 |

兩個平面透過 `worktree` 欄位雙向綁定：
- `task_1.json` 中有 `"worktree": "auth-refactor"`。
- `.worktrees/index.json` 中有 `"task_id": 1`。

`bind_worktree()` 方法在建立 Worktree 時同時更新兩側的狀態，確保兩個平面始終保持一致。

### Git Worktree 的核心原理

`git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD` 在**同一個 Git 儲存庫**中建立一個額外的工作目錄，並將其指向一個獨立分支。這讓多個任務可以：
- 修改相同的檔案（各自在自己的分支上，互不污染）。
- 隨時可以對各自的改動進行 `git diff` 或 `git commit`。

相比「複製整個專案目錄」，Worktree 幾乎不佔用額外磁碟空間，因為它共享同一個 `.git` 目錄的物件儲存。

### 事件流 (`events.jsonl`) 的完整類型

每個生命週期步驟都會記錄一條事件，便於外部系統（如 CI、監控面板）訂閱和審計：

| 事件類型 | 觸發時機 |
| :--- | :--- |
| `worktree.create.before` | `git worktree add` 執行前 |
| `worktree.create.after` | Worktree 成功建立並完成綁定後 |
| `worktree.create.failed` | 建立失敗（如分支名衝突） |
| `worktree.remove.before` | `git worktree remove` 執行前 |
| `worktree.remove.after` | 成功移除，任務同步標記完成 |
| `worktree.remove.failed` | 移除失敗（如目錄有未提交的改動） |
| `worktree.keep` | Worktree 被標記為保留（不移除） |
| `task.completed` | 任務狀態更新為 `completed` |

### `keep` vs `remove` 的選擇

| 操作 | 行為 | 適用場景 |
| :--- | :--- | :--- |
| `worktree_keep(name)` | 在 `index.json` 標記為 `kept`，保留目錄 | 需要事後人工查閱或手動合併 |
| `worktree_remove(name, complete_task=True)` | 刪除目錄 + 完成任務 + 觸發事件 | 自動化流程中的正常完成 |

### 崩潰恢復機制

若 Agent 程序意外崩潰，可以從磁碟重建現場：

1. 掃描 `.tasks/` 目錄 → 恢復所有任務的狀態。
2. 讀取 `.worktrees/index.json` → 恢復 Worktree 清單。
3. 比對兩者的 `task_id` 綁定關係 → 識別哪些任務在哪個目錄執行中。
4. 讀取 `.worktrees/events.jsonl` → 確認最後一個成功的事件，判斷哪些操作需要重做。

**核心原則**：對話記憶（messages）是易失的；磁碟狀態是持久的。崩潰後重建依靠的是磁碟，而非模型的記憶。

---

## 常見問題 Q&A

### 1. s12 看起來只是讓 Agent 分別在不同 Branch 上工作，最後再 merge，這在本質上有區別嗎？

這個觀察很準確——每個 Worktree 確實對應一個獨立分支（`wt/auth-refactor`）。但 Worktree 與「叫 Agent 自己切換 Branch」之間有一個**根本性的操作差異**：

**Branch 切換（沒有 Worktree）的問題：**

若多個 Agent 執行緒共用同一個工作目錄，要在不同任務間切換必須執行 `git checkout`：

```
Agent A 工作中（auth 任務，有未 commit 的改動）
Agent B 想切到 ui 分支 → 必須等 A 先 commit 或 stash
→ 兩個 Agent 必須排隊，無法真正並行
```

更嚴重的問題：兩個 Agent 同時對同一個實體目錄呼叫 `bash` 或 `edit_file`，寫入的是**同一份檔案**。Agent A 改了 `config.py` 還沒存檔，Agent B 也讀了 `config.py` 開始改——結果是資料競爭（Race Condition），誰後寫誰贏，前一個改動消失。

**Worktree 的本質差異：同時存在的實體目錄**

```
.worktrees/
├── auth-lane/    ← Agent A 的專屬目錄，有自己的 config.py
└── ui-lane/      ← Agent B 的專屬目錄，有自己的 config.py
```

兩個目錄**同時存在於磁碟上**，各自獨立，共享同一個 `.git` 物件庫（幾乎不佔額外空間）。Agent A 的 `worktree_run("auth-lane", "vim config.py")` 和 Agent B 的 `worktree_run("ui-lane", "vim config.py")` 操作的是**不同的實體檔案**，物理上不可能衝突。

**類比：**

| 方式 | 類比 |
| :--- | :--- |
| Branch 切換 | 兩個人共用同一張辦公桌，輪流使用，每次換人前要收拾桌面 |
| Git Worktree | 兩個人各有自己的辦公桌，同時工作，最後把成果合併到主桌 |

**三個具體差異：**

1. **真正的並行 vs 排隊切換**：Worktree 讓多執行緒 Agent 可以同時對各自目錄執行指令，不需要任何鎖或排隊。Branch 切換本質上是序列操作。

2. **未 commit 的改動安全**：Worktree A 中到一半的修改對 Worktree B 完全不可見，也不影響主分支。Branch 模式下，未 commit 的改動會阻塞其他 Agent 的切換操作。

3. **Agent 不需要追蹤「我現在在哪個 Branch」**：`worktree_run("auth-lane", cmd)` 讓 Agent 只需指定「在哪個通道執行」，底層目錄路由由系統處理。Branch 模式下 Agent 必須自己管理 `git checkout` 的時機與狀態。

**結論：** 概念上兩者都是「隔離的 Branch 工作」，但 Worktree 解決的是**並行執行的物理隔離問題**，不只是邏輯上的分支隔離。沒有 Worktree，多 Agent 並行修改同一個 repo 就是在等待資料競爭發生；有了 Worktree，每個 Agent 有自己的沙盒目錄，並行才真正安全。

### 2. 在 Claude Code 中是否有類似 s12 的機制，幫助 Agent 達到並行隔離作業？

**有，且是內建的。** Claude Code 的 `Agent` 工具支援 `isolation: "worktree"` 參數，這正是 s12 概念的生產實作。

**如何使用：**

在 `.claude/agents/` 的 Agent 定義檔中加入 `isolation` 欄位：

```yaml
---
name: refactor-agent
description: Refactors code in an isolated worktree to avoid conflicts with parallel agents.
tools: Read, Edit, Write, Bash
isolation: worktree
---
```

或在 Prompt 中明確要求：

```
Use worktrees for your agents when working in parallel.
```

**運作方式：**

每個帶有 `isolation: "worktree"` 的 Subagent 被 spawn 時，Claude Code 會在 `.claude/worktrees/` 下自動建立一個獨立的實體目錄，指向一條新的分支。多個 Subagent 同時執行時，各自在自己的目錄中讀寫，物理上不可能發生檔案衝突。

**清理行為：**

| Subagent 完成狀態 | 對應行為 |
| :--- | :--- |
| **沒有任何檔案變更** | Worktree 目錄與分支自動刪除 |
| **有變更或 commit** | Worktree 保留，回傳 worktree 路徑與分支名供後續處理（如 code review、merge） |

**注意：Agent Teams 不會自動套用 Worktree 隔離。** 官方文件對 Agent Teams 的衝突建議是「確保每個隊員負責不同的檔案集合」，不是自動 Worktree 隔離。Worktree 隔離目前只對 Subagent（`Agent` 工具）有效，若要在 Agent Teams 中使用，需手動為每個隊員定義 Agent 檔案並加上 `isolation: worktree`。

**`isolation: "worktree"` vs s12 WorktreeManager 的對比：**

| 面向 | Claude Code 內建 | s12 WorktreeManager |
| :--- | :--- | :--- |
| **生命週期** | 與 Subagent 執行綁定，完成後自動清理 | 與 Task 生命週期綁定，獨立於 Agent session |
| **狀態追蹤** | 隱式（Claude Code 管理） | 顯式（`.tasks/` + `index.json` + `events.jsonl`） |
| **崩潰恢復** | 無（session 結束後狀態遺失） | 有（可從磁碟重建任務與 Worktree 清單） |
| **事件可觀測性** | 透過 Hooks（`WorktreeCreate/Remove`） | 內建 `events.jsonl` JSONL 串流 |
| **設定成本** | 低（加一行 `isolation: worktree`） | 高（需要自訂 Python 腳本） |

**實用結論：**

*   **快速並行、不需崩潰恢復** → 使用 `isolation: "worktree"` 內建功能，零設定成本。
*   **長時間任務、需要跨 session 持久化** → 採用 s12 的 WorktreeManager，任務狀態寫入磁碟，崩潰後可完整重建。

s12 是 Claude Code 內建機制的**加強版**，增加了任務板雙向綁定、持久事件日誌與崩潰恢復能力，這也是從原型走向生產系統時必然需要補強的部分。

