# Page-Agent 工作流程（示例说明）

本文档结合项目根目录 `package.json` 和 `packages/page-agent/package.json`，通过一个具体的示例说明 Page-Agent 从启动到执行任务的典型工作流程，帮助理解各个 package 之间如何协同工作。

## 目标示例
场景：在网页上自动填写并提交一个表单（用户在演示页面点击「让机器人帮我填写表单」）。

## 先决条件（本地运行）
- 在仓库根目录，常用命令：
  - `npm run dev:demo`（根目录脚本，会在 workspace 中触发 `page-agent` 的 demo 监听与本地服务）
  - `npm start`（启动网站 demo：`@page-agent/website`）
- 打包构建：根目录 `npm run build`（会调用 `scripts/build.js` 构建所有包）；在单包中，`packages/page-agent` 有：
  - `npm run build`（内部使用 `vite build` 并生成 demo iife）
  - `npm run dev:demo`（在 `packages/page-agent` 中并发构建 iife 并启动本地静态服务器）

相关文件：
- 根 `package.json`（工作区与脚本定义）
- `packages/page-agent/package.json`（page-agent 包的构建/演示脚本）
- 代码入口：`packages/page-agent/src/PageAgent.ts`
- 关键子包：`packages/core`、`packages/page-controller`、`packages/llms`、`packages/ui`、`packages/extension`、`packages/mcp`

## 高层流程（步骤）
1. 启动演示
   - 在项目根执行 `npm run dev:demo`，它会触发 `page-agent` 包的 demo 构建并用 `serve` 提供静态页面（默认端口示例：5174）。
   - 打开演示页面（浏览器访问本地服务），页面会加载构建后的脚本（iife 或 esm），脚本中包含 `PageAgent` 的初始化代码。

2. PageAgent 初始化
   - 演示脚本（例如 `packages/page-agent/src/demo.ts`）创建 `new PageAgent(config)`。
   - `PageAgent` 在内部组装并初始化：
     - `core`：负责任务解析、计划与策略（`packages/core`）
     - `page-controller`：封装与页面 DOM 的交互（点击、输入、滚动、查询选择器等），并提供可回放的动作接口（`packages/page-controller`）
     - `llms`：负责与 LLM（如 OpenAI）通信，用于把自然语言指令转换为动作序列或补全步骤（`packages/llms`，例如 `OpenAIClient`）
     - `ui`：展示控制面板、日志、步骤状态与交互按钮（`packages/ui`）
   - 如果在浏览器扩展或多页场景，`extension` 与 `mcp` 会负责消息通道与多进程/多页面协同。

3. 用户发起任务（自然语言指令）
   - 用户在 UI 中输入指令（比如“帮我把表单里姓名、邮箱填上并提交”）。
   - UI 将指令发送给 `core`。

4. 规划与解析（Core + LLMS）
   - `core` 将自然语言指令转换为高层操作计划（可能调用 `llms` 将自然语言映射成更结构化的动作描述，例如：定位到姓名输入框 -> 输入张三 -> 定位邮箱 -> 输入 x@y.com -> 点击提交按钮）。
   - `llms`（`packages/llms`）可能会调用远端 LLM（`OpenAIClient`）来补全或校验每一步的意图与选择器建议。

5. 执行动作（PageController）
   - `page-controller` 接收由 `core` 输出的动作序列，依次在当前页面执行：
     - 查找元素（querySelector、基于属性或文本的智能匹配）
     - 设置输入框的值（dispatch input/change 事件）
     - 点击按钮、等待网络/DOM 变化
   - 每一步执行结果（成功/失败/异常）会回传给 `core` 与 `ui`，用于展示与可能的重试/修正策略。

6. 展示与回馈（UI）
   - `ui` 在侧边栏或浮层展示步骤进度、日志与可能的可视化选中元素（高亮）。
   - 用户可在 UI 中手工介入（跳过某步、修改输入、确认选择器），这些操作会更新 `core` 的执行计划并触发重新执行。

7. 结束与持久化
   - 动作序列执行完毕后，`core` 可以将执行结果、操作历史或生成的脚本保存到本地（或上报到后端），用于回放或分享。
   - 在发布包时，`packages/page-agent/package.json` 中的 `prepublishOnly` / `postpublish` 脚本会配合根脚本运行自定义的发布流程（参见 `scripts/pre-publish.js` 等）。

## 一个更具体的交互时间线（以填写表单为例）
1. 浏览器打开 demo 页面 -> demo 脚本实例化 `PageAgent` 并注入 `ui` 按钮。
2. 用户点击「自动填写」按钮，UI 发送指令 "填入姓名：张三，邮箱：zhang@example.com，提交表单"。
3. `core` 请求 `llms` 将自然语言转为动作列表（例如 JSON：[{type: 'fill', selector: 'input[name=realname]', value:'张三'}, ...]）。
4. `page-controller` 遍历动作：
   - 找到 `input[name=realname]`，设置 value 并触发事件。
   - 找到 `input[name=email]`，设置 value。
   - 找到 `button[type=submit]`，click。
5. 每一步执行后，页面可能出现验证错误或网络提交状态；`page-controller` 报告状态回 `core`；`ui` 更新显示。
6. 若提交成功，`ui` 显示成功提示，并可导出操作记录用于回放。

## 与发布/构建流程的关系
- 本地开发：使用 `npm run dev:demo`（或进入单包 `packages/page-agent` 执行 `npm run dev:demo`）方便调试 demo、UI 与页面动作。
- 单包构建（发布前）：`packages/page-agent` 的 `build` 会用 `vite build` 生成 `dist/`（包括 ESM 与 IIFE demo），`publishConfig` 在 `package.json` 中指向 `dist/esm` 中的产物。
- 仓库级构建：根目录 `npm run build` 会调用项目脚本来顺序或并行构建所有子包，保证依赖版本统一并生成最终可发布的包。

## 参考文件（位置）
- 根脚本：`/package.json`
- PageAgent 包配置：`/packages/page-agent/package.json`
- 入口实现：`/packages/page-agent/src/PageAgent.ts`
- Core：`/packages/core/src/PageAgentCore.ts`
- PageController：`/packages/page-controller/src/PageController.ts`
- LLMS 客户端：`/packages/llms/src/OpenAIClient.ts`
- UI：`/packages/ui/src/index.ts`

---

如果需要，我可以：
- 把示例的动作序列（JSON）和演示页面的调用示例一并生成到 demo 目录；
- 或者把本文档再细化成流程图并放到 `docs/` 下。
