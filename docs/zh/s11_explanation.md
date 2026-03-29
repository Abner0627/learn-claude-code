# s11_autonomous_agents.py 執行流程詳解

## 問題與解決方案摘要

### 問題

s09–s10 中，隊員只在被明確指派時才會行動。領導**必須為每個隊員撰寫 prompt**，任務看板上 10 個未認領的任務需要手動分配，這**無法擴展**。真正的自治需要隊員能自行掃描任務看板、認領無人執行的任務、完成後再找下一個。此外，**上下文壓縮後 Agent 可能忘記自己的身份**。

### 解決方案

隊員迴圈分為兩個階段：`WORK`（最多 50 輪）與 `IDLE`（最多 60 秒）。空閒階段持續輪詢收件匣和任務看板。`scan_unclaimed_tasks()` 找出 `pending`、無 `owner`、未被阻塞的任務，**自動認領**後恢復 `WORK` 階段。**身份重注機制**：上下文過短時，在開頭插入 `<identity>` 區塊，確保模型記得自己的角色與當前任務。60 秒空閒超時後自動關機。

---

本章介紹了 Agent 的「自主性（Autonomy）」。模型不再只是被動回應 Lead 的指令，而是具備「閒置輪詢」與「任務認領」的能力。

### 第一階段 (任務認領)

1.  **閒置狀態**：隊員 `Alice` 完成前一個任務後，進入 `idle` 循環。
2.  **主動掃描任務板**：
    *   `Alice` 呼叫 `scan_unclaimed_tasks` 讀取 `.tasks/`。
    *   發現一個任務：`#5: Add logging to API`。
3.  **認領任務**：
    *   `Alice` 呼叫 `claim_task(5)`。
    *   `.tasks/task_5.json` 被更新為 `owner: Alice`, `status: in_progress`。

---

### 第二階段 (身份重注與恢復)

1.  **恢復工作狀態**：
    *   領取成功後，`Alice` 從 `idle` 轉回 `working`。
2.  **身份重注 (Identity Re-injection)**：
    *   系統自動在 `messages` 中注入 `<identity>` 區塊：`"You are 'Alice', role: coder, team: default..."`。
    *   這確保模型即使在長時間閒置或對話歷史被壓縮後，仍保有明確的自我定義與角色目標。

---

### 第三階段 (執行與回報)

1.  **執行任務**：`Alice` 像往常一樣呼叫 `edit_file` 修補程式碼。
2.  **完成回報**：完成後將狀態設為 `completed` 並再次進入 `idle` 循環。

### 核心總結
本章展示了 **「自主驅動（Self-driven）」** 的 Agent 系統。透過「任務板 polling」與「身份重注」機制，模型不再只是一個指令接收器，而是能主動獲取環境資訊並持續運作的數位隊員。

---

## 自主性機制詳解

### WORK / IDLE 雙相設計

`_loop` 函數分為兩個明顯的階段：

**WORK 相**（最多 50 輪工具呼叫）：
- 模型像 s09 一樣呼叫工具、執行任務。
- 一旦 `stop_reason != "tool_use"` 或模型主動呼叫 `idle` 工具，退出 WORK 相，進入 IDLE 相。

**IDLE 相**（最多 60 秒，每 5 秒輪詢一次，共 12 次）：
1. 讀取信箱 → 有新訊息？→ 恢復 WORK 相。
2. 掃描任務看板 → 有未認領任務？→ 認領並恢復 WORK 相。
3. 以上皆無，超過 60 秒 → 自動 SHUTDOWN。

這種設計讓隊員既不會因為一時沒有工作就立刻退出，也不會無限期佔用資源。

### 任務認領的三個條件

`scan_unclaimed_tasks()` 只回傳同時滿足以下三個條件的任務：

1. `status == "pending"`：尚未開始執行。
2. `owner` 為空字串：沒有隊員已認領。
3. `blockedBy == []`：所有前置依賴已完成，沒有阻塞。

這三個條件確保了：不會重複認領同一任務、不會搶先執行尚未就緒的任務、完全尊重 s07 中建立的依賴關係。

### 身份重注 (Identity Re-injection) 的觸發機制

上下文壓縮（s06）後，`messages` 會被縮減為 2 條（摘要 + 確認）。s11 透過判斷 `len(messages) <= 3` 來偵測壓縮事件，並在 `messages` 開頭插入 `<identity>` 區塊：

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

這解決了一個微妙問題：模型的「身份感」來自系統提示，但在超長任務中，壓縮後的 `messages` 可能讓模型困惑「自己是誰」、「目前在做什麼」。身份重注是針對這個問題的最小化解法。

### 自主性的演進路徑

| 章節 | 工作觸發方式 | 任務來源 |
| :--- | :--- | :--- |
| s09 | Lead 主動發訊息委派 | 僅來自 Lead |
| s10 | 同 s09，但加入握手協議 | 僅來自 Lead（但隊員可拒絕） |
| s11 | **自主掃描任務看板** | 來自 Lead 或**任務看板** |

s11 的關鍵突破在於：隊員不再需要等待 Lead 的指令，而是主動從共享的任務看板中找工作，實現了真正的自組織（Self-organizing）。

---

## 常見問題 Q&A

### 1. 在 Claude Code 中，如何讓 Agent 執行持續數小時的複雜任務、自行組建團隊並讓成員自行認領任務？

這個需求融合了 s09（團隊）、s10（協議）、s11（自主認領）三個概念。先說清楚一個前提：

**Claude Code 目前沒有原生的 s11 式自主輪詢機制。** s11 的 IDLE 輪詢、`_claim_lock`、60 秒超時關機，都是自訂 Python 程式的行為，不是 Claude Code 內建的。

但你可以用以下兩條路達到「近似效果」：

---

**路線 A：直接執行 `s11_autonomous_agents.py`（最接近 s11 原型）**

這是最完整的做法。你只需要：

1. 預先建立任務看板（`.tasks/task_*.json`），填入所有子任務。
2. 執行 `python s11_autonomous_agents.py`，透過互動介面對 Lead 說：
   ```
   Spawn 3 teammates: alice (coder), bob (tester), carol (reviewer).
   The task board has 10 tasks ready. Let them work autonomously.
   ```
3. Lead 會 spawn 三個執行緒，每個隊員進入 WORK/IDLE 循環，自行掃描任務板認領工作，直到所有任務完成或 60 秒無新工作後自動關機。

這條路的優點是行為完全可預期，缺點是需要自己管理 Python 環境與任務 JSON。

---

**路線 B：在 Claude Code 中透過結構化設定模擬自主行為（推薦）**

Claude Code 本身的穩定多 Agent 能力是 s04 式的 Subagent（一次性、上下文隔離）。要達到長時間、自主、類團隊的效果，需要三層設定配合：

**第一層：在 `.claude/agents/` 定義角色**

為每個專責角色建立 Agent 定義檔：

```yaml
# .claude/agents/coder.md
---
name: coder
description: Use proactively for implementing features, fixing bugs, and writing code. Works autonomously on assigned tasks.
tools: Read, Edit, Write, Bash, Glob, Grep
---
You are a focused coder. When given a task, implement it completely before stopping.
Check the TodoWrite list for your assigned tasks. Mark each task completed when done.
```

```yaml
# .claude/agents/reviewer.md
---
name: reviewer
description: Use after code changes to review quality, security, and correctness.
tools: Read, Glob, Grep, Bash
---
You are a code reviewer. Read the changed files and report issues clearly.
```

**第二層：在 `CLAUDE.md` 定義任務分解策略**

```markdown
## 複雜任務執行策略
- 收到大型任務時，先用 TodoWrite 將任務分解為獨立子任務
- 標記每個任務的負責角色（coder / tester / reviewer）
- 使用 Agent 工具平行派發給對應角色的 subagent
- subagent 完成後，將結果摘要更新回對應的 Todo 項目
- 所有子任務完成後，召喚 reviewer 進行最終審查
```

**第三層：給 Claude Code 的初始 Prompt（這是關鍵）**

```
This is a multi-hour task. Please:
1. Break it into independent subtasks using TodoWrite
2. Spawn parallel agents for each role (coder, tester, reviewer)
3. Have each agent self-assign tasks from the todo list by role
4. Do not wait for my input between tasks — work autonomously until all todos are completed
5. Report back only when everything is done or if you hit a blocker that requires a decision

Task: [你的複雜任務描述]
```

---

**兩條路線的比較：**

| 面向 | 路線 A（s11 腳本） | 路線 B（Claude Code） |
| :--- | :--- | :--- |
| **自主程度** | 完整（真正輪詢、自動認領） | 部分（Subagent 是一次性的，不會輪詢） |
| **任務板** | `.tasks/` JSONL 檔案 | TodoWrite（記憶體內） |
| **跨任務記憶** | 有（持久執行緒） | 無（每個 subagent 上下文獨立） |
| **設定成本** | 低（直接跑腳本） | 高（需要設定 agents/ 和 CLAUDE.md） |
| **長時間穩定性** | 高（有超時保護與身份重注） | 中（受 Claude Code session 限制） |

**真正的長時間任務（數小時）建議：**

混合兩者——用 Claude Code 做任務分解與初始 Prompt 設計，產出結構化的 `.tasks/` 看板後，切換到 `s11_autonomous_agents.py` 執行真正的自主多執行緒工作。Claude Code 負責「想清楚要做什麼」，s11 負責「不間斷地把它做完」。

### 2. 加入自主輪詢 Skill 並在 Prompt 中提及，能達到連續工作數小時的效果嗎？有沒有更有效率且節省 Token 的做法？

**Skill 加輪詢指令可以運作，但有一個根本性的 Token 成本問題。**

**Skill 輪詢方案的限制：**

Skill 本質上是 Prompt 注入——它告訴模型「按照這個工作流反覆執行」。Claude Code 的對話循環本身就是「輪詢」，模型每次呼叫工具後繼續執行即為一次迭代。這在短期內有效，但存在一個無法迴避的問題：

```
每次 LLM 呼叫的 Token 成本 = 完整對話歷史 + 工具定義 + 新輸入
```

隨著任務推進，對話歷史不斷累積，每一輪呼叫要重新處理的 Token 越來越多。一個 4 小時的任務，後期每次呼叫的成本可能是初期的 10 倍以上。

**Claude Code 的自動 Prompt Caching（降低重複成本）：**

Claude Code 會自動對靜態內容（System Prompt、工具定義、CLAUDE.md）啟用 Prompt Caching，快取命中的 Token 費用僅為原本的 **0.1 倍（節省 90%）**，無需任何設定。這能大幅降低重複部分的成本，但無法阻止對話歷史本身的增長。

**更有效率且節省 Token 的做法：主代理 + 隔離式子代理（s04 模式）**

這是 Claude Code 官方文件明確推薦的多小時任務架構：

```
主代理（Main Agent）
│  context 保持精簡
│  只看任務清單與摘要
│
├─► 子代理 A（獨立 context，最多 200K）
│     執行高噪音操作（掃描、測試、分析）
│     完成後只回傳摘要 ──► 主代理 context +1 條記錄
│
├─► 子代理 B（獨立 context，最多 200K）
│     執行另一個子任務
│     完成後只回傳摘要 ──► 主代理 context +1 條記錄
│
└─► 子代理 C ...
```

關鍵在於：**詳細的工具呼叫過程全部留在子代理的獨立 context 中，主代理只看結論**。這與 s04 的「Context Discarding」原理完全相同——子代理做完即棄，垃圾回收帶走所有中間步驟。

**三種方案的 Token 效率比較：**

| 方案 | Token 效率 | Context 行為 | 適用場景 |
| :--- | :--- | :--- | :--- |
| **Skill 輪詢（單一長對話）** | 差 | 歷史無限累積，後期每輪成本暴增 | 短期輪詢（查狀態、等部署） |
| **主代理 + 隔離子代理** | **最佳** | 主代理 context 保持精簡，子代理各自隔離 | 多小時自主任務 |
| **多次獨立 `claude -p` 呼叫** | 中等 | 每次重新載入環境，無歷史累積但也無跨任務記憶 | CI/CD 管道、無狀態任務 |

**實際的初始 Prompt 設計（主代理 + 子代理模式）：**

```
This is a multi-hour autonomous task. Follow this workflow:

1. Use TodoWrite to break the work into independent subtasks (each <30 min)
2. For each subtask, use the Agent tool to spawn an isolated subagent
3. The subagent handles ALL the detailed work; you only receive its summary
4. After each subagent completes, update the corresponding Todo to done
5. Continue until all todos are completed — do not ask me for input unless you hit a hard blocker

Never do high-volume operations (file scanning, test runs, log analysis) in the main conversation.
Always delegate those to subagents.

Task: [你的複雜任務描述]
```

**關於 `/loop` Skill（另一個選項）：**

`/loop` 每次迭代仍在**同一個 session context** 中執行，context 會持續累積，適合短期輪詢（如每 5 分鐘確認部署狀態），不適合多小時自主工作。若每次迭代都派發子代理處理實際工作，`/loop` 可以作為觸發機制，但主要的 Token 節省來自子代理隔離，不是 `/loop` 本身。

**結論：Skill 輪詢 + 子代理隔離才是完整解法**

| 層次 | 機制 | 作用 |
| :--- | :--- | :--- |
| **工作流控制** | Skill（輪詢指令）| 告訴主代理「反覆找任務、執行、回報」 |
| **Token 隔離** | Agent 工具（子代理）| 讓高噪音操作不污染主代理 context |
| **成本優化** | 自動 Prompt Caching | 靜態內容（System Prompt、工具定義）節省 90% |

三者缺一，長時間任務的成本控制都是不完整的。

