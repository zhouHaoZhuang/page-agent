# Page-Agent 工作流程详细解析

本文件补充说明 Page-Agent 的每个高层流程步骤的详细过程，帮助开发者深入理解各模块的协作与数据流。

---

## 详细流程解析（逐步拆解）

### 1. 启动演示
- 在根目录执行 `npm run dev:demo`，会调用根 `package.json` 的脚本，实际触发 `packages/page-agent` 的 demo 构建与本地静态服务启动。
- 相关脚本：
  - 根目录 `package.json`：
    ```json
    "scripts": {
      "dev:demo": "pnpm --filter @page-agent/page-agent... run dev:demo"
    }
    ```
  - `packages/page-agent/package.json`：
    ```json
    "scripts": {
      "dev:demo": "vite build --config vite.iife.config.js && serve dist/demo"
    }
    ```
- 产物：`dist/demo` 目录下的 HTML 和 IIFE 构建产物。
- 访问本地端口（如 5174），加载 demo 页面。

### 2. PageAgent 初始化
- demo 页面入口脚本（如 `src/demo.ts`）会执行：
  ```ts
  import { PageAgent } from './PageAgent';
  const agent = new PageAgent(config);
  agent.mountUI();
  ```
- `PageAgent` 构造函数内部：
  - 实例化 `PageAgentCore`（`packages/core/src/PageAgentCore.ts`），负责任务流转。
  - 实例化 `PageController`（`packages/page-controller/src/PageController.ts`），负责页面 DOM 操作。
  - 实例化 LLM 客户端（如 `OpenAIClient`，`packages/llms/src/OpenAIClient.ts`），用于自然语言解析。
  - 实例化 UI 组件（`packages/ui/src/index.ts`），负责交互面板。
- 依赖注入：各模块通过构造参数或 set 方法互相持有引用，便于回调和事件流转。

### 3. 用户发起任务（自然语言指令）
- 用户在 UI 面板输入指令（如“帮我填写表单”），点击按钮后触发事件：
  - UI 组件通过事件（如 `onSubmit`）将指令文本传递给 `PageAgentCore`。
  - 代码示例：
    ```ts
    ui.on('submit', (input) => core.handleUserInput(input));
    ```

### 4. 规划与解析（Core + LLMS）
- `PageAgentCore` 接收到指令后，调用 LLM 客户端（如 `OpenAIClient`）：
  - 发送 prompt，请求将自然语言转为结构化动作序列（如 JSON）。
  - 代码示例：
    ```ts
    const actions = await llms.planActions(userInput);
    // actions: [{ type: 'fill', selector: 'input[name=realname]', value: '张三' }, ...]
    ```
- LLM 可能返回建议的选择器、操作类型、参数等。
- `PageAgentCore` 负责校验、补全、优化动作序列。

### 5. 执行动作（PageController）
- `PageController` 依次执行动作序列：
  - 查找元素：
    ```ts
    const el = document.querySelector(action.selector);
    ```
  - 填写输入框：
    ```ts
    el.value = action.value;
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
    ```
  - 点击按钮：
    ```ts
    el.click();
    ```
  - 支持异步等待、重试、异常捕获。
- 每一步执行结果通过回调或事件上报 `PageAgentCore` 和 UI。

### 6. 展示与回馈（UI）
- UI 组件监听任务进度和状态变更：
  - 展示当前步骤、日志、错误提示。
  - 高亮当前操作元素（如加边框、遮罩）。
  - 用户可手动介入（如修改 selector、跳过、重试），通过事件回传 `PageAgentCore`。
- 典型代码：
  ```ts
  core.on('step', (info) => ui.updateStep(info));
  core.on('error', (err) => ui.showError(err));
  ```

### 7. 结束与持久化
- 所有动作执行完毕后：
  - `PageAgentCore` 汇总执行历史、结果。
  - 可调用导出/保存方法，将操作序列保存为 JSON 或脚本。
  - 支持本地存储、下载、或上报后端。
- 发布相关脚本（如 `prepublishOnly`）会自动构建、校验产物，保证包的可用性。

---

如需进一步细化（如时序图、伪代码、关键数据结构），可继续补充。
