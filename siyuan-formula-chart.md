# SiYuan 公式与图表渲染代码实现分析

## 一、概述

SiYuan（思源笔记）的公式与图表渲染系统构建在 **Protyle** 富文本编辑器内核之上，采用「内容识别 → 任务调度 → 资源加载 → 异步渲染 → 状态标记 → 错误降级」的协作流水线。本文档系统分析其各环节协作机制，特别聚焦异步渲染、缓存利用、降级显示和安全边界。

---

## 二、核心架构概览

### 2.1 渲染引擎清单

SiYuan 支持以下 8 种「特殊渲染代码块」，语言标识通过 `data-subtype` 属性挂载于 DOM：

| 渲染类型 | data-subtype 值 | 底层引擎 | 核心实现文件 |
|---|---|---|---|
| 数学公式（KaTeX） | `math` | KaTeX 0.16.9 + mhchem | [mathRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts) |
| ECharts 图表 | `echarts` | ECharts 5.3.2 + echarts-gl 2.0.9 | [chartRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/chartRender.ts) |
| Mermaid 图 | `mermaid` | Mermaid 11.13.0 + zenuml | [mermaidRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts) |
| 思维导图 | `mindmap` | ECharts tree 系列 | [mindmapRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mindmapRender.ts) |
| Graphviz | `graphviz` | Viz.js 3.11.0 | [graphvizRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/graphvizRender.ts) |
| 流程图（flowchart.js） | `flowchart` | flowchart.js 1.18.0 | [flowchartRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/flowchartRender.ts) |
| PlantUML | `plantuml` | plantuml-encoder + 远端服务 | [plantumlRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/plantumlRender.ts) |
| ABC 乐谱 | `abc` | abcjs 6.5.0 | [abcRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/abcRender.ts) |

> 另外 `NodeHTMLBlock`（HTML 块）和 `NodeBlockQueryEmbed`（嵌入块）走独立的 [htmlRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/htmlRender.ts) 和 [blockRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/blockRender.ts) 路径。

### 2.2 统一调度入口

所有渲染任务的调度入口是 [processCode.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/processCode.ts#L48-L73) 中的 `RENDER_MAP` 映射表和 `processRender()` 函数：

```
RENDER_MAP = {
  abc: abcRender,
  plantuml: plantumlRender,
  mermaid: mermaidRender,
  flowchart: flowchartRender,
  echarts: chartRender,
  mindmap: mindmapRender,
  graphviz: graphvizRender,
  math: mathRender,
}
```

`processRender(element)` 执行逻辑：
1. 若元素有 `data-subtype` 且匹配 RENDER_MAP → 执行对应渲染器
2. 若元素 `data-type === "NodeHTMLBlock"` → 执行 htmlRender
3. 否则遍历执行全部渲染器 + htmlRender（针对父容器批量渲染）

---

## 三、流程分解：渲染五阶段协作

### 3.1 阶段一：内容识别（语法解析 + DOM 标记）

#### 3.1.1 Lute 引擎语法解析

[setLute.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/setLute.ts) 配置了底层 Markdown 解析引擎 Lute 的渲染模式，关键配置：

| 配置项 | 作用 |
|---|---|
| `SetProtyleWYSIWYG(true)` | 启用所见即所得模式 |
| `SetInlineMath(true/false)` | 行级公式开关（受编辑器配置控制） |
| `SetInlineMathAllowDigitAfterOpenMarker(true)` | 允许 `$2` 等数字开头公式 |
| `SetSanitize(options.sanitize)` | HTML 消毒开关 |
| `SetKramdownIAL(true)` | 启用 IAL 块属性语法 |
| `SetSpin(true)` | 启用 Spin 双向转换（DOM ↔ 抽象语法树） |

Lute 解析后，代码块被标记为：
- 块级公式：`<div data-type="NodeMathBlock" data-subtype="math" data-content="...">`
- 行级公式：`<span data-type="inline-math" data-subtype="math" data-content="...">`
- 特殊代码块：`<div data-type="NodeCodeBlock" data-subtype="echarts/mermaid/..." data-content="...">`

#### 3.1.2 可渲染语言白名单

[constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/constants.ts#L841-L843) 定义：
```typescript
SIYUAN_RENDER_CODE_LANGUAGES = [
  "abc", "plantuml", "mermaid", "flowchart", 
  "echarts", "mindmap", "graphviz", "math"
]
```

### 3.2 阶段二：渲染任务触发（43 处调用点）

`processRender()` 被广泛调用，据 grep 统计共 **43 处调用**，分布在以下关键路径：

| 触发场景 | 调用位置 | 触发时机 |
|---|---|---|
| 事务更新 | [transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/wysiwyg/transaction.ts) | 11 处，块新增/修改/删除/合并后 |
| 回车分裂 | [enter.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/wysiwyg/enter.ts) | 6 处，回车拆分块 |
| 编辑器内联编辑 | [toolbar/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/toolbar/index.ts) | 5 处，公式/图表浮层编辑 input 事件 |
| 文档加载 | [onGet.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/onGet.ts) | 打开文档后全量渲染 |
| 粘贴操作 | [paste.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/paste.ts) | 3 处，粘贴内容后 |
| 嵌入块递归 | [blockRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/blockRender.ts#L120) | 嵌入块内容返回后 |
| 预览面板 | [preview/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/preview/index.ts) | 预览模式渲染 |
| 导出 | [export/util.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/export/util.ts) | PDF/Word 导出 |
| 提示面板 | [hint/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/hint/index.ts) | 自动补全提示 |

### 3.3 阶段三：资源加载（异步脚本 + 幂等注入）

#### 3.3.1 脚本加载机制：addScript

[addScript.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/addScript.ts#L21-L44) 实现了带缓存的脚本加载器：

```typescript
export const addScript = (path: string, id: string) => {
  return new Promise((resolve) => {
    // 缓存命中：id 已存在 DOM 中则直接 resolve
    if (document.getElementById(id)) {
      resolve(false);
      return false;
    }
    const scriptElement = document.createElement("script");
    scriptElement.src = path;
    scriptElement.async = true;
    document.head.appendChild(scriptElement);
    scriptElement.onload = () => {
      // 循环调用处理：加载完成后清除临时标签，再赋值 id
      if (document.getElementById(id)) {
        scriptElement.remove();
        resolve(false);
        return false;
      }
      scriptElement.id = id;  // id 作为全局缓存标记
      resolve(true);
    };
  });
};
```

**关键设计：**
- 以 `id`（如 `protyleKatexScript`、`protyleEchartsScript`）作为全局唯一标识
- **Promise 化**：所有渲染器通过 `.then()` 链式等待脚本就绪
- **幂等**：重复调用直接 resolve，不重复网络请求
- **async=true**：不阻塞主渲染线程

#### 3.3.2 样式加载：addStyle

[addStyle.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/addStyle.ts#L1-L15) 以 `id` 为 key 防止 CSS 重复注入。

#### 3.3.3 资源依赖示例（层级加载）

以数学公式为例，采用**三级串行加载链**（[mathRender.ts#L19-L22](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L19-L22)）：

```
katex.min.css → katex.min.js → mhchem.min.js → 开始渲染
```

以 Mermaid 为例（[mermaidRender.ts#L16-L26](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L16-L26)）：

```
mermaid.min.js → mermaid-zenuml.min.js → registerExternalDiagrams → registerIconPacks → fetch icons.json → initialize(config) → 开始渲染
```

### 3.4 阶段四：异步渲染执行

#### 3.4.1 通用渲染骨架（所有渲染器共通模式）

```typescript
export const xxxRender = (element: Element, cdn = Constants.PROTYLE_CDN) => {
  // 1. DOM 查询：仅选择 data-render != "true" 的节点
  let elements = element.querySelectorAll('[data-subtype="xxx"]:not([data-render="true"])');
  if (elements.length === 0) return;

  // 2. 资源加载（Promise 链）
  addScript(`${cdn}/js/xxx/xxx.min.js`, "protyleXxxScript").then(() => {
    elements.forEach(async (e: HTMLElement) => {
      // 3. 立即标记渲染状态（防重复，请求返回前就标记）
      e.setAttribute("data-render", "true");

      // 4. 生成操作图标（刷新/编辑/更多）
      if (!e.firstElementChild.classList.contains("protyle-icons")) {
        e.insertAdjacentHTML("afterbegin", genIconHTML(wysiswgElement));
      }

      try {
        // 5. 获取并反转义内容
        const content = Lute.UnEscapeHTMLStr(e.getAttribute("data-content"));
        // 6. 调用引擎渲染
        const result = await window.Engine.render(content);
        // 7. 注入 DOM（contenteditable="false" 防止编辑）
        renderElement.innerHTML = `<div contenteditable="false">${result}</div>`;
      } catch (error) {
        // 8. 错误降级：显示 ft__error
        renderElement.innerHTML = `<div class="ft__error">render error: <br>${error}</div>`;
      }
    });
  });
};
```

#### 3.4.2 隐藏元素的延迟渲染（Mermaid / Flowchart）

针对折叠块、卡片面板中的不可见元素，**clientWidth=0** 导致 SVG 尺寸错误。Mermaid 和 Flowchart 采用 `MutationObserver` 延迟渲染策略（[mermaidRender.ts#L51-L76](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L51-L76)）：

```
元素分组：
  ├─ 可见元素（clientWidth>0） → 立即 initMermaid()
  └─ 隐藏元素（clientWidth=0） → 挂载 MutationObserver
        ├─ 监听折叠块 fold 属性变化
        └─ 监听卡片 class 变化
        └─ 变化触发后，一次性执行 initMermaid() 并 disconnect observer
```

#### 3.4.3 KaTeX 行级/块级差异化处理

[mathRender.ts#L30-L102](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L30-L102) 中针对 `isBlock = (tagName === "DIV")` 走不同路径：

| 特性 | 块级公式 (DIV) | 行级公式 (SPAN) |
|---|---|---|
| 渲染框架 | `genRenderFrame()` 生成双层嵌套 | 直接替换 innerHTML |
| 光标处理 | 内建 protyle-cursor | 前后插入 ZWSP / 换行符（10 处边界条件） |
| 溢出处理 | 自动换行处理 + base 元素占位符 | max-width:100% + overflow-x:auto |
| 样式修复 | katex-html newline display:block | 检测块宽度超限切换内联滚动 |

#### 3.4.4 导出 PDF 时的字体缩放

[mathRender.ts#L105-L118](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L105-L118) 针对 `maxWidth=true`（导出场景）：
- 块级公式：按 `clientWidth/scrollWidth` 比例缩小 font-size
- 行级公式：按 `blockWidth/offsetWidth` 比例缩小

### 3.5 阶段五：错误展示与降级显示

#### 3.5.1 统一降级样式

所有渲染器错误统一使用 CSS 类 `ft__error` 标记，实现视觉一致性：

| 渲染器 | 降级展示方式 |
|---|---|
| mathRender | 块级/行级均设置 `classList.add("ft__error")`，显示 e.message |
| chartRender | `echarts.dispose()` 后显示 `ft__error` + 保留容器高度 |
| mermaidRender | 先渲染错误节点 outerHTML，再 `<div class="ft__error">` + 横线分隔 |
| graphvizRender | Promise `.catch()` 捕获异步错误，显示 ft__error |
| flowchartRender | try/catch 同步捕获，显示 ft__error |
| plantumlRender | object 加载失败自动 fallback 为 `<img>`，编码错误显示 ft__error |
| mindmapRender | echarts.dispose() 后 ft__error，保留 420px 默认高度 |
| abcRender | 无显式 try/catch（依赖 abcjs 内部异常抛出） |

#### 3.5.2 空内容降级

所有渲染器首先检测 `data-content` 是否为空，空则只渲染零宽占位符 `Constants.ZWSP`（`\u200b`），不调用引擎。

---

## 四、模块间关系与协作

### 4.1 核心协作图

```
                    ┌────────────────────┐
                    │  用户输入/粘贴/加载  │
                    └─────────┬──────────┘
                              │
                              ▼
┌────────────┐    ┌─────────────────────────┐    ┌─────────────────┐
│  Lute 引擎 │───▶│   setLute 配置 / Spin    │───▶│  DOM 节点生成    │
│  (解析层)   │    └─────────────────────────┘    │  data-subtype   │
└────────────┘                                   └────────┬────────┘
                                                          │
                                                          ▼
                    ┌─────────────────────────────────────────────┐
                    │           processRender() 调度中心           │
                    │  processCode.ts  RENDER_MAP 8 路分发        │
                    └────────┬───────────┬───────────┬────────────┘
                             │           │           │
                    ┌────────▼──┐  ┌─────▼────┐  ┌──▼─────────┐
                    │  addScript│  │ addStyle │  │ addScriptSync│
                    │  (异步)   │  │  (CSS)   │  │  (同步)     │
                    └────────┬──┘  └─────┬────┘  └────────────┘
                             │           │
                             ▼           ▼
                    ┌─────────────────────────┐
                    │  各渲染引擎（KaTeX 等）  │
                    │  Promise.then(async)    │
                    └────────┬────────────────┘
                             │
              ┌──────────────┼──────────────────┐
              │              │                  │
              ▼              ▼                  ▼
     ┌─────────────┐ ┌───────────────┐ ┌───────────────┐
     │ 渲染成功     │ │ 渲染失败       │ │ 隐藏元素延迟   │
     │ DOM 注入     │ │ ft__error 降级 │ │ MutationObserver│
     │ contenteditable│               │ │ 监听折叠/展示   │
     │ ="false"     │ │               │ │                │
     └──────┬──────┘ └───────────────┘ └───────┬───────┘
            │                                   │
            ▼                                   ▼
     ┌──────────────────────────────────────────────────┐
     │  data-render="true"  ──▶  后续调用跳过该节点      │
     └──────────────────────────────────────────────────┘
```

### 4.2 关键协作模式

#### 4.2.1 编辑态 ↔ 渲染态双向切换

[toolbar/index.ts#L1050-L1180](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/toolbar/index.ts#L1050-L1180) 实现了编辑浮层（showRender）：

```
用户点击公式/图表
  → toolbar.showRender() 弹出 textarea 编辑器
  → input 事件触发：
      1. 更新 data-content = Lute.EscapeHTMLStr(value)
      2. **移除 data-render 属性**（关键！标记需重新渲染）
      3. processRender(renderElement) 重新渲染
  → Esc / ⌘↩ 关闭浮层：
      1. noChange 检测（oldTextValue === value）
      2. inline-math 空值：outerHTML = "<wbr>" 自销毁
```

#### 4.2.2 data-render 状态机

`data-render` 属性是防止重复渲染的核心状态标记：

| 状态 | 值 | 含义 | 触发操作 |
|---|---|---|---|
| 待渲染 | 不存在 / "false" | 需要执行渲染 | 编辑后 removeAttribute("data-render") |
| 渲染中 | "true"（脚本加载后立即设置） | **请求返回前就标记** | 第一行就 `setAttribute("data-render", "true")` |
| 渲染完成 | "true" | 已完成，跳过 | 下次 processRender 时 `:not([data-render="true"])` 过滤 |

> ⚠️ 关键设计：在 [blockRender.ts#L23-L24](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/blockRender.ts#L23-L24) 注释特别强调——**必须在异步请求发出前就标记 `data-render="true"`**，否则快速滚动会导致重复请求。

#### 4.2.3 ZWSP（零宽空格）光标锚点

`Constants.ZWSP = "\u200b"` 被大量用于渲染元素的边界锚点，解决光标无法定位到渲染元素前后的浏览器兼容性问题。[mathRender.ts#L69-L101](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L69-L101) 中有至少 8 种边界场景需要插入 ZWSP 或 `\n`。

---

## 五、缓存利用策略

### 5.1 三级缓存体系

| 缓存层级 | 实现机制 | 生命周期 |
|---|---|---|
| **L1 DOM 级** | `document.getElementById(id)` 检测脚本/样式标签 | 页面会话级 |
| **L2 元素级** | `data-render="true"` 属性标记 | 元素存在期（编辑时移除重渲染） |
| **L3 浏览器级** | `<script async>` + 浏览器 HTTP 缓存（URL 带版本号 `?v=x.y.z`） | 跨会话 |

### 5.2 URL 版本号控制

所有静态资源 URL 均显式追加版本号：
```
/stage/protyle/js/katex/katex.min.js?v=0.16.9
/stage/protyle/js/echarts/echarts.min.js?v=5.3.2
/stage/protyle/js/mermaid/mermaid.min.js?v=11.13.0
```

确保升级版本时强制绕过浏览器缓存。

### 5.3 ECharts 实例复用

[chartRender.ts#L40-L47](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/chartRender.ts#L40-L47)：
```typescript
const chartInstance = window.echarts.getInstanceById(dom._echarts_instance_);
if (chartInstance) {
  if (chartInstance.getOption().series[0]?.type !== option.series[0]?.type) {
    chartInstance.clear();  // 类型变化才清，否则复用
  }
  chartInstance.resize();
}
```

---

## 六、安全边界分析

### 6.1 DOMPurify XSS 防御

Mermaid 渲染后对 SVG 输出进行严格清洗（[mermaidRender.ts#L101-L108](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L101-L108)）：

```typescript
let svg = mermaidData.svg.replace(/(href|src|xlink:href)\s*=\s*["']\\\\/gi, 
  (match, p1) => `${p1}="about:blank"`);  // 外链替换为 about:blank
svg = window.DOMPurify.sanitize(svg, {
  USE_PROFILES: { svg: true, svgFilters: true },
  ADD_TAGS: ["foreignObject", "use", "style"],
  ADD_ATTR: ["dominant-baseline", "xlink:href", "href"],
  HTML_INTEGRATION_POINTS: { foreignobject: true }
});
```

### 6.2 KaTeX trust 模式权衡

[mathRender.ts#L36](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L36)：
```typescript
trust: true,  // 允许 \href, \includegraphics 等 HTML 扩展功能
strict: (errorCode) => errorCode === "unicodeTextInMathMode" ? "ignore" : "warn",
```

**风险点：** `trust: true` 开启了 KaTeX 的原始 URL 链接和 HTML 扩展，需依赖上层 sanitize。

### 6.3 Mermaid securityLevel: "loose"

[mermaidRender.ts#L27](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L27)：
```typescript
securityLevel: "loose",  // 注释：升级后无 #3587 问题，可使用该选项
```

loose 模式允许 Mermaid 使用 `<script>` 标签和点击事件，依赖 SiYuan 上游 sanitize 和下游 DOMPurify 双层保护。

### 6.4 looseJsonParse Function 沙箱

[functions.ts#L85-L87](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/util/functions.ts#L85-L87)：
```typescript
export const looseJsonParse = (text: string) => {
  return Function(`"use strict";return (${text})`)();
};
```

用于解析 ECharts option、KaTeX macros、ABC 乐谱参数等**非标准 JSON**（允许函数/表达式）。采用 `"use strict"` 但运行在全局作用域 Function 中，存在代码执行风险。

**调用点：**
- [chartRender.ts#L41](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/chartRender.ts#L41)：ECharts option 解析（**未 try/catch**，async 函数外层 catch）
- [mathRender.ts#L25-L29](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L25-L29)：KaTeX macros 解析（有 try/catch，失败用空对象）
- [abcRender.ts#L18-L23](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/abcRender.ts#L18-L23)：ABC params 解析（有 try/catch，失败用默认）

### 6.5 Lute SetSanitize 消毒

[setLute.ts#L20](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/setLute.ts#L20) `SetSanitize(options.sanitize)` 控制 Lute 输出的 HTML 消毒，影响所有渲染前的内容生成。

### 6.6 PlantUML 远端服务风险

[plantumlRender.ts#L29](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/plantumlRender.ts#L29)：
```typescript
const url = `${window.siyuan.config.editor.plantUMLServePath}${window.plantumlEncoder.encode(...)}`;
```

用户可自定义 `plantUMLServePath`，若指向恶意服务器可能导致 SSRF 或内容注入；使用 `<object type="image/svg+xml">` 渲染，SVG 中可能携带脚本。

---

## 七、潜在风险清单

| 风险 ID | 风险描述 | 严重程度 | 位置 | 建议 |
|---|---|---|---|---|
| R-01 | looseJsonParse 使用全局 Function，ECharts option 中可注入任意 JS | 🔴 高 | [functions.ts#L85](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/util/functions.ts#L85) | 改为白名单 AST 解析，或在沙箱 Worker 中执行 |
| R-02 | Mermaid `securityLevel: "loose"` 配合 `htmlLabels: true` 允许图表中注入点击事件 | 🟠 中 | [mermaidRender.ts#L27](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L27) | 评估是否仍需 loose；已用 DOMPurify 二次清洗，需验证覆盖 |
| R-03 | KaTeX `trust: true` 允许 `\href` 指向 `javascript:` 协议 | 🟠 中 | [mathRender.ts#L36](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L36) | 配置 trust 的 context 检查函数过滤 javascript: |
| R-04 | PlantUML 用户配置远端服务 + `<object>` 渲染 SVG | 🟠 中 | [plantumlRender.ts#L29-L34](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/plantumlRender.ts#L29-L34) | 对 SVG 输出追加 DOMPurify sanitize；默认服务 URL 白名单 |
| R-05 | blockRender 中 `new Function` 执行 `//!js` 自定义嵌入脚本 | 🔴 高 | [blockRender.ts#L44-L82](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/blockRender.ts#L44-L82) | 已暴露 fetchSyncPost/item/protyle/top 4 个变量，需文档化并考虑加签名 |
| R-06 | MutationObserver 未超时释放，若折叠状态永不改变则内存泄漏 | 🟡 低 | [mermaidRender.ts#L61-L64](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L61-L64) | 加 30s 超时自动 disconnect |
| R-07 | 公式渲染异常时，错误信息直接 innerHTML 注入（`e.message`） | 🟡 低 | [mathRender.ts#L119-L129](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L119-L129) | 异常信息先 escapeHtml，防止 KaTeX 内部错误带 HTML 片段 |
| R-08 | addScript 加载失败无 timeout，Promise 永久 pending | 🟡 低 | [addScript.ts#L21-L44](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/addScript.ts#L21-L44) | 添加 script.onerror 和超时机制 |
| R-09 | abcRender 缺少 try/catch，abcjs 解析异常会中断循环 | 🟡 低 | [abcRender.ts#L47](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/abcRender.ts#L47) | 补充 try/catch，同其他渲染器保持一致 |
| R-10 | Mermaid icons.json 通过 fetch 加载，返回未 sanitize | 🟡 低 | [mermaidRender.ts#L22-L24](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L22-L24) | 图标数据校验 JSON 结构 |

---

## 八、后续检查清单（Checklist）

### 8.1 安全审查
- [ ] 验证所有进入 innerHTML 的 `e.message` 是否经过 escapeHtml
- [ ] 检查 `looseJsonParse` 是否可改为 JSON5 + AST 函数白名单模式
- [ ] 审查 Mermaid DOMPurify 配置：`onclick`/`onmouseover` 等事件属性是否被过滤
- [ ] 为 KaTeX `trust` 选项提供自定义 context 校验函数
- [ ] PlantUML SVG 结果添加 DOMPurify 清洗
- [ ] 确认 Lute `SetSanitize` 在发布/预览/导出三种模式下配置一致

### 8.2 性能与缓存
- [ ] 审计 data-render=true 标记是否过早（如脚本加载失败后无法重渲染）
- [ ] 为 addScript / addStyle 添加 onerror 回退和版本降级逻辑
- [ ] 评估 Mermaid registerIconPacks 是否可缓存到 localStorage
- [ ] 检查 ECharts mindmap 是否存在实例未 dispose 的内存泄漏
- [ ] MutationObserver 增加超时自动 disconnect

### 8.3 错误处理
- [ ] 补充 abcRender 的 try/catch 降级
- [ ] plantumlRender `<img>` fallback 的错误事件处理
- [ ] 统一错误 UI 模板，增加「点击查看详情」折叠
- [ ] 收集渲染错误上报（仅开发/内测环境）

### 8.4 编辑协作
- [ ] 验证内联公式空内容自销毁 `<wbr>` 的所有光标路径
- [ ] 测试行级公式与相邻元素（img、inline-math、th/td）间 ZWSP 插入逻辑
- [ ] 检查 showRender 浮层关闭后 noChange=true 是否仍会错误 removeAttribute
- [ ] 导出 PDF 时 maxWidth 模式下的公式宽度重新计算准确性

### 8.5 可维护性
- [ ] 抽取通用渲染骨架函数（目前 8 个渲染器重复 70% 代码）
- [ ] RENDER_MAP 支持渲染器注册优先级
- [ ] 所有外部资源版本号集中管理（目前散落在各文件）
- [ ] 为 `//` 开头的 `%%params` 语法添加单元测试（abcRender）

---

## 九、代码参考索引

| 模块 | 文件路径 | 关键行号 |
|---|---|---|
| 渲染调度中心 | [processCode.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/processCode.ts#L48-L73) | L48-L73 |
| 数学公式渲染 | [mathRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mathRender.ts#L1-L133) | L1-L133 |
| ECharts 渲染 | [chartRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/chartRender.ts#L1-L56) | L1-L56 |
| Mermaid 渲染 | [mermaidRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/mermaidRender.ts#L1-L116) | L1-L116 |
| 脚本加载 | [addScript.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/util/addScript.ts#L21-L44) | L21-L44 |
| Lute 配置 | [setLute.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/setLute.ts#L1-L56) | L1-L56 |
| 编辑浮层 | [toolbar/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/toolbar/index.ts#L1050-L1180) | L1050-L1180 |
| 图标/框架生成 | [render/util.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/util.ts#L5-L43) | L5-L43 |
| 嵌入块渲染 | [blockRender.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/render/blockRender.ts#L12-L139) | L12-L139 |
| 常量定义 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/constants.ts#L832-L843) | L832-L843 |
| 工具函数 | [functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/util/functions.ts#L85-L87) | L85-L87 |
| 事务处理 | [transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/305-siyuan/app/src/protyle/wysiwyg/transaction.ts#L63-L80) | L63-L80 |

---

*文档生成时间：2026-06-15 · 基于 SiYuan 3.6.x 分支代码分析*
