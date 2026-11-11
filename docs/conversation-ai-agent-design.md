# AI Agent 設計原則與四大核心亮點 — 對話紀錄

> 上傳日期：2026-11-11
> Repository：jiwawa1006/DIC-_ProjectRefactor

## 使用者需求

使用者要求分析內容並製作 A4 橫式資訊圖表，主題聚焦於：

- AI Agent 設計原則與專業表達
- 四大核心亮點

## AI Agent 設計原則

### 1. Reliability over Reach｜可靠度優先於涵蓋度
不以「盡可能回答」為目標，而以「正確解決問題」為核心；證據不足時應主動拒答。

### 2. Program-Controlled Decision｜程式強控處置
由明確規則與 Agent 流程決定 `ANSWERED / ABSTAIN / CLARIFY / HANDOFF`，避免讓 LLM 自由決定高風險行為。

### 3. Evidence-Grounded Generation｜證據導向生成
回答必須建立在商店知識庫與檢索結果上，透過雙重證據閘門與生成後依據檢查降低政策幻覺。

### 4. Domain-Aware Reasoning｜領域感知推理
將台灣消費者保護法、退換貨政策、PCI DSS 與個資保護等領域知識納入 Agent 決策流程，而非僅依賴通用 LLM。

### 5. Safe Escalation｜安全轉接真人
當問題涉及權限、訂單、退款或 Agent 無法可靠處理時，應追問必要資訊或轉接人工，而不是強行生成答案。

### 6. Auditable & Traceable｜可稽核與可追溯
保留意圖分類、檢索證據、處置狀態、引用依據與轉接原因，使 Agent 行為能被驗證與分析。

## 四大核心亮點

### ① 可靠度優先｜Reliability over Reach
不為了回答而回答。以「正確解決問題」為目標，證據不足時 ABSTAIN，資訊不足時 CLARIFY，超出權限時 HANDOFF。

### ② 證據驅動｜Evidence-Grounded AI
讓每個回答都有依據。結合 BM25 + Dense + RRF 混合檢索、雙重證據閘門，以及生成後的引用與數字核驗，降低政策幻覺與錯誤回答。

### ③ 領域專精｜Domain-Aware Reasoning
不是通用 Chatbot，而是懂電商規範的 Agent。將台灣《消費者保護法》、退換貨規範、PCI DSS 與個資保護等領域知識整合至決策流程。

### ④ 可控可稽核｜Controlled & Auditable Agent
讓 AI 的決策可以被追蹤與驗證。由程式規則控制處置狀態，搭配來源引用、處置標籤與轉接原因代碼，使 Agent 行為具備可追溯、可驗證、可稽核的特性。

## 核心定位

> 專業 AI Agent 並非「更會回答」，而是能在領域知識、證據、規則與風險約束下，做出可解釋、可驗證、可轉接的可靠決策。

## ShopAssist 定位

> ShopAssist = 領域知識 × 證據檢索 × 程式強控 × 安全轉接，打造「可靠度優先」的電商 AI Agent。

## 備註

本文件整理本次對話中已明確提出與整理的內容，作為專題文件與後續 A4 橫式資訊圖表製作的文字基礎。
