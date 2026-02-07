# 實作經驗整理

## 錯誤與解決方法
- **未觀察到新增程式錯誤**：本次工作主要是文件整理，未觸及執行邏輯。
- **可預期的工具重試問題**：專案內已透過 tool call 限制與相似度警示避免重試迴圈，必要時可依警示改用其他工具或調整查詢語句。

## 需要注意的陷阱或限制
- **工具結果上下文可能被清理**：當上下文超出 token 閾值時，最舊的工具結果會被移出迭代 prompt（但 JSONL 仍完整保留），需留意長查詢的資訊完整性。
- **LLM 供應商差異**：Anthropic 會用 cache_control 包裝 system prompt，其餘供應商使用 ChatPromptTemplate，行為上可能出現差異。

## 最佳實踐與建議
- **盡量使用現有工具路由**：financial_search / financial_metrics 內部已以 LLM 路由適當子工具，避免自行拆解查詢導致重複呼叫。
- **Skill 工作流擴充**：需要固定流程（如估值）時，建議新增 SKILL.md，並由 skill tool 觸發，以確保提示詞與流程一致。

## 效能或安全性重要發現
- **Prompt 快取策略**：Anthropic 將 system prompt 標記為 ephemeral，可降低重複呼叫成本。
- **工具結果持久化**：scratchpad 以 JSONL 記錄完整結果，便於後續除錯或輸出整合。
- **API Key 管理**：敏感金鑰由 `.env`/環境變數提供，避免直接寫入程式碼或提交。
