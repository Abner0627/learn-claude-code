# s11_autonomous_agents.py 執行流程詳解

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
