好的，下面结合 packages/core 目录的源码结构，对 PageAgentCore 的详细原理进行深入分析：

---

## 1. 目录结构与核心文件

- PageAgentCore.ts  
  PageAgentCore 的主类实现，包含主循环、事件系统、与 LLM/工具/PageController 的集成。
- `src/types.ts`  
  定义了 agent 的核心类型（如 AgentConfig、AgentStatus、HistoricalEvent、工具类型等）。
- `src/prompts/system_prompt.md`  
  系统提示词模板，作为 LLM 推理的基础上下文。
- `src/tools/`  
  内置工具的注册与实现（如点击、输入、等待、done等）。
- `src/utils/`  
  常用工具函数（如 uid 生成、等待、断言、normalizeResponse 等）。

---

## 2. PageAgentCore 的核心原理

### 2.1 生命周期与主循环

PageAgentCore 以“ReAct Agent Loop”为核心思想，每次任务执行都经历 observe（观察）→ think（思考/LLM）→ act（执行动作）→ loop（循环）：

- **初始化**：构造时注入 PageController、LLM、工具集、配置等，注册 LLM 事件监听。
- **execute(task: string)**：主入口，接收用户任务，初始化状态，进入循环。
- **每一步循环**：
  1. **observe**：通过 PageController 获取页面状态，检测 URL 变化、累计等待等，生成 observation。
  2. **prompt 组装**：拼接 system prompt、用户请求、历史、页面状态等，形成 LLM 输入。
  3. **think**：调用 LLM，输出结构化动作（通过宏工具，包含反思、记忆、下一目标、action）。
  4. **act**：查找对应工具（如 click、fill、wait、done），执行 DOM 操作（由 PageController 实现）。
  5. **历史记录**：每步的反思、动作、结果、原始 LLM 响应都持久化到 history。
  6. **事件派发**：每步状态、历史、activity 通过 EventTarget 派发，供 UI/外部监听。
  7. **终止条件**：遇到 done、异常或超步数即终止，触发 onDone。

### 2.2 工具系统（src/tools/）

- 所有可被 LLM 调用的动作都以“工具”注册（如 click、fill、wait、done、ask_user 等）。
- 工具定义了输入/输出 schema、描述、执行函数。
- LLM 输出 action 字段，指定要调用的工具及参数，core 自动查找并执行。
- 支持自定义工具注入、禁用、扩展。

### 2.3 事件与历史系统

- **EventTarget**：PageAgentCore 继承自 EventTarget，支持 statuschange、historychange、activity、dispose 等事件。
- **history**：每步的 step、observation、error、user_takeover 等都记录在 history 数组，作为 agent 的“短期记忆”，每步 prompt 都会带上。
- **activity**：用于 UI 实时反馈（如 thinking、executing、executed、retrying、error）。

### 2.4 prompt 组装与 LLM 调用

- **system_prompt.md**：作为 LLM 的基础指令，支持多语言切换。
- **#assembleUserPrompt**：拼接 instructions、agent_state（用户请求、步数、时间）、agent_history（历史步骤、observation）、browser_state（页面结构/内容）。
- **LLM 调用**：通过 llms 包的 LLM 类，支持重试、异常捕获、normalizeResponse 结构化输出。

### 2.5 与 PageController 的协作

- PageController 负责所有 DOM 操作（如查找元素、输入、点击、滚动、等待等）。
- core 只负责调度和数据流转，不直接操作 DOM。
- 每步 act 阶段，调用工具时会通过 PageController 执行具体页面动作，并收集反馈。

### 2.6 用户介入与扩展性

- 支持 ask_user 工具和 onAskUser 回调，LLM 可主动请求用户输入。
- 支持自定义工具、系统提示、页面内容转换、任务前后钩子等高度扩展。

---

## 3. 伪代码与数据流

```typescript
// 1. 用户输入任务
core.execute('帮我填写表单')

// 2. 主循环
while (step < maxSteps) {
  // 观察
  browserState = pageController.getBrowserState()
  // prompt 组装
  prompt = systemPrompt + userRequest + history + browserState
  // LLM 推理
  result = llm.invoke(prompt, macroTool)
  // 解析动作
  action = result.toolResult.input.action
  // 执行动作
  tool = tools.get(action.name)
  toolResult = await tool.execute(action.input)
  // 记录历史
  history.push({ ... })
  // 事件派发
  dispatchEvent('historychange')
  // 终止判断
  if (action.name === 'done') break
}
```

---

## 4. 总结

PageAgentCore 以“事件驱动+ReAct循环”为核心，解耦 LLM 推理、页面操作、历史记忆、UI 展示等各层，具备高度可扩展性和可观测性。其核心优势在于：
- 结构化历史与反思，提升 LLM 多轮推理能力
- 工具化动作体系，便于扩展和安全控制
- 事件系统，方便 UI/外部集成
- 与 PageController、llms、UI 等模块松耦合协作

如需进一步分析某个子模块（如 tools、prompt 组装、事件系统等），可继续指定！