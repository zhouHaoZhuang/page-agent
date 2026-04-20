# Page Agent 核心逻辑详解文档

## 1. 项目概述

Page Agent 是一个**浏览器自动化 AI Agent**，通过 LLM 控制浏览器完成用户任务（如导航、填表、数据提取等）。采用 **ReAct**（Reasoning + Action）循环模式。

## 2. 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                      PageAgent                              │
│  (packages/page-agent/src/PageAgent.ts)                    │
│  - extends PageAgentCore                                   │
│  - adds Panel UI                                          │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌──────────────┐  ┌──────────────┐
│  @page-agent/llms    │  │  @page-agent/core    │  │  @page-agent/ui      │
│  LLM 客户端       │  │  核心 Agent 逻辑  │  │  Panel UI         │
└─────────────────┘  └──────────────┘  └──────────────────┘
          │                   │
          │                   ▼
          │         ┌─────────────────────────┐
          │         │  @page-agent/page-controller │
          │         │  DOM 操作 + SimulatorMask   │
          │         └─────────────────────────┘
          │                   │
          ▼                   ▼
┌─────────────────┐  ┌─────────────────────────┐
│  OpenAI Client  │  │  FlatDomTree            │
│  (API 调用+重试) │  │  DOM 提取引擎           │
└─────────────────┘  └─────────────────────────┘
```

### 2.1 包说明

| 包 | 路径 | 说明 |
|-----|------|------|
| `page-agent` | `packages/page-agent/` | 主入口，带 UI Panel |
| `core` | `packages/core/` | `@page-agent/core` - 无 UI 的核心代理逻辑 |
| `llms` | `packages/llms/` | `@page-agent/llms` - LLM 客户端 |
| `page-controller` | `packages/page-controller/` | `@page-agent/page-controller` - DOM 操作 |
| `ui` | `packages/ui/` | `@page-agent/ui` - Panel 和 i18n |
| `extension` | `packages/extension/` | 浏览器扩展（WXT + React） |
| `website` | `packages/website/` | React 文档和落地页 |

## 3. 核心循环 (ReAct Agent Loop)

```
┌─────────────────────────────────────────┐
│           while (true)                    │
│  ┌───────────────────────────────────┐  │
│  │ 1. OBSERVE                         │  │
│  │    ↓ getBrowserState()            │  │
│  │    ↓ updateTree() 刷新 DOM        │  │
│  │    ↓ 生成页面状态描述              │  │
│  └───────────────────────────────────┘  │
│                   │                      │
│                   ▼                      │
│  ┌───────────────────────────────────┐  │
│  │ 2. THINK (LLM 调用)               │  │
│  │    ↓ 组装 Prompt                  │  │
│  │    ↓ 调用 LLM                     │  │
│  │    ↓ 解析工具调用                  │  │
│  └───────────────────────────────────┘  │
│                   │                      │
│                   ▼                      │
│  ┌───────────────────────────────────┐  │
│  │ 3. ACT (执行工具)                 │  │
│  │    ↓ click_element_by_index       │  │
│  │    ↓ input_text                   │  │
│  │    ↓ scroll                       │  │
│  │    ↓ wait                         │  │
│  └───────────────────────────────────┘  │
│                   │                      │
│                   ▼                      │
│  ┌───────────────────────────────────┐  │
│  │ 4. CHECK 完成条件                  │  │
│  │    ↓ action === 'done'?           │  │
│  │    ↓ step > maxSteps?             │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**关键文件：**
- 源码位置：`packages/core/src/PageAgentCore.ts`
- 主方法：`execute(task: string)`

## 4. 详细流程分解

### 4.1 OBSERVE 阶段

```typescript
// 获取浏览器状态
this.#states.browserState = await this.pageController.getBrowserState()

// 处理观察结果（累积等待时间警告、URL 变化检测、剩余步数警告）
await this.#handleObservations(step)
```

**BrowserState 结构：**

```typescript
interface BrowserState {
  url: string
  title: string
  header: string      // "Current Page: [title](url)\nPage info: ..."
  content: string     // 简化后的 HTML（交互元素列表）
  footer: string      // "... 300px below ..."
}
```

**handleObservations 逻辑：**

1. **累积等待时间警告**：等待超过 3 秒时提醒
2. **URL 变化检测**：导航后记录 `Page navigated to → {url}`
3. **剩余步数警告**：
   - 剩 5 步：提醒考虑收尾
   - 剩 2 步：强制完成

### 4.2 THINK 阶段

**Prompt 组装顺序：**

```
<instructions>                    // 可选：系统指令、页面指令、llms.txt
<agent_state>
  <user_request>                 // 用户任务
  <step_info>                     // "Step X of Y"
</agent_state>

<agent_history>                  // 历史事件流
  <step_1>
    Evaluation: ...
    Memory: ...
    Next Goal: ...
    Action Results: ...
  </step_1>
  <sys>Observation...</sys>       // 系统消息
</agent_history>

<browser_state>                  // 页面状态
  header + content + footer
</browser_state>
```

**Reflection-Before-Action 思维模型：**

每个工具调用前，LLM 必须输出：

```json
{
  "evaluation_previous_goal": "上一 action 做得如何？成功/失败/不确定",
  "memory": "记忆：关键信息、进度、计数等",
  "next_goal": "下一步目标",
  "action": {
    "tool_name": { /* 工具参数 */ }
  }
}
```

**关键文件：**
- Prompt 模板：`packages/core/src/prompts/system_prompt.md`
- Prompt 组装：`PageAgentCore.#assembleUserPrompt()`
- 工具打包：`PageAgentCore.#packMacroTool()`

### 4.3 ACT 阶段

**内置工具列表：**

| 工具名 | 参数 | 功能 |
|--------|------|------|
| `done` | `{ text: string, success: boolean }` | 完成任务 |
| `wait` | `{ seconds: number }` | 等待（最多 10 秒） |
| `ask_user` | `{ question: string }` | 向用户提问 |
| `click_element_by_index` | `{ index: number }` | 点击元素 |
| `input_text` | `{ index: number, text: string }` | 输入文本 |
| `select_dropdown_option` | `{ index: number, text: string }` | 选择下拉选项 |
| `scroll` | `{ down, numPages, pixels, index }` | 垂直滚动 |
| `scroll_horizontally` | `{ right, pixels, index }` | 水平滚动 |
| `execute_javascript` | `{ script: string }` | 执行 JS（实验性） |

**关键文件：**
- 工具定义：`packages/core/src/tools/index.ts`

### 4.4 MacroTool 执行

所有工具被合并为一个 `MacroTool`，LLM 每次必须调用：

```typescript
{
  "evaluation_previous_goal": "...",
  "memory": "...",
  "next_goal": "...",
  "action": {
    "click_element_by_index": { "index": 42 }
  }
}
```

**MacroTool 结构（Zod schema）：**

```typescript
const macroToolSchema = z.object({
  evaluation_previous_goal: z.string().optional(),
  memory: z.string().optional(),
  next_goal: z.string().optional(),
  action: z.union([
    z.object({ done: z.object({...}) }),
    z.object({ wait: z.object({...}) }),
    z.object({ click_element_by_index: z.object({...}) }),
    // ... 其他工具
  ])
})
```

## 5. 事件系统

```
┌──────────────────────────────────────────────────────┐
│  EventTarget (PageAgentCore extends EventTarget)     │
├──────────────────────────────────────────────────────┤
│  'statuschange'   → AgentStatus 变化                 │
│                    (idle/running/completed/error)    │
│  'historychange'  → History 事件更新（持久化）         │
│  'activity'       → 实时活动（transient）             │
│  'dispose'       → 清理触发                          │
└──────────────────────────────────────────────────────┘
```

### 5.1 事件类型详解

**AgentStatus：**
```typescript
type AgentStatus = 'idle' | 'running' | 'completed' | 'error'
```

**HistoricalEvent（持久化）：**
```typescript
type HistoricalEvent =
  | AgentStepEvent      // type: 'step' - 包含 reflection + action
  | ObservationEvent   // type: 'observation' - 观察消息
  | UserTakeoverEvent  // type: 'user_takeover' - 用户接管
  | RetryEvent         // type: 'retry' - LLM 重试
  | AgentErrorEvent    // type: 'error' - 错误
```

**AgentActivity（临时 UI 反馈）：**
```typescript
type AgentActivity =
  | { type: 'thinking' }
  | { type: 'executing'; tool: string; input: unknown }
  | { type: 'executed'; tool: string; input: unknown; output: string; duration: number }
  | { type: 'retrying'; attempt: number; maxAttempts: number }
  | { type: 'error'; message: string }
```

## 6. PageController 详解

### 6.1 DOM Pipeline

```
Live DOM
    │
    ▼
┌─────────────────────────────────────┐
│  getFlatTree()                      │
│  (packages/page-controller/src/     │
│   dom/dom_tree/index.js)            │
│                                      │
│  遍历 DOM，构建 FlatDomTree：         │
│  - 过滤：aria-hidden,               │
│    data-browser-use-ignore          │
│  - 检测：isElementVisible           │
│  - 检测：isInteractiveElement       │
│  - 检测：isScrollableElement        │
│  - 滚动容器检测                     │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  getSelectorMap()                   │
│  index → InteractiveElementDomNode  │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  flatTreeToString()                 │
│  生成简化文本：                      │
│  格式：[index]<tag>text</tag>       │
└─────────────────────────────────────┘
    │
    ▼
Simplified HTML for LLM
```

### 6.2 交互元素检测逻辑

**检测优先级：**

1. **光标检测（主要方法）**
   ```javascript
   const interactiveCursors = [
     'pointer', 'move', 'text', 'grab', 'grabbing',
     'cell', 'copy', 'alias', 'all-scroll', 'col-resize',
     // ... 更多
   ]
   if (style.cursor in interactiveCursors) return true
   ```

2. **标签名检测**
   ```javascript
   const interactiveElements = [
     'a', 'button', 'input', 'select', 'textarea',
     'details', 'summary', 'label', 'option', 'optgroup'
   ]
   ```

3. **ARIA role 检测**
   ```javascript
   const interactiveRoles = [
     'button', 'menu', 'menubar', 'menuitem', 'menuitemradio',
     'menuitemcheckbox', 'radio', 'checkbox', 'tab', 'switch',
     'slider', 'spinbutton', 'combobox', 'searchbox', 'textbox'
   ]
   ```

4. **事件监听器检测（备选）**
   ```javascript
   getEventListeners(element)  // Chrome DevTools 函数
   ```

5. **滚动容器检测**
   ```javascript
   isScrollableElement(element)
   // 条件：overflow: auto/scroll + scrollWidth > clientWidth + threshold > 4
   ```

### 6.3 元素操作（actions.ts）

**点击模拟（W3C Pointer Events 顺序）：**
```
pointerover → pointerenter → mouseover → mouseenter
    ↓
pointerdown → mousedown → [focus]
    ↓
pointerup → mouseup → click
```

**输入模拟：**
- 普通 input/textarea：直接设置 `.value`，dispatch `input` 事件
- contenteditable：dispatch `beforeinput` + `input` 事件
- Fallback：`execCommand('insertText')`（已废弃但广泛支持）

**关键文件：**
- `packages/page-controller/src/actions.ts`

### 6.4 SimulatorMask

可选的视觉遮罩层，在自动化执行时阻止用户操作。

```typescript
// 启用
const controller = new PageController({ enableMask: true })

// 显示遮罩
await controller.showMask()

// 隐藏遮罩
await controller.hideMask()
```

**关键文件：**
- `packages/page-controller/src/mask/SimulatorMask.ts`

### 6.5 PageController API

```typescript
class PageController extends EventTarget {
  // 状态查询
  async getCurrentUrl(): Promise<string>
  async getLastUpdateTime(): Promise<number>
  async getBrowserState(): Promise<BrowserState>

  // DOM 操作
  async updateTree(): Promise<string>
  async cleanUpHighlights(): Promise<void>

  // 元素操作
  async clickElement(index: number): Promise<ActionResult>
  async inputText(index: number, text: string): Promise<ActionResult>
  async selectOption(index: number, optionText: string): Promise<ActionResult>
  async scroll(options: { down: boolean; numPages: number; pixels?: number; index?: number }): Promise<ActionResult>
  async scrollHorizontally(options: { right: boolean; pixels: number; index?: number }): Promise<ActionResult>
  async executeJavascript(script: string): Promise<ActionResult>

  // 遮罩操作
  async showMask(): Promise<void>
  async hideMask(): Promise<void>

  // 生命周期
  dispose(): void
}
```

## 7. LLM 客户端详解

### 7.1 类结构

```
LLM (packages/llms/src/index.ts)
    │
    └── OpenAIClient (packages/llms/src/OpenAIClient.ts)
```

**LLM 类职责：**
- 配置解析和验证
- 重试机制包装
- 事件派发（retry、error）

**OpenAIClient 类职责：**
- 构建 OpenAI 格式请求
- 发送 API 请求
- 解析和验证响应
- 工具执行

### 7.2 重试机制

```typescript
async function withRetry(fn, { maxRetries, onRetry, onError }) {
  let attempt = 0
  while (attempt <= maxRetries) {
    try {
      return await fn()
    } catch (error) {
      // AbortError 不重试
      if (error.name === 'AbortError') throw error

      // InvokeError 有 retryable 标志
      if (error instanceof InvokeError && !error.retryable) throw error

      onRetry(attempt++)
      await wait(100)  // 固定延迟后重试
    }
  }
  throw lastError
}
```

### 7.3 InvokeError 类型

```typescript
enum InvokeErrorType {
  NETWORK_ERROR,        // 网络错误
  AUTH_ERROR,          // 401/403
  RATE_LIMIT,          // 429
  SERVER_ERROR,        // 5xx
  CONTEXT_LENGTH,      // max tokens
  CONTENT_FILTER,      // 安全过滤
  NO_TOOL_CALL,        // 无工具调用
  INVALID_TOOL_ARGS,   // 参数解析/验证失败
  TOOL_EXECUTION_ERROR // 工具执行失败
}
```

### 7.4 响应标准化

```typescript
normalizeResponse: (res) => normalizeResponse(res, this.tools)
```

用于修复模型输出的各种格式错误：
- JSON 语法错误
- 字段类型错误
- 缺少必需字段

### 7.5 工具调用流程

```
1. 构建请求体
   - messages: [{ role: 'system', content: ... }, { role: 'user', content: ... }]
   - tools: OpenAI 格式工具定义
   - tool_choice: 'required' 或指定工具名

2. 发送请求
   POST {baseURL}/chat/completions
   Headers: { 'Content-Type': 'application/json', Authorization: 'Bearer ...' }

3. 解析响应
   - 检查 HTTP 状态码
   - 检查 finish_reason
   - 提取 tool_calls[0].function

4. 验证参数
   - JSON.parse(arguments)
   - Zod schema safeParse

5. 执行工具
   - tool.execute(input)
   - 返回 { input, output }
```

**关键文件：**
- `packages/llms/src/index.ts`
- `packages/llms/src/OpenAIClient.ts`
- `packages/llms/src/types.ts`
- `packages/llms/src/errors.ts`

## 8. 配置项详解

```typescript
interface AgentConfig {
  // LLM 配置（必需）
  baseURL: string           // LLM API 地址
  model: string             // 模型名称
  apiKey?: string

  // 行为配置
  language?: 'en-US' | 'zh-CN'  // 默认 'en-US'
  maxSteps?: number         // 最大步数，默认 40

  // 扩展工具
  customTools?: Record<string, PageAgentTool | null>
  // 示例：移除 ask_user 工具
  // customTools: { ask_user: null }

  // 指令
  instructions?: {
    system?: string                    // 全局系统指令
    getPageInstructions?: (url: string) => string | null  // 动态页面指令
  }

  // 生命周期钩子
  onBeforeStep?: (agent: PageAgentCore, stepCount: number) => Promise<void>
  onAfterStep?: (agent: PageAgentCore, history: HistoricalEvent[]) => Promise<void>
  onBeforeTask?: (agent: PageAgentCore) => Promise<void>
  onAfterTask?: (agent: PageAgentCore, result: ExecutionResult) => Promise<void>
  onDispose?: (agent: PageAgentCore) => void

  // 实验性功能
  experimentalScriptExecutionTool?: boolean  // 启用 execute_javascript
  experimentalLlmsTxt?: boolean            // 启用 /llms.txt 获取
  transformPageContent?: (content: string) => string  // 内容转换/脱敏
  customSystemPrompt?: string              // 完全自定义 system prompt
  stepDelay?: number                      // 步间延迟，默认 0.4s
}
```

## 9. DOM 元素索引规则

### 9.1 索引格式

```
[33]<div>User form</div>
    *[35]<button>Submit</button>

说明：
- [33] 是元素的数字索引
- * 表示该元素是上一步之后新出现的
- \t (tab缩进) 表示 HTML 层级关系（子元素）
- 只有带 [index] 的元素才可以交互
```

### 9.2 高亮系统

- 每个交互元素被分配一个数字索引（0, 1, 2, ...）
- 索引显示在元素右上角的彩色标签中
- 标签颜色根据索引循环（12 色）
- 可通过 `data-page-agent-not-interactive` 属性排除元素

### 9.3 可滚动容器

可滚动元素会被标记：
```
[42]<div data-scrollable>Container</div>
```

滚动信息包含各方向的滚动距离。

## 10. System Prompt 要点

来自 `packages/core/src/prompts/system_prompt.md`：

### 10.1 语言规则
- 默认工作语言：**English**
- 使用用户的语言返回

### 10.2 交互规则
- 只交互有索引的元素
- 只使用提供的索引
- 默认只列出可见视口内的元素
- 3 次重复 action 后要换策略
- captcha 无法处理，直接告知用户

### 10.3 任务类型处理
1. **明确步骤的任务**：严格按步骤执行，不跳过
2. **开放任务**：自己规划，保持创造性

### 10.4 完成条件
必须调用 `done` 的情况：
- 完成任务
- 达到最大步数
- 卡住或无法继续
- 请求不明确或不适合

### 10.5 推理规则
- 从 history 跟踪进度
- 明确判断成功/失败/不确定
- 从 history 和 browser_state 分析状态
- 卡住时考虑换方案或问用户

## 11. 文件索引

| 功能 | 文件路径 |
|------|---------|
| 核心 Agent | `packages/core/src/PageAgentCore.ts` |
| 工具定义 | `packages/core/src/tools/index.ts` |
| 类型定义 | `packages/core/src/types.ts` |
| 工具函数 | `packages/core/src/utils/index.ts` |
| 系统 Prompt | `packages/core/src/prompts/system_prompt.md` |
| LLM 客户端 | `packages/llms/src/index.ts` |
| OpenAI Client | `packages/llms/src/OpenAIClient.ts` |
| LLM 类型 | `packages/llms/src/types.ts` |
| PageController | `packages/page-controller/src/PageController.ts` |
| DOM 操作 | `packages/page-controller/src/actions.ts` |
| DOM 提取 | `packages/page-controller/src/dom/dom_tree/index.js` |
| SimulatorMask | `packages/page-controller/src/mask/SimulatorMask.ts` |

## 12. 常见问题

### Q: 如何添加新工具？
1. 在 `packages/core/src/tools/index.ts` 中使用 `tools.set()` 注册
2. 如果需要 DOM 操作，先在 `PageController` 中添加方法
3. 工具通过 `this.pageController` 访问 DOM

### Q: 如何自定义工具？
```typescript
const customTools = {
  // 覆盖已有工具
  click_element_by_index: tool({
    description: 'Custom click',
    inputSchema: z.object({ index: z.int() }),
    execute: async function (this: PageAgentCore, input) {
      // 自定义逻辑
      return 'clicked'
    }
  }),
  // 移除工具
  ask_user: null
}
```

### Q: 如何处理页面特定指令？
```typescript
const agent = new PageAgentCore({
  instructions: {
    getPageInstructions: (url) => {
      if (url.includes('github.com')) {
        return 'On GitHub, use the search bar to find repositories'
      }
      return null
    }
  }
})
```

### Q: 如何启用 llms.txt？
```typescript
const agent = new PageAgentCore({
  experimentalLlmsTxt: true  // 自动获取 /llms.txt 并注入 prompt
})
```

---

*文档版本：2026-04-20*
