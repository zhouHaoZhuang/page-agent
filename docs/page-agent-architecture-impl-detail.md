# PageAgent 关键模块协作：实现细节、时序图与伪代码

本文件在架构说明基础上，进一步细化 PageAgent、PageAgentCore、PageController、llms、ui 的具体实现要点、典型时序图与伪代码，便于开发者理解和扩展。

---

## 1. 典型时序图（文字版）

```
用户         UI         PageAgent      PageAgentCore      llms         PageController
 |    输入指令/点击   |      |                |                |
 |------------------->|      |                |                |
 |                    |  onUserInput         |                |
 |                    |--------------------->|                |
 |                    |                      | planActions     |
 |                    |                      |--------------->|
 |                    |                      |   LLM API      |
 |                    |                      |<---------------|
 |                    |                      |  动作序列       |
 |                    |   onPlanReady        |                |
 |                    |<---------------------|                |
 |                    |  dispatchActions     |                |
 |                    |--------------------->|                |
 |                    |                      |  nextAction    |
 |                    |                      |--------------->|
 |                    |                      |                | 执行DOM操作
 |                    |                      |                | 反馈结果
 |                    |   onStepResult       |                |
 |<-------------------|<---------------------|<---------------|
 |                    |                      |                |
```

---

## 2. 关键实现要点

### PageAgent
- 组合 core、controller、llms、ui，负责依赖注入和事件桥接。
- 伪代码：
  ```ts
  class PageAgent {
    constructor(config) {
      this.core = new PageAgentCore({ llms, controller });
      this.ui = new UI({ onInput: this.handleInput });
      // ...依赖注入
    }
    handleInput(input) {
      this.core.handleUserInput(input);
    }
    // 监听 core/controller 事件，转发给 ui
  }
  ```

### PageAgentCore
- 负责任务状态、动作规划、调度 controller 执行。
- 伪代码：
  ```ts
  class PageAgentCore {
    async handleUserInput(input) {
      const actions = await this.llms.planActions(input);
      for (const action of actions) {
        const result = await this.controller.exec(action);
        this.ui.updateStep({ action, result });
        if (!result.success) break;
      }
    }
  }
  ```

### PageController
- 封装 DOM 操作，支持异步、异常处理。
- 伪代码：
  ```ts
  class PageController {
    async exec(action) {
      try {
        const el = document.querySelector(action.selector);
        if (!el) return { success: false, error: 'not found' };
        if (action.type === 'fill') {
          el.value = action.value;
          el.dispatchEvent(new Event('input', { bubbles: true }));
        } else if (action.type === 'click') {
          el.click();
        }
        return { success: true };
      } catch (e) {
        return { success: false, error: e };
      }
    }
  }
  ```

### llms
- 封装 LLM API 调用，prompt 构造与结果解析。
- 伪代码：
  ```ts
  class OpenAIClient {
    async planActions(input) {
      const prompt = this.buildPrompt(input);
      const resp = await fetchLLM(prompt);
      return this.parseActions(resp);
    }
  }
  ```

### ui
- 事件驱动，展示进度、收集输入、支持用户介入。
- 伪代码：
  ```ts
  class UI {
    constructor({ onInput }) {
      this.onInput = onInput;
      // 绑定按钮、输入框事件
    }
    updateStep({ action, result }) {
      // 展示当前步骤、状态
    }
  }
  ```

---

## 3. 典型数据结构

- 动作描述：
  ```ts
  type Action = {
    type: 'fill' | 'click';
    selector: string;
    value?: string;
  };
  ```
- 执行结果：
  ```ts
  type StepResult = {
    success: boolean;
    error?: string;
  };
  ```

---

## 4. 典型流程伪代码（端到端）

```ts
// 用户输入
ui.onInput = (input) => pageAgent.handleInput(input);

// PageAgent 转发
pageAgent.handleInput = (input) => core.handleUserInput(input);

// Core 调用 llms 规划
core.handleUserInput = async (input) => {
  const actions = await llms.planActions(input);
  for (const action of actions) {
    const result = await controller.exec(action);
    ui.updateStep({ action, result });
    if (!result.success) break;
  }
};
```

---

如需进一步细化具体模块实现、接口定义或补充时序图图片，可继续补充。
