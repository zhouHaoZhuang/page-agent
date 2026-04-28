# PageAgent 关键模块协作与详细工作过程

本文件详细解析 PageAgent 体系中 pageagent、PageAgentCore、PageController、llms、ui 五大核心模块的职责、内部流程，以及它们之间的协作关系，帮助开发者深入理解整体架构和数据流转。

---

## 1. PageAgent（总控协调器）
- 作用：作为顶层协调者，负责各子模块的初始化、依赖注入、生命周期管理和事件分发。
- 主要职责：
  - 初始化 core、controller、llms、ui 等子模块。
  - 负责模块间的依赖注入和引用传递。
  - 监听 UI、core、controller 等模块的事件，进行统一调度和状态同步。
  - 对外暴露统一 API（如 mountUI、runTask、exportHistory 等）。
- 典型流程：
  1. 构造时依次实例化各子模块，将必要依赖传递给下游。
  2. 监听 UI 的用户输入事件，转发给 core。
  3. 监听 core 的任务进度、异常、结果事件，转发给 UI。
  4. 监听 controller 的执行反馈，协调 UI 展示和 core 状态。

## 2. PageAgentCore（任务规划与调度）
- 作用：负责自然语言指令的解析、任务规划、动作序列生成、异常处理和任务状态管理。
- 主要职责：
  - 接收用户输入（自然语言），调用 llms 进行语义解析和动作规划。
  - 维护任务执行状态机，调度 controller 执行动作序列。
  - 处理异常、重试、回滚等高级控制逻辑。
  - 向 UI 和 PageAgent 汇报任务进度、结果、异常。
- 典型流程：
  1. 接收 PageAgent/ UI 的用户指令。
  2. 调用 llms 进行动作规划，获得结构化动作列表。
  3. 依次调度 PageController 执行动作，监听执行反馈。
  4. 根据执行结果动态调整后续计划（如重试、修正 selector）。
  5. 汇总执行历史，支持导出/回放。

## 3. PageController（页面动作执行器）
- 作用：负责在真实页面环境中执行具体的 DOM 操作（如查找元素、输入、点击、等待等），并将执行结果反馈给 core。
- 主要职责：
  - 接收 core 下发的动作序列，依次在页面执行。
  - 封装常见 DOM 操作（querySelector、输入、点击、滚动、等待等）。
  - 支持智能 selector 匹配、异常捕获、异步等待、回放等能力。
  - 将每一步执行结果（成功/失败/异常）通过事件或回调上报 core。
- 典型流程：
  1. 接收动作列表（如 [{type: 'fill', selector: 'input[name=realname]', value: '张三'}]）。
  2. 依次查找元素、执行操作、触发事件。
  3. 捕获异常或页面变化，及时反馈。
  4. 支持回放、撤销等高级功能。

## 4. llms（大模型接口/动作规划器）
- 作用：负责将自然语言指令转化为结构化的动作序列，或对动作计划进行补全、优化、校验。
- 主要职责：
  - 接收 core 的 prompt，调用远端 LLM（如 OpenAI）进行语义理解和动作规划。
  - 输出结构化动作描述（如 JSON 数组，每项包含 type、selector、value 等）。
  - 支持 selector 智能建议、意图补全、异常诊断等。
- 典型流程：
  1. 接收自然语言指令（如“帮我填写表单”）。
  2. 构造 prompt，调用 LLM API。
  3. 解析 LLM 返回的动作序列，反馈给 core。
  4. 支持多轮补全、校验。

## 5. ui（用户界面与交互层）
- 作用：负责与用户交互，展示任务进度、日志、异常、可视化选中元素，并收集用户输入。
- 主要职责：
  - 展示输入框、按钮、任务进度、日志、异常提示等。
  - 高亮当前操作元素，支持可视化 selector 选择。
  - 监听用户操作（如输入、跳过、重试、修改 selector），通过事件通知 PageAgent/core。
  - 实时同步任务状态、步骤进度。
- 典型流程：
  1. 用户输入自然语言指令，点击“执行”按钮。
  2. 展示任务进度、每步详情、异常提示。
  3. 支持用户手动介入（如修改 selector、跳过步骤）。
  4. 展示最终结果、导出操作历史。

---

## 6. 模块间协作流程图（简述）

1. 用户在 UI 输入指令 → UI 触发事件 → PageAgent 收到后转发给 PageAgentCore
2. PageAgentCore 调用 llms 解析指令，获得动作序列
3. PageAgentCore 调用 PageController 执行动作
4. PageController 执行每一步，结果回传 PageAgentCore
5. PageAgentCore 更新状态，通知 UI 展示进度/异常
6. 用户可在 UI 介入，操作同步到 PageAgentCore，流程继续
7. 所有动作完成后，PageAgentCore 汇总结果，UI 展示，PageAgent 可导出/回放

---

## 7. 关键数据流示例

- 用户输入：
  ```json
  {
    "input": "请帮我填写表单并提交"
  }
  ```
- llms 输出动作序列：
  ```json
  [
    {"type": "fill", "selector": "input[name=realname]", "value": "张三"},
    {"type": "fill", "selector": "input[name=email]", "value": "zhang@example.com"},
    {"type": "click", "selector": "button[type=submit]"}
  ]
  ```
- PageController 执行反馈：
  ```json
  [
    {"step": 1, "status": "success"},
    {"step": 2, "status": "success"},
    {"step": 3, "status": "success"}
  ]
  ```

---

如需进一步细化具体实现、时序图或伪代码，可随时补充。
