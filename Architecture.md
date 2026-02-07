# Dexter 架構文件

## 系統架構概述
Dexter 是以 CLI 互動為核心的研究代理，入口在 `src/index.tsx`，啟動 Ink/React 介面後交由 `src/cli.tsx` 控制使用者輸入、模型選擇與 Agent 執行。整體採用「CLI UI → Agent 迭代 → 工具呼叫 → 最終回答」的流程，並透過 scratchpad 記錄工具結果與上下文。 

## 技術棧
- **Runtime / Tooling**：Bun（`bun run`/`bun test`），TypeScript（`tsc`）
- **CLI UI**：Ink + React
- **LLM / Tooling**：LangChain（OpenAI/Anthropic/Gemini/Ollama 等 provider）
- **其他**：Playwright（瀏覽器工具）、Zod（schema）、dotenv（環境變數載入）

## 主要模組與職責
- **CLI / 互動層**：
  - `src/index.tsx`：啟動入口
  - `src/cli.tsx`：輸入處理、顯示歷史、觸發 Agent
  - `src/hooks/useAgentRunner.ts`：Agent 執行、事件流管理
- **Agent / 流程層**：
  - `src/agent/agent.ts`：迭代式工具呼叫與最終回答
  - `src/agent/prompts.ts`：system/iteration/final prompt 組裝
  - `src/agent/scratchpad.ts`：JSONL 形式記錄工具結果與上下文清理
- **LLM / Provider 層**：
  - `src/model/llm.ts`：模型選擇、system prompt 注入、工具綁定與 Anthropic cache_control
- **工具 / Skill 層**：
  - `src/tools/registry.ts`：工具註冊與描述注入
  - `src/tools/descriptions/*`：工具描述
  - `src/tools/finance/*`：財金工具與 LLM 路由提示詞
  - `src/skills/*`：以 SKILL.md 管理的工作流
- **對話上下文**：
  - `src/utils/in-memory-chat-history.ts`：對話摘要、相關性挑選

## 資料流向與互動關係
1. **CLI 接收輸入** → `useAgentRunner` 建立執行流程並保存對話歷史。
2. **Agent 建立** → `Agent.create()` 依模型產生 system prompt 與工具清單。
3. **迭代執行** → LLM 回應含 tool calls 時，執行工具並寫入 scratchpad。
4. **上下文管理** → 超過 token 閾值時清理最舊工具結果，但保留完整 JSONL 記錄。
5. **最終回答** → 將 scratchpad 內容合併為 final prompt，生成最終回覆。

## 提示詞（Prompts）位置地圖
- **Agent 核心 prompt**：`src/agent/prompts.ts`（system/iteration/final）
- **工具描述注入**：`src/tools/descriptions/*` + `src/tools/registry.ts`
- **金融工具路由 prompt**：
  - `src/tools/finance/financial-search.ts`
  - `src/tools/finance/financial-metrics.ts`
  - `src/tools/finance/read-filings.ts`
- **Skills prompt**：`src/skills/*/SKILL.md` + `src/skills/registry.ts`
- **多輪摘要 / 相關性 prompt**：`src/utils/in-memory-chat-history.ts`
- **Evals prompt**：`src/evals/run.ts`

## 重要設計決策與原因
- **Scratchpad 以 JSONL 持久化**：保留完整工具輸出，便於除錯與最終回答時回填上下文。
- **工具描述注入 system prompt**：讓 LLM 能理解工具用途與限制，降低誤用。
- **LLM 路由金融工具**：`financial_search`/`financial_metrics` 使用 LLM 進行子工具選擇，避免硬編排規則。
- **Anthropic cache_control**：system prompt 標記為 ephemeral，降低重複呼叫成本。
- **Skill 系統**：以 SKILL.md 保存流程，允許專業任務（如 DCF）在提示詞層擴充。

## 程式碼範例
以下為 Agent 啟動與執行流程的精簡示意：

```ts
const agent = Agent.create({ model: 'gpt-5.2', maxIterations: 10 });
for await (const event of agent.run(query, inMemoryChatHistory)) {
  // CLI 依 event 更新 UI 或顯示工具執行狀態
}
```
