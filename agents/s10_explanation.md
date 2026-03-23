# s10_team_protocols.py 執行流程詳解

本章介紹了結構化協作協議。透過 `request_id` 關聯模式，讓模型之間可以進行可靠的「握手」確認。當 Lead 決定關閉隊員 `Alice` 時，流程如下：

### 第一階段 (發起請求)

1.  **Lead 決定終止**：
    *   Lead 呼叫 `shutdown_request("Alice")`。
    *   系統自動生成 `request_id`（例如：`req_456`）。
2.  **存入追蹤器 (Tracker)**：
    *   `shutdown_requests["req_456"]` 狀態設為 `pending`。
3.  **發送協議訊息**：
    *   將帶有 `request_id` 的關機請求訊息寫入 `Alice.jsonl`。

---

### 第二階段 (接收與決策)

1.  **Alice 接收協議**：
    *   `Alice` 讀取信箱，看到類型為 `shutdown_request` 的協議訊息。
2.  **回覆協議**：
    *   `Alice` 呼叫 `shutdown_response` 工具。
    *   帶入相同的 `request_id`（`req_456`）並設 `approve: true`。
3.  **回傳確認**：確認訊息被發送回 Lead。

---

### 第三階段 (執行與結案)

1.  **Lead 匹配回覆**：
    *   Lead 端的 `MessageBus` 收到回覆。
    *   透過 `request_id` 匹配，將 `shutdown_requests["req_456"]` 標記為 `approved`。
2.  **安全退場**：`Alice` 的執行緒在完成最後一次工具回傳後，偵測到已同意關機，正式停止循環。

### 核心總結
本章展示了 **「狀態關聯協議」** 的實作。透過 `request_id` 模式，我們可以建立起可靠的對話握手機制，解決了非同步通訊中「訊息順序錯亂」或「回覆對象不明」的問題。

---

## 常見問題 Q&A
