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

## 常見問題 Q&A
