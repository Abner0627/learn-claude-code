# s05_skill_loading.py 執行流程詳解

本章介紹了「隨選知識（On-demand Knowledge）」的注入機制，透過兩層式注入 (Two-layer Injection) 來優化系統提示詞 (System Prompt) 的大小。

### 初始化階段 (System Prompt 構建)

1.  **第一層注入 (Layer 1)**：
    *   `SkillLoader` 掃描 `skills/` 目錄。
    *   將所有技能的「名稱」與「簡短描述」注入 System Prompt。
    *   範例：`"pdf: Process PDF files..."`。

---

### 第一輪迴圈 (Iteration 1: 請求詳細知識)

1.  **使用者請求**：`"Extract all tables from data.pdf."`
2.  **模型發覺需要特定技能**：模型在提示詞中看到了 `pdf` 這個技能，但並不知道具體操作 SOP。
3.  **發出 `load_skill` 請求**：
    ```json
    {
        "type": "tool_use",
        "name": "load_skill",
        "input": {"name": "pdf"}
    }
    ```

---

### 第二輪迴圈 (Iteration 2: 注入詳細內容)

1.  **第二層注入 (Layer 2)**：
    *   工具回傳 `SKILL.md` 的完整 Body 內容（包含詳細步驟、最佳實踐）。
    *   這些詳細指引以 `tool_result` 的形式傳入 `messages`。
2.  **獲得新知**：模型現在完整理解了處理 PDF 的 SOP（例如：先讀取 Meta、使用特定套件解析等）。
3.  **開始工作**：模型依據新注入的知識，決定下一步的工具呼叫。

### 核心總結
本章展示了 **「延遲加載知識」** 的價值。透過先傳遞 metadata，只有在模型真正需要時才注入完整 SOP，有效地將單次對話的 Token 成本降至最低。

---

## 常見問題 Q&A
