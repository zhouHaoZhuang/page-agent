# LLMs 包工作流程详解

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        LLM (index.ts)                        │
│  - 对外入口，管理配置和重试                                   │
│  - 继承 EventTarget，发送 retry/error 事件                    │
└─────────────────────────┬───────────────────────────────────┘
                          │ 组合
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   OpenAIClient (OpenAIClient.ts)            │
│  - 具体实现，调用 OpenAI 兼容 API                            │
│  - 处理请求/响应、错误分类                                   │
└─────────────────────────────────────────────────────────────┘
```

## 调用流程图

```
PageAgentCore                          LLM                              OpenAIClient
     │                                   │                                  │
     │  invoke(messages, tools, signal)   │                                  │
     │ ─────────────────────────────────▶                                  │
     │                                   │                                  │
     │                          withRetry(                                │
     │                            fn: () => {                            │
     │                              client.invoke(...)  ←─────────────────│
     │                            }                                      │
     │                          )                                        │
     │                                   │                               │
     │                                   │   1. 转换工具格式               │
     │                                   │   2. 发送 HTTP 请求             │
     │                                   │   3. 解析响应                   │
     │                                   │   4. 验证工具调用               │
     │                                   │   5. 执行工具                   │
     │                                   │   6. 返回 InvokeResult          │
     │                                   │ ─────────────────────────────────▶│
     │                                   │                               │
     │                                   │ ◀────────────────────────────────
     │                                   │   InvokeResult                 │
     │                          返回结果  │                               │
     │ ◀─────────────────────────────────                                │
     │                                   │                               │
```

## 详细步骤解析

### 1. LLM.invoke() — 对外入口

**文件**: `packages/llms/src/index.ts`

```typescript
async invoke(
    messages: Message[],           // 对话历史
    tools: Record<string, Tool>,  // 可用工具
    abortSignal: AbortSignal,     // 中断信号
    options?: InvokeOptions        // 可选配置
): Promise<InvokeResult>
```

**核心逻辑**：包裹 `withRetry`，实现自动重试

### 2. withRetry() — 重试机制

```typescript
async withRetry(fn, settings) {
    let attempt = 0
    while (attempt <= settings.maxRetries) {
        try {
            return await fn()  // 成功则返回
        } catch (error) {
            // AbortError → 不重试，直接抛出
            if ((error as any)?.rawError?.name === 'AbortError') throw error

            // 非可重试错误 → 直接抛出
            if (error instanceof InvokeError && !error.retryable) throw error

            // 可重试错误 → 等待 100ms，重试
            attempt++
        }
    }
}
```

### 3. OpenAIClient.invoke() — 核心实现

**文件**: `packages/llms/src/OpenAIClient.ts`

#### Step 1: 准备请求

```typescript
// 1. 转换工具格式 (Zod → OpenAI tool format)
const openaiTools = Object.entries(tools).map(([name, t]) => zodToOpenAITool(name, t))

// 2. 构建请求体
const requestBody = {
    model: this.config.model,
    temperature: this.config.temperature,
    messages,
    tools: openaiTools,
    parallel_tool_calls: false,      // 禁用并行工具调用
    tool_choice: toolChoice,         // 强制选择指定工具
}
```

#### Step 2: 发送请求

```typescript
response = await this.fetch(`${this.config.baseURL}/chat/completions`, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${this.config.apiKey}`,
    },
    body: JSON.stringify(requestBody),
    signal: abortSignal,  // 支持中断
})
```

#### Step 3: 处理 HTTP 错误

| 状态码 | 错误类型 | 可重试 |
|---|---|---|
| 401/403 | `AUTH_ERROR` | ❌ |
| 429 | `RATE_LIMIT` | ✅ |
| 5xx | `SERVER_ERROR` | ✅ |
| 其他 | `UNKNOWN` | ✅ |

#### Step 4: 解析响应

```typescript
// 检查 finish_reason
switch (choice.finish_reason) {
    case 'tool_calls':
    case 'function_call':  // gemini 兼容
    case 'stop':
        // 正常，继续处理
        break
    case 'length':
        throw InvokeError(CONTEXT_LENGTH, ...)  // Token 超限
    case 'content_filter':
        throw InvokeError(CONTENT_FILTER, ...)  // 内容被过滤
}
```

#### Step 5: 提取并验证工具调用

```typescript
// 1. 提取工具名
const toolCallName = normalizedChoice.message.tool_calls[0].function.name

// 2. 检查工具是否存在
const tool = tools[toolCallName]
if (!tool) throw InvokeError(UNKNOWN, `Tool not found`)

// 3. 解析参数
const parsedArgs = JSON.parse(argString)

// 4. Zod 验证
const validation = tool.inputSchema.safeParse(parsedArgs)
if (!validation.success) throw InvokeError(INVALID_TOOL_ARGS, ...)
```

#### Step 6: 执行工具并返回

```typescript
// 执行工具（由 PageAgentCore 定义）
const toolResult = await tool.execute(toolInput)

// 返回结构化结果
return {
    toolCall: { name: toolCallName, args: toolInput },
    toolResult,        // 工具执行结果
    usage: {           // Token 用量
        promptTokens,
        completionTokens,
        totalTokens,
    },
    rawResponse: data,   // 原始 API 响应
    rawRequest: requestBody,
}
```

## 错误类型定义

**文件**: `packages/llms/src/errors.ts`

```typescript
export const InvokeErrorType = {
    // 可重试
    NETWORK_ERROR: 'network_error',       // 网络错误
    RATE_LIMIT: 'rate_limit',             // 速率限制
    SERVER_ERROR: 'server_error',         // 5xx 错误
    NO_TOOL_CALL: 'no_tool_call',        // 模型未调用工具
    INVALID_TOOL_ARGS: 'invalid_tool_args', // 工具参数无效
    TOOL_EXECUTION_ERROR: 'tool_execution_error', // 工具执行错误
    UNKNOWN: 'unknown',                   // 未知错误

    // 不可重试
    AUTH_ERROR: 'auth_error',             // 认证失败
    CONTEXT_LENGTH: 'context_length',     // Prompt 太长
    CONTENT_FILTER: 'content_filter',     // 内容被过滤
} as const
```

## 事件系统

LLM 类继承 `EventTarget`，支持以下事件：

```typescript
llm.addEventListener('retry', (e) => {
    const { attempt, maxAttempts } = e.detail
    console.log(`Retrying... ${attempt}/${maxAttempts}`)
})

llm.addEventListener('error', (e) => {
    const { error } = e.detail
    console.error('LLM Error:', error)
})
```

## 配置项

```typescript
interface LLMConfig {
    baseURL: string           // API 地址 (必需)
    model: string            // 模型名称 (必需)
    apiKey?: string          // API Key
    temperature?: number     // 温度参数 (默认 0.7)
    maxRetries?: number      // 最大重试次数 (默认 5)
    disableNamedToolChoice?: boolean  // 禁用强制工具选择
    customFetch?: typeof fetch  // 自定义 fetch 函数
}
```

## 总结

| 组件 | 职责 |
|---|---|
| **LLM** | 配置解析 + 重试包装 + 事件分发 |
| **OpenAIClient** | HTTP 调用 + 响应解析 + 工具验证 + 工具执行 |
| **withRetry** | 错误分类 + 决定是否重试 |

### 关键设计

1. **一次 LLM 调用 + 一次工具执行** = `invoke()` 的原子操作
2. 工具执行由调用方（PageAgentCore）定义，LLM 层只负责调用
3. 支持 `AbortSignal` 中断，可取消正在进行的请求
4. 错误分类明确，支持按错误类型决定是否重试
