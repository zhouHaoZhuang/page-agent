# Page Controller 工作流程详解

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      PageController                              │
│  - 管理 DOM 状态和元素交互                                       │
│  - 独立的异步 API，支持远程调用                                   │
│  - 可选的可视遮罩层                                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   DOM 模块    │   │   Actions     │   │  SimulatorMask │
│   - DOM 提取   │   │  - click      │   │  - 视觉遮罩   │
│   - 简化 HTML │   │  - inputText  │   │               │
│   - 高亮管理  │   │  - scroll     │   │               │
└───────────────┘   └───────────────┘   └───────────────┘
```

## 核心职责

| 模块 | 职责 |
|------|------|
| **PageController** | 对外 API，协调 DOM 和 Actions |
| **DOM 模块** | 提取页面元素，构建可交互的 DOM 树 |
| **Actions** | 模拟用户交互（点击、输入、滚动） |
| **SimulatorMask** | 视觉遮罩，阻止用户操作 |

---

## 1. DOM 提取流程 (updateTree)

### 流程图

```
PageController.updateTree()
       │
       ▼
┌─────────────────────────────────────────┐
│  1. 触发 beforeUpdate 事件              │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  2. 暂时禁用遮罩 (pointerEvents: none)  │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  3. 清理旧的高亮元素                     │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  4. 构建黑名单                           │
│     - 配置中的 blacklist                 │
│     - data-page-agent-not-interactive   │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  5. getFlatTree() → 提取 DOM 树         │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  6. flatTreeToString() → 简化 HTML      │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  7. 构建 selectorMap (索引→元素)        │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  8. 构建 elementTextMap (索引→文本)     │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  9. 恢复遮罩 (pointerEvents: auto)      │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  10. 触发 afterUpdate 事件              │
└─────────────────────────────────────────┘
```

### DOM 树构建详解

**文件**: `packages/page-controller/src/dom/dom_tree/index.js`

```
buildDomTree(document.body)
       │
       ▼
┌─────────────────────────────────────────┐
│  遍历每个 DOM 节点                       │
│                                         │
│  检查项:                                │
│  - isElementAccepted() - 是否应处理     │
│  - isElementVisible() - 是否可见         │
│  - isTopElement() - 是否顶层元素         │
│  - isInteractiveElement() - 是否可交互  │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  交互元素检测逻辑                        │
│                                         │
│  1. 标签名检测:                         │
│     a, button, input, select,           │
│     textarea, details, summary, label    │
│                                         │
│  2. Cursor 检测:                        │
│     pointer, move, text, grab...         │
│                                         │
│  3. ARIA 属性检测:                      │
│     role, aria-expanded, aria-checked.. │
│                                         │
│  4. 事件监听器检测:                     │
│     onclick, onmousedown...            │
│                                         │
│  5. 可滚动容器检测:                     │
│     overflow: auto/scroll              │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  高亮索引分配                           │
│                                         │
│  条件:                                  │
│  - 元素是交互元素                       │
│  - 元素在视口内 (或 viewportExpansion=-1)│
│  - 父元素未被高亮 或 元素是独立交互      │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  生成简化 HTML                          │
│  (用于 LLM 理解页面结构)                │
└─────────────────────────────────────────┘
```

### 元素可交互性判断

```typescript
function isInteractiveElement(element) {
    // 1. 黑名单检查
    if (interactiveBlacklist.includes(element)) return false
    if (interactiveWhitelist.includes(element)) return true

    // 2. Cursor 样式检测 (最有效)
    const interactiveCursors = ['pointer', 'move', 'text', 'grab', ...]
    if (interactiveCursors.has(style.cursor)) return true

    // 3. 标签名检测
    const interactiveTags = ['a', 'button', 'input', 'select', ...]
    if (interactiveTags.has(tagName)) return true

    // 4. ARIA role 检测
    const interactiveRoles = ['button', 'menuitem', 'tab', ...]
    if (interactiveRoles.has(role)) return true

    // 5. 内容编辑检测
    if (element.isContentEditable) return true

    // 6. 事件监听器检测
    if (hasClickListener(element)) return true

    // 7. 可滚动容器检测
    if (isScrollableElement(element)) return true

    return false
}
```

---

## 2. 元素操作流程 (Actions)

### clickElement

```typescript
async function clickElement(element: HTMLElement) {
    // 1. 模糊上一个点击的元素
    blurLastClickedElement()

    // 2. 滚动到可视区域
    await scrollIntoViewIfNeeded(element)

    // 3. 移动指针到元素中心
    await movePointerToElement(element, x, y)
    await clickPointer()

    // 4. Hit-test 检测实际点击目标
    const hitTarget = doc.elementFromPoint(x, y)
    const target = hitTarget instanceof HTMLElement && element.contains(hitTarget)
        ? hitTarget
        : element

    // 5. 触发完整的事件序列 (W3C 规范顺序)
    target.dispatchEvent(new PointerEvent('pointerover', pointerOpts))
    target.dispatchEvent(new PointerEvent('pointerenter', pointerOpts))
    target.dispatchEvent(new MouseEvent('mouseover', mouseOpts))
    target.dispatchEvent(new PointerEvent('pointerdown', pointerOpts))
    target.dispatchEvent(new MouseEvent('mousedown', mouseOpts))

    // 6. 聚焦元素
    element.focus({ preventScroll: true })

    // 7. 触发释放事件
    target.dispatchEvent(new PointerEvent('pointerup', pointerOpts))
    target.dispatchEvent(new MouseEvent('mouseup', mouseOpts))

    // 8. 触发 click (激活行为: 导航、表单提交等)
    target.click()
}
```

**事件触发顺序** (符合 W3C Pointer Events 规范):
```
pointerover → pointerenter → mouseover → mouseenter
    ↓
pointerdown → mousedown
    ↓
[focus]
    ↓
pointerup → mouseup
    ↓
click
```

### inputTextElement

```typescript
async function inputTextElement(element: HTMLElement, text: string) {
    // 1. 先点击元素
    await clickElement(element)

    // 2. 根据元素类型处理
    if (element.isContentEditable) {
        // 富文本编辑器处理 (Plan A + Plan B fallback)
        // Plan A: 派发 synthetic events
        // Plan B: execCommand (deprecated 但广泛支持)
    } else {
        // 普通输入框: 直接设置 value
        getNativeValueSetter(element).call(element, text)
        element.dispatchEvent(new Event('input', { bubbles: true }))
    }

    // 3. 触发 blur
    element.blur()
}
```

### scrollVertically

```typescript
async function scrollVertically(scroll_amount, element?: HTMLElement) {
    if (element) {
        // 元素内滚动
        // 向上遍历找可滚动容器
        while (currentElement && attempts < 10) {
            if (hasScrollableY && canScrollVertically) {
                // 执行滚动
                currentElement.scrollTop = beforeScroll + scrollAmount
                return `Scrolled container (${tagName}) by ${scrollDelta}px`
            }
            currentElement = currentElement.parentElement
        }
    } else {
        // 页面级滚动
        if (isScrollingElement) {
            window.scrollBy(0, dy)
            return `✅ Scrolled page by ${scrolled}px.`
        } else {
            // 回退到容器滚动
            el.scrollBy({ top: dy, behavior: 'smooth' })
        }
    }
}
```

---

## 3. BrowserState 生成

**文件**: `packages/page-controller/src/PageController.ts:129-165`

```typescript
async getBrowserState(): Promise<BrowserState> {
    // 1. 获取基本信息
    const url = window.location.href
    const title = document.title

    // 2. 更新 DOM 树 (自动调用 updateTree)
    await this.updateTree()

    // 3. 构建 header
    const header = `
Current Page: [${title}](${url})

Page info: ${viewport_width}x${viewport_height}px viewport,
${page_width}x${page_height}px total page size,
${pages_above.toFixed(1)} pages above,
${pages_below.toFixed(1)} pages below,
${total_pages.toFixed(1)} total pages,
at ${(current_page_position * 100).toFixed(0)}% of page

Interactive elements from top layer of the current page:
${scrollHintAbove}
    `.trim()

    // 4. content = simplifiedHTML
    const content = this.simplifiedHTML

    // 5. 构建 footer
    const footer = hasContentBelow
        ? `... ${pixels_below} pixels below (${pages_below.toFixed(1)} pages) - scroll to see more ...`
        : '[End of page]'

    return { url, title, header, content, footer }
}
```

**返回示例**:
```
Current Page: [Google](https://www.google.com)

Page info: 1920x1080px viewport, 1920x5000px total page size, 0.0 pages above, 3.6 pages below, 4.6 total pages, at 0% of page

Interactive elements from top layer of the current page:

[Start of page]

<div data-interactve="true">
  <input type="text" name="q" placeholder="Search..." />
  <button type="submit">Search</button>
</div>
...

[End of page]
```

---

## 4. 遮罩系统 (SimulatorMask)

### 作用

在 Agent 执行自动化任务期间，遮罩层覆盖整个页面：
- 阻止用户的鼠标/键盘操作干扰自动化流程
- 显示高亮元素供可视化调试

### 生命周期

```typescript
// 初始化 (可选，需要 enableMask: true)
initMask() {
    this.maskReady = (async () => {
        const { SimulatorMask } = await import('./mask/SimulatorMask')
        this.mask = new SimulatorMask()
    })()
}

// 显示遮罩
async showMask() {
    await this.maskReady
    this.mask?.show()
}

// 隐藏遮罩
async hideMask() {
    await this.maskReady
    this.mask?.hide()
}

// 销毁
dispose() {
    this.mask?.dispose()
    this.mask = null
}
```

### DOM 提取期间的特殊处理

```typescript
// updateTree() 期间临时禁用遮罩
if (this.mask) {
    this.mask.wrapper.style.pointerEvents = 'none'  // 允许 DOM 提取通过
}
// ... DOM 提取 ...
if (this.mask) {
    this.mask.wrapper.style.pointerEvents = 'auto'  // 恢复阻止
}
```

---

## 5. 索引操作流程

### 从 LLM 指令到页面操作

```
LLM 返回: { action: "click_element_by_index", input: { index: 5 } }
                    │
                    ▼
PageAgentCore.click_element_by_index(5)
                    │
                    ▼
pageController.clickElement(5)
                    │
                    ▼
getElementByIndex(selectorMap, 5)  // 查找索引 5 的元素
                    │
                    ▼
clickElement(element)  // 执行实际点击
                    │
                    ▼
返回 ActionResult
```

### assertIndexed 保护

```typescript
private assertIndexed(): void {
    if (!this.isIndexed) {
        throw new Error('DOM tree not indexed yet. Can not perform actions on elements.')
    }
}
```

**防止在页面状态未知的情况下执行操作**

---

## 6. 错误处理

### Action 级别的错误处理

```typescript
async clickElement(index: number): Promise<ActionResult> {
    try {
        this.assertIndexed()
        const element = getElementByIndex(this.selectorMap, index)
        await clickElement(element)

        return { success: true, message: `✅ Clicked element (${index}).` }
    } catch (error) {
        return { success: false, message: `❌ Failed to click element: ${error}` }
    }
}
```

**特点**: 返回结构化的成功/失败结果，不抛出异常

### 错误类型

| 错误 | 原因 |
|------|------|
| `No interactive element found at index` | 索引超出范围 |
| `Element is not an input/textarea/contenteditable` | 元素不支持文本输入 |
| `Option with text "X" not found` | 下拉框选项不存在 |
| `Failed to click element: ...` | 点击执行失败 |

---

## 7. React 补丁

**文件**: `packages/page-controller/src/patches/react.ts`

PageController 初始化时会对 React 进行补丁：

```typescript
patchReact(this)

// 可能的补丁内容:
// - 强制 React 重新渲染受控组件
// - 修补 React 事件系统以支持 synthetic events
// - 确保 React 正确处理外部触发的 DOM 操作
```

---

## 8. 完整交互流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        PageAgentCore                            │
│                                                                  │
│  execute(task)                                                   │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Observe 阶段                                              │ │
│  │  pageController.getBrowserState()                          │ │
│  │      │                                                      │ │
│  │      ▼                                                      │ │
│  │  updateTree() → simplifiedHTML → LLM                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Think 阶段 (LLM 返回 action)                              │ │
│  └─────────────────────────────────────────────────────────────┘ │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Act 阶段                                                   │ │
│  │      │                                                      │ │
│  │      ├── click_element_by_index → pageController.click()   │ │
│  │      ├── input_text → pageController.inputText()           │ │
│  │      ├── scroll → pageController.scroll()                  │ │
│  │      └── ...                                                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│      │                                                           │
│      ▼                                                           │
│  Loop (回到 Observe)                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. 性能优化

### 缓存机制

```typescript
const DOM_CACHE = {
    boundingRects: new WeakMap(),    // 边界矩形缓存
    clientRects: new WeakMap(),      // 客户端矩形缓存
    computedStyles: new WeakMap(),   // 计算样式缓存
    clearCache: () => { ... }
}
```

### 可滚动检测优化

```typescript
// 首次检测时缓存结果
function isScrollableElement(element) {
    const scrollData = { top, right, bottom, left }
    addExtraData(element, { scrollable: true, scrollData })
    return scrollData
}
```

---

## 10. 配置选项

```typescript
interface PageControllerConfig {
    // DOM 配置
    interactiveBlacklist?: Element[]   // 交互元素黑名单
    interactiveWhitelist?: Element[]   // 交互元素白名单
    includeAttributes?: string[]       // 额外包含的属性
    keepSemanticTags?: boolean         // 保留语义标签

    // 视口配置
    viewportExpansion?: number        // 视口扩展像素数 (-1=全页)

    // 遮罩配置
    enableMask?: boolean              // 启用视觉遮罩
}
```
