# SiYuan 公式与图表渲染代码实现分析

## 一、概述

SiYuan（思源笔记）的公式与图表渲染系统构建在 **Protyle** 富文本编辑器内核之上，采用「内容识别 → 任务调度 → 资源加载 → 异步渲染 → 状态标记 → 错误降级」的协作流水线。本文档系统分析其各环节协作机制，特别聚焦异步渲染、缓存利用、降级显示和安全边界，所有结论均附代码证据链。

---

## 二、核心架构概览

### 2.1 渲染引擎清单

SiYuan 支持以下 8 种「特殊渲染代码块」，语言标识通过 `data-subtype` 属性挂载于 DOM 节点：

| 渲染类型 | data-subtype 值 | 底层引擎及版本 | 核心实现文件（仓库相对路径） |
|---|---|---|---|
| 数学公式（KaTeX） | `math` | KaTeX 0.16.9 + mhchem | `app/src/protyle/render/mathRender.ts` |
| ECharts 图表 | `echarts` | ECharts 5.3.2 + echarts-gl 2.0.9 | `app/src/protyle/render/chartRender.ts` |
| Mermaid 图 | `mermaid` | Mermaid 11.13.0 + zenuml 0.2.2 | `app/src/protyle/render/mermaidRender.ts` |
| 思维导图 | `mindmap` | ECharts tree 系列 | `app/src/protyle/render/mindmapRender.ts` |
| Graphviz | `graphviz` | Viz.js 3.11.0 | `app/src/protyle/render/graphvizRender.ts` |
| 流程图（flowchart.js） | `flowchart` | flowchart.js 1.18.0 | `app/src/protyle/render/flowchartRender.ts` |
| PlantUML | `plantuml` | plantuml-encoder + 可配置远端服务 | `app/src/protyle/render/plantumlRender.ts` |
| ABC 乐谱 | `abc` | abcjs 6.5.0 | `app/src/protyle/render/abcRender.ts` |

> 另外 `NodeHTMLBlock`（HTML 块）走 `app/src/protyle/render/htmlRender.ts`，`NodeBlockQueryEmbed`（嵌入块）走 `app/src/protyle/render/blockRender.ts`，二者不计入 8 种图表渲染器，但共享同一调度体系。

### 2.2 统一调度入口

所有渲染任务的调度入口是 `app/src/protyle/util/processCode.ts` 第 48-73 行定义的 `RENDER_MAP` 映射表和 `processRender()` 函数。

**证据链代码（processCode.ts:L48-L73）：**
```typescript
const RENDER_MAP: Record<string, (previewPanel: Element) => void> = {
    abc: abcRender,
    plantuml: plantumlRender,
    mermaid: mermaidRender,
    flowchart: flowchartRender,
    echarts: chartRender,
    mindmap: mindmapRender,
    graphviz: graphvizRender,
    math: mathRender,
};

export const processRender = (previewPanel: Element) => {
    const language = previewPanel.getAttribute("data-subtype");
    if (RENDER_MAP[language]) {
        RENDER_MAP[language](previewPanel);  // 单元素精确路由
        return;
    }
    if (previewPanel.getAttribute("data-type") === "NodeHTMLBlock") {
        htmlRender(previewPanel);
        return;
    }
    for (const render of Object.values(RENDER_MAP)) {
        render(previewPanel);  // 容器批量扫描
    }
    htmlRender(previewPanel);
};
```

`processRender(element)` 三路分发逻辑：
1. **精确路由**：元素有 `data-subtype` 且命中 RENDER_MAP → 仅执行对应渲染器
2. **HTML 块路由**：`data-type === "NodeHTMLBlock"` → 执行 htmlRender
3. **批量扫描**：否则遍历全部 8 个渲染器 + htmlRender（针对父容器场景，每个渲染器内部用 `querySelectorAll` 自行过滤）

---

## 三、流程分解：渲染五阶段协作

### 3.1 阶段一：内容识别（语法解析 + DOM 标记）

#### 3.1.1 Lute 引擎语法解析

`app/src/protyle/render/setLute.ts` 第 1-56 行配置了底层 Markdown 解析引擎 Lute 的渲染模式。

**证据链（setLute.ts:L13-L37，部分关键配置）：**
```typescript
lute.SetInlineMathAllowDigitAfterOpenMarker(true);   // L13: 允许 $2 这类数字开头公式
lute.SetSanitize(options.sanitize);                   // L20: HTML 消毒
lute.SetKramdownIAL(true);                            // L24: 启用 IAL 块属性
lute.SetInlineMath(window.siyuan.config.editor.markdown.inlineMath);  // L33: 行级公式开关
lute.SetProtyleWYSIWYG(true);                         // L38: 所见即所得模式
lute.SetSpin(true);                                   // L37: DOM ↔ AST 双向转换
```

经 Lute 解析 / Spin 转换后，特殊代码块在 DOM 中被标准化标记为：

| 节点类型 | DOM 结构特征 |
|---|---|
| 块级公式 | `<div data-type="NodeMathBlock" data-subtype="math" data-content="...">` |
| 行级公式 | `<span data-type="inline-math" data-subtype="math" data-content="...">` |
| 特殊代码块 | `<div data-type="NodeCodeBlock" data-subtype="echarts/mermaid/..." data-content="...">` |
| HTML 块 | `<div data-type="NodeHTMLBlock">` |
| 嵌入块 | `<div data-type="NodeBlockQueryEmbed" data-content="...">` |

#### 3.1.2 可渲染语言白名单

**证据链（`app/src/constants.ts`，L841-L843）：**
```typescript
public static readonly SIYUAN_RENDER_CODE_LANGUAGES: string[] = [
    "abc", "plantuml", "mermaid", "flowchart", "echarts", "mindmap", "graphviz", "math"
];
```

该白名单在以下位置被用于判断是否切换为渲染模式：
- `app/src/protyle/wysiwyg/enter.ts` L87-L99：回车将普通代码块切换为渲染块
- `app/src/protyle/toolbar/index.ts` L1830-L1835：语言切换时切换渲染模式

---

### 3.2 阶段二：渲染任务触发（精确统计：**14 个文件，共 33 处真实调用**）

> **统计说明**：grep `processRender` 返回 47 行结果，其中 **13 行是 import 语句**，**1 行是函数定义**（processCode.ts:L59），剩余 **33 行是真实调用**，分布于 **14 个源文件**。

#### 3.2.1 调用点全景分类

| 触发场景分类 | 文件数 | 调用次数 | 文件列表（仓库相对路径）及具体行号 |
|---|---|---|---|
| **事务更新** | 1 | 10 次 | `app/src/protyle/wysiwyg/transaction.ts` L115, L235, L315, L375, L429, L888, L1255, L1359, L1494 |
| **回车分裂块** | 1 | 6 次 | `app/src/protyle/wysiwyg/enter.ts` L93, L99, L483, L564, L571 |
| **编辑器浮层编辑** | 1 | 4 次 | `app/src/protyle/toolbar/index.ts` L1081, L1163, L1176, L1835 |
| **粘贴操作** | 1 | 3 次 | `app/src/protyle/util/paste.ts` L459, L561, L630 |
| **反链渲染** | 1 | 2 次 | `app/src/protyle/wysiwyg/renderBacklink.ts` L24, L79 |
| **文档初始加载** | 2 | 2 次 | `app/src/protyle/util/onGet.ts` L232<br>`app/src/mobile/util/MobileBackFoward.ts` L92 |
| **嵌入块递归渲染** | 1 | 1 次 | `app/src/protyle/render/blockRender.ts` L120 |
| **预览面板** | 1 | 1 次 | `app/src/protyle/preview/index.ts` L193 |
| **导出流程** | 1 | 1 次 | `app/src/protyle/export/util.ts` L166 |
| **自动补全提示** | 2 | 2 次 | `app/src/protyle/hint/index.ts` L865<br>`app/src/protyle/hint/extend.ts` L559 |
| **AV 画廊渲染** | 1 | 1 次 | `app/src/protyle/render/av/gallery/render.ts` L157 |
| **AI 内容填充** | 1 | 1 次 | `app/src/ai/actions.ts` L25 |
| **合计** | **14** | **33** | |

#### 3.2.2 各场景调用上下文（证据链抽样）

**事务更新触发（transaction.ts:L115）**——WebSocket 推送远端块更新后整体重渲染：
```typescript
Array.from(protyle.wysiwyg.element.querySelectorAll(`[data-node-id="${operation.id}"]`)).forEach(item => {
    if (!isInEmbedBlock(item)) {
        item.outerHTML = operation.data.replace("<wbr>", "");
    }
});
processRender(protyle.wysiwyg.element);   // L115
highlightRender(protyle.wysiwyg.element);
avRender(protyle.wysiwyg.element, protyle);
blockRender(protyle, protyle.wysiwyg.element);
```

**回车分裂触发（enter.ts:L93-L99）**——在代码块内回车、语言属于可渲染白名单时，立即切换为渲染模式：
```typescript
if (Constants.SIYUAN_RENDER_CODE_LANGUAGES.includes(languageElement.textContent)) {
    blockElement.dataset.content = "";
    blockElement.dataset.subtype = languageElement.textContent;
    blockElement.className = "render-node";
    blockElement.innerHTML = `<div spin="1"></div><div class="protyle-attr" contenteditable="false">${Constants.ZWSP}</div>`;
    protyle.toolbar.showRender(protyle, blockElement);
    processRender(blockElement);  // L93
} else {
    protyle.toolbar.showRender(protyle, blockElement);
    processRender(blockElement);  // L99
}
```

**浮层编辑 input 事件触发（toolbar/index.ts:L1076-L1081）**——用户在公式/图表编辑浮层中输入时实时重渲染：
```typescript
renderElement.setAttribute("data-content", Lute.EscapeHTMLStr(textElement.value));
renderElement.removeAttribute("data-render");   // 关键：先清除渲染标记
// ...
if (!types.includes("NodeBlockQueryEmbed") || !types.includes("NodeHTMLBlock") || !isInlineMemo) {
    processRender(renderElement);  // L1081
}
```

**AI 内容填充触发（ai/actions.ts:L23-L25）**——AI 生成内容插入后重渲染：
```typescript
insertHTML(protyle.lute.SpinBlockDOM(data), protyle, true, true);
blockRender(protyle, protyle.wysiwyg.element);
processRender(protyle.wysiwyg.element);  // L25
```

---

### 3.3 阶段三：资源加载（异步脚本 + 幂等注入）

#### 3.3.1 脚本加载机制：addScript

**证据链（`app/src/protyle/util/addScript.ts` L21-L44）：**
```typescript
export const addScript = (path: string, id: string) => {
    return new Promise((resolve) => {
        // 缓存命中：id 已存在 DOM 中则直接 resolve，不发网络请求
        if (document.getElementById(id)) {
            resolve(false);
            return false;
        }
        const scriptElement = document.createElement("script");
        scriptElement.src = path;
        scriptElement.async = true;      // 不阻塞主渲染线程
        document.head.appendChild(scriptElement);
        scriptElement.onload = () => {
            if (document.getElementById(id)) {
                scriptElement.remove();   // 循环调用时 Chrome 不会重复请求，但会产生重复 DOM 节点
                resolve(false);
                return false;
            }
            scriptElement.id = id;        // id 作为全局缓存标记（L1 DOM 级缓存）
            resolve(true);
        };
    });
};
```

**关键设计要素：**

| 要素 | 实现方式 | 证据行 |
|---|---|---|
| Promise 化异步 | `new Promise` + `onload` 回调 | L22, L33 |
| 幂等性（防重复加载） | `document.getElementById(id)` 前置检查 | L23-L27 |
| 非阻塞 | `scriptElement.async = true` | L30 |
| 循环调用兼容 | onload 时再次检查 id，冲突则移除临时节点 | L34-L39 |

#### 3.3.2 样式加载：addStyle

**证据链（`app/src/protyle/util/addStyle.ts` L1-L15）：**
```typescript
export const addStyle = (url: string, id: string) => {
    if (!document.getElementById(id)) {  // 同 addScript 一样，id 作缓存键
        const styleElement = document.createElement("link");
        styleElement.id = id;
        styleElement.rel = "stylesheet";
        // ...
        document.getElementsByTagName("head")[0].appendChild(styleElement);
    }
};
```

#### 3.3.3 资源依赖链（层级加载实例）

**数学公式三级加载链**（`app/src/protyle/render/mathRender.ts` L19-L22）：
```typescript
addStyle(`${cdn}/js/katex/katex.min.css?v=0.16.9`, "protyleKatexStyle");           // L19: CSS
addScript(`${cdn}/js/katex/katex.min.js?v=0.16.9`, "protyleKatexScript").then(() => {  // L20: 主库
    addScript(`${cdn}/js/katex/mhchem.min.js?v=0.16.9`, "protyleKatexMhchemScript")  // L21: 化学公式扩展
        .then(() => { /* 开始渲染 */ });
});
```

**Mermaid 五级加载链**（`app/src/protyle/render/mermaidRender.ts` L16-L50）：
```
mermaid.min.js (L16)
  → mermaid-zenuml.min.js (L17)
    → registerExternalDiagrams([zenuml]) (L18)
    → registerIconPacks → fetch icons.json (L19-L24)
    → initialize(config) (L50)
      → 开始渲染
```

---

### 3.4 阶段四：异步渲染执行

#### 3.4.1 通用渲染骨架（8 个渲染器的共通模式）

所有渲染器（除 htmlRender 外）均遵循以下骨架模式：

```typescript
export const xxxRender = (element: Element, cdn = Constants.PROTYLE_CDN) => {
  // 1. 目标节点筛选（:not([data-render="true"]) 防重复）
  let elements = element.querySelectorAll('[data-subtype="xxx"]:not([data-render="true"])');
  if (elements.length === 0) return;

  // 2. 资源加载（Promise 链）
  addScript(`${cdn}/js/xxx/xxx.min.js`, "protyleXxxScript").then(() => {
    elements.forEach(async (e: HTMLElement) => {
      // 3. 立即标记 data-render="true"（脚本加载后异步回调前就标记）
      e.setAttribute("data-render", "true");

      // 4. 操作图标注入
      if (!e.firstElementChild.classList.contains("protyle-icons")) {
        e.insertAdjacentHTML("afterbegin", genIconHTML(wysiswgElement, ["refresh", "edit", "more"]));
      }

      try {
        // 5. 内容反转义（Lute 保存时做了 EscapeHTMLStr）
        const content = Lute.UnEscapeHTMLStr(e.getAttribute("data-content"));
        // 6. 调用第三方引擎渲染
        const result = await window.Engine.render(content);
        // 7. 结果注入 DOM（统一 contenteditable="false" 防误编辑）
        renderElement.innerHTML = `<div contenteditable="false">${result}</div>`;
      } catch (error) {
        // 8. 错误降级：ft__error 样式
        renderElement.innerHTML = `<div class="ft__error">render error: <br>${error}</div>`;
      }
    });
  });
};
```

各渲染器实现该骨架的证据：

| 渲染器 | 目标节点筛选 | data-render 标记行 | try/catch 降级行 |
|---|---|---|---|
| mathRender | L11-L15 | L23 | L119-L129 |
| chartRender | L9-L13 | L25 | L49-L52 |
| mermaidRender | L8-L12 | L88 | L110-L114 |
| mindmapRender | L8-L12 | L23 | L81-L84 |
| graphvizRender | L8-L12 | L19 | L32-L34 |
| flowchartRender | L12-L16 | L57 | L69-L71 |
| plantumlRender | L8-L12 | L19 | L35-L38 |
| abcRender | L29-L33 | L40 | **缺失**（风险 R-09） |

#### 3.4.2 隐藏元素的延迟渲染（MutationObserver）

**问题**：折叠块（`fold="1"`）或卡片面板（`card__block`）中的图表 `clientWidth=0`，此时生成 SVG 尺寸为 0，展开后无法自动恢复。

**证据链**（`app/src/protyle/render/mermaidRender.ts` L51-L77，Flowchart 同构实现见 flowchartRender.ts L21-L47）：
```typescript
const hideElements: Element[] = [];    // 不可见元素
const normalElements: Element[] = [];  // 可见元素
mermaidElements.forEach(item => {
    if (item.firstElementChild.clientWidth === 0) {
        hideElements.push(item);
    } else {
        normalElements.push(item);
    }
});
if (hideElements.length > 0) {
    const observer = new MutationObserver(() => {
        initMermaid(hideElements);   // 状态变化触发延迟渲染
        observer.disconnect();       // 一次性：触发后立即解绑
    });
    hideElements.forEach(item => {
        const hideElement = hasClosestByAttribute(item, "fold", "1");
        if (hideElement) {
            observer.observe(hideElement, {attributeFilter: ["fold"]});  // 监听折叠属性
        } else {
            const cardElement = hasClosestByClassName(item, "card__block", true);
            if (cardElement) {
                observer.observe(cardElement, {attributeFilter: ["class"]});  // 监听卡片显示
            }
        }
    });
}
initMermaid(normalElements);  // 可见元素立即渲染
```

#### 3.4.3 KaTeX 行级/块级差异化处理

**证据链**（`app/src/protyle/render/mathRender.ts` L30-L102）：

| 特性 | 块级公式（`tagName === "DIV"`） | 行级公式（`tagName !== "DIV"`） | 证据行 |
|---|---|---|---|
| 渲染框架 | `genRenderFrame()` 生成双层嵌套 + protyle-cursor | 直接替换 innerHTML | L40-L42 vs L56-L58 |
| contenteditable | 强制 `="false"` | N/A | L43 |
| 溢出处理 | 自动换行 + `.base` 末尾插入 `fn__flex-1` 占位 | `max-width:100%` + `overflow-x:auto` | L46-L49 vs L58-L68 |
| 换行修复 | `.katex-html > .newline` 父元素 `display:block` | N/A | L51-L54 |
| 光标锚点 | 内建 protyle-cursor | 8 种边界场景插入 ZWSP / 换行符 | L69-L101 |

#### 3.4.4 导出 PDF 时的字体自适应缩放

**证据链**（`app/src/protyle/render/mathRender.ts` L105-L118）：
```typescript
if (maxWidth) {   // 仅导出 PDF 时 maxWidth=true
    setTimeout(() => {
        if (isBlock) {
            const katexElement = mathElement.querySelector(".katex-display");
            if (katexElement.clientWidth < katexElement.scrollWidth) {
                katexElement.firstElementChild.setAttribute(
                    "style", `font-size:${katexElement.clientWidth * 100 / katexElement.scrollWidth}%`
                );
            }
        } else {
            if (blockElement && mathElement.offsetWidth > blockElement.clientWidth) {
                mathElement.firstElementChild.setAttribute(
                    "style", `font-size:${blockElement.clientWidth * 100 / mathElement.offsetWidth}%`
                );
            }
        }
    });
}
```

---

### 3.5 阶段五：错误展示与降级显示

#### 3.5.1 统一降级样式 ft__error

grep 结果显示渲染相关代码中共 **13 处** `ft__error` 使用（不含通用 UI 场景），覆盖全部 8 个渲染器中的 7 个。

**各渲染器降级证据链：**

| 渲染器 | 降级展示代码 | 证据行 |
|---|---|---|
| mathRender（块级） | `mathElement.firstElementChild.firstElementChild.classList.add("ft__error"); mathElement.firstElementChild.firstElementChild.innerHTML = e.message;` | L122-L125 |
| mathRender（行级） | `mathElement.innerHTML = e.message; mathElement.classList.add("ft__error");` | L126-L128 |
| chartRender | `renderElement.innerHTML = \`<div class="ft__error" style="height:${e.style.height \|\| "420px"}">echarts render error: <br>${error}</div>\`` | L51 |
| mermaidRender | `renderElement.lastElementChild.innerHTML = ${errorElement.outerHTML}<div class="fn__hr"></div><div class="ft__error">${e.message.replace(/\n/, "<br>")}</div>` | L112 |
| mindmapRender | `renderElement.innerHTML = \`<div class="ft__error" style="height:${e.style.height \|\| "420px"}">Mindmap render error: <br>${error}</div>\`` | L83 |
| graphvizRender | `renderElement.innerHTML = \`<div class="ft__error">graphviz render error: <br>${error}</div>\`` | L33 |
| flowchartRender | `renderElement.innerHTML = \`<div class="ft__error">Flow Chart render error: <br>${error}</div>\`` | L70 |
| plantumlRender | `renderElement.classList.add("ft__error"); renderElement.innerHTML = \`plantuml render error: <br>${error}\`` | L36-L37 |
| abcRender | **缺失 try/catch** | — |

#### 3.5.2 空内容降级

所有渲染器首行检测 `data-content` 是否为空，空则只渲染零宽占位符 `Constants.ZWSP`（`\u200b`）：

```typescript
if (!e.getAttribute("data-content")) {
    renderElement.innerHTML = `<span style="position: absolute;left:0;top:0;width: 1px;">${Constants.ZWSP}</span>`;
    return;   // 不调用第三方引擎，直接返回
}
```

证据行：chartRender L30-L33、mermaidRender L93-L96、mindmapRender L28-L31、graphvizRender L24-L27、flowchartRender L62-L65、plantumlRender L24-L27、abcRender L44（abcRender 空内容检测略有不同但逻辑一致）。

#### 3.5.3 PlantUML 双层降级

**证据链**（`app/src/protyle/render/plantumlRender.ts` L29-L38）：
```typescript
const url = `${window.siyuan.config.editor.plantUMLServePath}${window.plantumlEncoder.encode(...)}`;
renderElement.innerHTML = `<object type="image/svg+xml" data="${url}"/>`;  // L30: 首选 SVG object
renderElement.firstElementChild.addEventListener("error", () => {
    renderElement.innerHTML = `<img src=${url}">`;   // L33: object 加载失败降级为 <img>
});
// ...
catch (error) {
    renderElement.classList.add("ft__error");        // L36-L37: 编码异常再降级为错误文本
    renderElement.innerHTML = `plantuml render error: <br>${error}`;
}
```

降级链：`SVG object` → `img 标签` → `ft__error 错误文本`

---

## 四、模块间关系与协作

### 4.1 核心协作流水线

```
                     ┌──────────────────────────┐
                     │  用户输入/粘贴/加载/推送   │
                     └────────────┬─────────────┘
                                  │
                                  ▼
┌──────────────┐     ┌─────────────────────────────┐     ┌────────────────────┐
│  Lute 引擎    │────▶│ setLute 配置 + Spin 双向转换 │────▶│ DOM 节点标准化标记  │
│ (Markdown 解析)│     └─────────────────────────────┘     │ data-subtype       │
└──────────────┘                                           │ data-content       │
                                                           │ data-type          │
                                                           └─────────┬──────────┘
                                                                     │
                                            14 个文件，33 处真实调用触发
                                                                     │
                                                                     ▼
                                   ┌───────────────────────────────────────────────┐
                                   │   processRender() 调度中心（三路分发）          │
                                   │   app/src/protyle/util/processCode.ts L59-L73  │
                                   └─────────┬───────────────┬───────────┬──────────┘
                                             │               │           │
                                   ┌─────────▼─────┐  ┌──────▼─────┐  ┌──▼─────────────┐
                                   │  addScript     │  │ addStyle   │  │ addScriptSync  │
                                   │  Promise 异步   │  │  CSS 注入   │  │ 同步阻塞加载   │
                                   │  id 全局缓存    │  │ id 去重    │  │ Lute 核心依赖   │
                                   └─────────┬─────┘  └──────┬─────┘  └────────────────┘
                                             │               │
                                             ▼               ▼
                                   ┌─────────────────────────────────┐
                                   │  第三方渲染引擎（8 种）          │
                                   │  Promise.then() + async/await   │
                                   └───────────────┬─────────────────┘
                                                   │
                          ┌────────────────────────┼──────────────────────────┐
                          │                        │                          │
                          ▼                        ▼                          ▼
               ┌──────────────────┐   ┌──────────────────────┐   ┌────────────────────┐
               │ 渲染成功          │   │ 渲染失败              │   │ 隐藏元素延迟渲染    │
               │ contenteditable   │   │ ft__error 样式降级    │   │ MutationObserver   │
               │ ="false" 保护     │   │ 错误信息注入 DOM      │   │ 监听 fold / class   │
               │ data-render="true"│   │ data-render="true"    │   │ 一次性 disconnect   │
               └────────┬─────────┘   └──────────┬───────────┘   └─────────┬──────────┘
                        │                        │                         │
                        └────────────────────────┼─────────────────────────┘
                                                 │
                                                 ▼
                                    ┌────────────────────────────────┐
                                    │ data-render="true"             │
                                    │ 下次 querySelectorAll 过滤跳过 │
                                    │ 编辑时 removeAttribute 重渲染   │
                                    └────────────────────────────────┘
```

### 4.2 data-render 状态机（防重复渲染核心）

`data-render` 属性是整个渲染协作体系的核心状态标记，全代码库共 **91 处**引用（grep 结果），其中：

| 操作类型 | 次数 | 典型场景 |
|---|---|---|
| **设置为 true** | 18 处 | 各渲染器 forEach 内首行；AV 渲染完成 |
| **查询 true / 非 true** | 26 处 | 各渲染器 `:not([data-render="true"])` 过滤 |
| **移除属性** | 47 处 | 编辑浮层 input；事务更新；回车；粘贴；代码语言切换；撤销重做等 |

**状态转换表：**

| 状态 | 属性值 | 含义 | 触发操作及证据 |
|---|---|---|---|
| 待渲染 | 属性不存在 / 非 "true" | 需要执行渲染 | 编辑浮层：`toolbar/index.ts` L1078 `removeAttribute("data-render")`<br>事务推送：`transaction.ts` L135, L189, L340 等 |
| 渲染中 / 完成 | "true" | 已渲染 / 正在渲染，跳过 | 各渲染器：`setAttribute("data-render", "true")` 如 mathRender L23 |

**关键设计证据**（`app/src/protyle/render/blockRender.ts` L23-L24）：
```typescript
// 需置于请求返回前，否则快速滚动会导致重复加载
// https://ld246.com/article/1666857862494?r=88250
item.setAttribute("data-render", "true");
```

> ⚠️ **核心原则**：`data-render="true"` 必须在**异步操作发起前**立即设置，而非在 then/catch 回调中。这是防止快速滚动、快速输入导致重复渲染 / 重复请求的关键。

### 4.3 编辑态 ↔ 渲染态双向切换

**证据链**（`app/src/protyle/toolbar/index.ts` L1050-L1180，浮层编辑流程）：

```
用户点击公式/图表块
  ↓
toolbar.showRender() 弹出 textarea 编辑器
  ↓
input 事件触发（L1052）：
  ├─ 更新 data-content = Lute.EscapeHTMLStr(value)   // L1077 / L1161 / L1170
  ├─ 移除 data-render 属性                            // L1078 / L1162 / L1171（关键标记）
  └─ 调用 processRender(renderElement) 实时重渲染     // L1081 / L1163 / L1176
  ↓
Esc / ⌘↩ 关闭浮层（L1095-L1098）：
  ├─ noChange 检测：oldTextValue === value 则不修改 DOM
  ├─ inline-math 空值 → outerHTML = "<wbr>" 自销毁   // L1165-L1167
  └─ 其他类型 → 保留 data-content，移除 data-render
```

### 4.4 ZWSP（零宽空格）光标锚点体系

`Constants.ZWSP = "\u200b"`（定义于 `app/src/constants.ts` L832）被系统性地用于解决渲染元素前后光标不可见的浏览器兼容性问题。仅 `mathRender.ts` L69-L101 就包含 **8 种边界场景**：

| 边界场景 | 处理方式 | 证据行 |
|---|---|---|
| 行级公式后无兄弟节点 + 非表格单元格 | `insertAdjacentText("afterend", "\n")` | L72-L75 |
| 行级公式后无兄弟节点 + 表格单元格 | `insertAdjacentText("afterend", Constants.ZWSP)` | L77-L79 |
| 相邻下一个兄弟是 inline-math 或 img | `after(document.createTextNode(Constants.ZWSP))` | L85-L87 |
| 下一个兄弟文本不以 `\n` 开头且非 ZWSP | `insertAdjacentHTML("beforeend", "&#xFEFF;")` | L90-L94 |
| 前一个兄弟文本以 `\n` 结尾 | `insertAdjacentText("beforebegin", Constants.ZWSP)` | L96-L98 |
| 无前兄弟 + 在表格单元格中 | `insertAdjacentText("afterbegin", Constants.ZWSP)` | L98-L101 |

---

## 五、缓存利用策略（三级缓存体系）

### 5.1 缓存层级全景

| 层级 | 实现机制 | 证据链 | 生命周期 |
|---|---|---|---|
| **L1 DOM 级**（全局脚本/样式缓存） | `document.getElementById(id)` 检查是否已注入 | addScript.ts L23-L27<br>addStyle.ts L2 | 页面会话级（刷新失效） |
| **L2 元素级**（防重复渲染） | `data-render="true"` 属性标记 + `querySelectorAll(':not([data-render="true"])')` 过滤 | 各渲染器首行，如 mathRender L11-L15 | 元素存在期；编辑时 `removeAttribute("data-render")` 失效 |
| **L3 浏览器级**（HTTP 缓存） | 资源 URL 显式追加版本号参数 `?v=x.y.z` | 各渲染器 addScript 调用，如 mathRender L19-L21 | 跨会话；版本号变更时强制失效 |

### 5.2 L3 资源版本号控制证据

所有外部资源 URL 均携带具体版本号，确保升级不命中陈旧缓存：

| 资源 | 版本号示例 | 证据行 |
|---|---|---|
| KaTeX CSS + JS + mhchem | `?v=0.16.9` | mathRender L19-L21 |
| ECharts + echarts-gl | `?v=5.3.2` + `?v=2.0.9` | chartRender L17-L18 |
| Mermaid | `?v=11.13.0` | mermaidRender L16 |
| Mermaid zenuml 扩展 | `?v=0.2.2` | mermaidRender L17 |
| Mermaid icons.json | `?v=11.11.0` | mermaidRender L23 |
| abcjs | `?v=6.5.0` | abcRender L37 |
| Graphviz Viz.js | `?v=3.11.0` | graphvizRender L16 |
| flowchart.js | `?v=1.18.0` | flowchartRender L20 |
| highlight.js code theme | `?v=11.11.1` | render/util.ts L66 |

### 5.3 L4 运行时实例复用缓存

**ECharts 实例复用证据**（`app/src/protyle/render/chartRender.ts` L40-L48）：
```typescript
const chartInstance = window.echarts.getInstanceById(
    renderElement.lastElementChild?.getAttribute("_echarts_instance_")
);
if (chartInstance) {
    if (chartInstance.getOption().series[0]?.type !== option.series[0]?.type) {
        chartInstance.clear();   // 系列类型变化才销毁，否则直接复用
    }
    chartInstance.resize();      // 旧实例调整尺寸
}
// 类型变化时 clear() 后重新 init
window.echarts.init(renderElement.lastElementChild, ...).setOption(option);
```

> 注：mindmapRender 虽然也使用 ECharts，但未实现此复用逻辑（L38-L40 每次都 init），存在优化空间。

---

## 六、安全边界分析

### 6.1 安全防御层全景

```
┌─────────────────────────────────────────────────────────────────┐
│                      用户输入（Markdown 文本）                    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  L1：Lute SetSanitize      │
                │  setLute.ts L20            │
                │  HTML 标签消毒             │
                └─────────────┬─────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  L2：looseJsonParse        │
                │  functions.ts L85-L87      │
                │  Function 执行非标准 JSON  │  ← 🔴 高风险点（R-01）
                └─────────────┬─────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
┌─────────▼────────┐ ┌────────▼─────────┐ ┌──────▼───────────┐
│ KaTeX trust:true  │ │ Mermaid loose    │ │ PlantUML object   │
│ mathRender L36    │ │ mermaidRender L27│ │ plantumlRender L30│
│ 允许 \href/HTML   │ │ 允许脚本/点击事件│ │ SVG 可能带脚本     │
└─────────┬────────┘ └────────┬─────────┘ └──────┬───────────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  L3：DOMPurify 消毒        │
                │  mermaidRender L102-L108   │
                │  仅 Mermaid SVG 启用        │  ← 🟡 其他渲染器缺失
                └─────────────┬─────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  L4：ft__error 降级        │
                │  异常阻断注入式攻击        │
                └───────────────────────────┘
```

### 6.2 DOMPurify XSS 防御（唯一主动消毒层）

**证据链**（`app/src/protyle/render/mermaidRender.ts` L101-L108，目前仅 Mermaid 启用）：
```typescript
let svg = mermaidData.svg.replace(/(href|src|xlink:href)\s*=\s*["']\\\\/gi, 
    (match, p1) => `${p1}="about:blank"`);  // 外链替换为 about:blank
svg = window.DOMPurify.sanitize(svg, {
    USE_PROFILES: { svg: true, svgFilters: true },
    ADD_TAGS: ["foreignObject", "use", "style"],
    ADD_ATTR: ["dominant-baseline", "xlink:href", "href"],
    HTML_INTEGRATION_POINTS: { foreignobject: true }  // foreignObject 内 HTML 不清空
});
```

**风险**：其他 7 个渲染器（math / echarts / mindmap / graphviz / flowchart / plantuml / abc）均未对输出做 DOMPurify 消毒，完全依赖引擎自身和上游 Lute sanitize。

### 6.3 KaTeX trust 模式权衡

**证据链**（`app/src/protyle/render/mathRender.ts` L32-L38）：
```typescript
const mathHTML = window.katex.renderToString(Lute.UnEscapeHTMLStr(...), {
    displayMode: isBlock,
    output: "html",
    macros,
    trust: true,                       // L36: 允许 \href, \includegraphics 等 HTML 扩展
    strict: (errorCode) => 
        errorCode === "unicodeTextInMathMode" ? "ignore" : "warn",
});
```

**攻击面**：`trust: true` 下 `\href{javascript:alert(1)}{click}` 可生成可执行脚本链接。KaTeX 内部虽对 `\href` 协议有默认校验，但需确认是否覆盖所有边缘情况。

### 6.4 Mermaid securityLevel: "loose"

**证据链**（`app/src/protyle/render/mermaidRender.ts` L26-L45）：
```typescript
const config: any = {
    securityLevel: "loose",  // L27: 注释「升级后无 #3587 问题，可使用该选项」
    flowchart: { htmlLabels: true, ... },  // L32: HTML 标签启用
    sequence: { ... },
    // ...
};
// 配合 L102 DOMPurify 下游二次消毒
```

loose 模式允许 Mermaid 使用 `<script>` 标签和 `click` 节点跳转。当前依赖下游 DOMPurify 二次消毒，但需验证 DOMPurify 配置是否过滤 `onclick` 等事件属性。

### 6.5 looseJsonParse Function 沙箱（最高风险点）

**证据链**（`app/src/util/functions.ts` L85-L87）：
```typescript
// REF https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/eval
export const looseJsonParse = (text: string) => {
    return Function(`"use strict";return (${text})`)();
};
```

该函数使用全局 `Function` 构造器，在**全局作用域**执行任意代码（虽然有 `"use strict"`，但仍可访问 `window`、`document` 等）。

**调用点清单及防护分析：**

| 调用位置 | 用途 | 是否 try/catch | 攻击面 |
|---|---|---|---|
| `chartRender.ts` L41 | 解析 ECharts option（含回调函数） | 外层 async catch | **最大**：option 完全由用户控制 |
| `mathRender.ts` L25-L29 | 解析用户自定义 KaTeX macros | ✅ 有，失败回退 `{}` | 中：macros 为 JSON 对象 |
| `abcRender.ts` L18-L23 | 解析 ABC `%%params` 行参数 | ✅ 有，失败用默认 | 小：仅 params 首行 |
| `blockRender.ts` L44-L82 | 执行 `//!js` 嵌入脚本（`new Function`） | ✅ 有，失败回退 | 最大：完全用户自定义脚本 |

### 6.6 PlantUML 远端服务 + SVG object 风险

**证据链**（`app/src/protyle/render/plantumlRender.ts` L29-L34）：
```typescript
const url = `${window.siyuan.config.editor.plantUMLServePath}${window.plantumlEncoder.encode(...)}`;
renderElement.innerHTML = `<object type="image/svg+xml" data="${url}"/>`;
renderElement.firstElementChild.addEventListener("error", () => {
    renderElement.innerHTML = `<img src=${url}">`;  // fallback 为 img
});
```

**风险点：**
1. `plantUMLServePath` 可由用户任意配置，可能指向恶意服务器
2. `<object>` 加载的 SVG 可在同源 context 中执行脚本（若服务器返回带 script 的 SVG）
3. fallback 的 `<img>` 缺少引号闭合：`src=${url}">`（注意 `>` 前引号缺失，潜在 HTML 注入）

### 6.7 Lute 上游消毒层

**证据链**（`app/src/protyle/render/setLute.ts` L20）：
```typescript
lute.SetSanitize(options.sanitize);
```

消毒选项由调用方传入，需确认发布、预览、导出、编辑四种模式下 options.sanitize 值是否一致。

---

## 七、潜在风险清单

| ID | 风险描述 | 严重程度 | 证据位置 | 修复建议 |
|---|---|---|---|---|
| **R-01** | `looseJsonParse` 使用全局 `Function` 构造器执行用户输入，ECharts option 可注入任意 JS 代码 | 🔴 高 | `app/src/util/functions.ts` L85-L87<br>`app/src/protyle/render/chartRender.ts` L41 | 改为 JSON5 解析 + 函数字段白名单 AST 提取；或移入 Web Worker 沙箱 |
| **R-02** | `blockRender` 中 `new Function` 执行 `//!js` 用户自定义嵌入脚本，暴露 fetchSyncPost/item/protyle/top | 🔴 高 | `app/src/protyle/render/blockRender.ts` L44-L82 | 文档化为高级特性；增加执行前用户确认；限制可用 API 子集 |
| **R-03** | Mermaid `securityLevel:"loose"` + `htmlLabels:true` 配合，需验证 DOMPurify 是否过滤所有事件属性 | 🟠 中 | `app/src/protyle/render/mermaidRender.ts` L27, L102-L108 | 对 DOMPurify 输出补充测试用例（onclick/onerror 等） |
| **R-04** | KaTeX `trust:true` 允许 `\href` 指向 `javascript:` 等危险协议 | 🟠 中 | `app/src/protyle/render/mathRender.ts` L36 | 配置 `trust` 的 context 校验函数，白名单 http/https/mailto |
| **R-05** | PlantUML `<img src=${url}">` 引号缺失 + SVG object 可执行脚本 + 用户可配置服务端 | 🟠 中 | `app/src/protyle/render/plantumlRender.ts` L30-L37 | 修复引号；SVG 输出追加 DOMPurify；默认服务端加入白名单校验 |
| **R-06** | mathRender/chartRender 等渲染器错误信息 `e.message` 直接 innerHTML，若引擎返回含 HTML 片段则造成注入 | 🟠 中 | `app/src/protyle/render/mathRender.ts` L123, L126<br>`app/src/protyle/render/chartRender.ts` L51 | 所有 `e.message` 先 `escapeHtml()` 再注入 |
| **R-07** | MutationObserver 无超时，折叠状态永不改变时 observer 永不 disconnect，造成内存泄漏 | 🟡 低 | `app/src/protyle/render/mermaidRender.ts` L61-L64<br>`app/src/protyle/render/flowchartRender.ts` L31-L34 | 增加 30s setTimeout 自动 disconnect |
| **R-08** | `addScript` 无 onerror 处理，网络异常时 Promise 永久 pending，渲染静默失败 | 🟡 低 | `app/src/protyle/util/addScript.ts` L21-L44 | 增加 `scriptElement.onerror` reject + 10s 超时 |
| **R-09** | `abcRender` 缺少 try/catch，abcjs 解析异常会中断后续 forEach 循环 | 🟡 低 | `app/src/protyle/render/abcRender.ts` L47 | 补充 try/catch，降级为 ft__error，与其他 7 个渲染器一致 |
| **R-10** | Mermaid icons.json 通过 `fetch` 加载，返回 JSON 未做 schema 校验 | 🟡 低 | `app/src/protyle/render/mermaidRender.ts` L22-L24 | 校验返回数据结构；追加 catch 降级 |
| **R-11** | mindmapRender 使用 ECharts 但未复用 `getInstanceById` 逻辑，每次编辑销毁重建 | 🟡 低 | `app/src/protyle/render/mindmapRender.ts` L38-L40 | 对齐 chartRender L40-L47 复用逻辑 |
| **R-12** | 除 Mermaid 外的 7 个渲染器输出均未经过 DOMPurify 二次消毒 | 🟠 中 | math/chart/mindmap/graphviz/flowchart/plantuml/abc 各渲染器 | 评估是否需要统一消毒管道；至少覆盖 HTML 注入型输出 |

---

## 八、后续检查清单（Checklist）

### 8.1 安全审查
- [ ] 为所有 `e.message` 注入点添加 `escapeHtml()` 包裹（R-06）
- [ ] 重新设计 `looseJsonParse`：拆分为 JSON5 解析 + 函数字段白名单提取（R-01）
- [ ] DOMPurify 覆盖率测试：验证 onclick/onerror/onmouseover 等事件属性是否被过滤（R-03）
- [ ] KaTeX `trust` 选项配置协议白名单上下文函数（R-04）
- [ ] PlantUML fallback img 引号修复 + SVG DOMPurify + 服务端白名单（R-05）
- [ ] 确认 Lute `SetSanitize` 在编辑 / 预览 / 导出 / 发布四种场景的配置值是否一致

### 8.2 性能与缓存
- [ ] mindmapRender 对齐 chartRender 的 ECharts 实例复用逻辑（R-11）
- [ ] MutationObserver 增加超时自动 disconnect（R-07）
- [ ] addScript / addStyle 增加 onerror + timeout 处理（R-08）
- [ ] Mermaid icons.json 结果可考虑缓存至 localStorage
- [ ] 审计 `data-render="true"` 设置时机：脚本加载失败时属性已置 true，无法自动重试

### 8.3 错误处理
- [ ] 补充 abcRender try/catch 降级（R-09）
- [ ] plantumlRender img fallback 的 error 事件链完整化
- [ ] 统一错误 UI 模板：抽象公共 `renderError(msg, container)` 函数
- [ ] Mermaid icons.json fetch 增加 catch 降级（R-10）

### 8.4 编辑协作
- [ ] 覆盖测试 inline-math 8 种 ZWSP 边界场景的光标定位
- [ ] 验证 showRender 浮层 noChange=true 时是否仍错误 removeAttribute("data-render")
- [ ] 导出 PDF maxWidth 模式下公式宽度重计算的精度测试
- [ ] 33 处 processRender 调用：检查传入容器是否可能为 null / 已 detached

### 8.5 可维护性
- [ ] 抽取 8 个渲染器的公共骨架为 `baseRender(type, engineLoader, renderer)` 函数
- [ ] 所有外部资源版本号集中至 `constants.ts` 统一管理
- [ ] RENDER_MAP 支持优先级与依赖声明，避免层级 `.then()` 嵌套
- [ ] 为 abcRender `%%params` 语法补充单元测试
- [ ] 为 33 处 processRender 调用分类建档，明确每处的触发用户行为

---

## 九、代码参考索引（仓库相对路径 + 精确行号）

| 模块 | 仓库相对路径 | 关键行号范围 |
|---|---|---|
| 渲染调度中心（定义 + 路由） | `app/src/protyle/util/processCode.ts` | L48-L73 |
| 数学公式渲染 | `app/src/protyle/render/mathRender.ts` | L1-L133 |
| ECharts 图表渲染 | `app/src/protyle/render/chartRender.ts` | L1-L56 |
| Mermaid 图渲染 | `app/src/protyle/render/mermaidRender.ts` | L1-L116 |
| 思维导图渲染 | `app/src/protyle/render/mindmapRender.ts` | L1-L87 |
| Graphviz 渲染 | `app/src/protyle/render/graphvizRender.ts` | L1-L40 |
| Flowchart 渲染 | `app/src/protyle/render/flowchartRender.ts` | L1-L73 |
| PlantUML 渲染 | `app/src/protyle/render/plantumlRender.ts` | L1-L41 |
| ABC 乐谱渲染 | `app/src/protyle/render/abcRender.ts` | L1-L50 |
| HTML 块渲染 | `app/src/protyle/render/htmlRender.ts` | L1-L16 |
| 嵌入块渲染（含 //!js） | `app/src/protyle/render/blockRender.ts` | L1-L139 |
| 异步脚本加载器 | `app/src/protyle/util/addScript.ts` | L1-L44 |
| 样式加载器 | `app/src/protyle/util/addStyle.ts` | L1-L15 |
| Lute 引擎配置 | `app/src/protyle/render/setLute.ts` | L1-L56 |
| 通用工具（looseJsonParse） | `app/src/util/functions.ts` | L85-L87 |
| 可渲染语言白名单 | `app/src/constants.ts` | L841-L843 |
| 图标 / 渲染框架生成 | `app/src/protyle/render/util.ts` | L5-L43 |
| 编辑浮层（showRender） | `app/src/protyle/toolbar/index.ts` | L1050-L1180 |
| 事务处理（10 处调用） | `app/src/protyle/wysiwyg/transaction.ts` | L115, L235, L315, L375, L429, L888, L1255, L1359, L1494 |
| 回车分裂（6 处调用） | `app/src/protyle/wysiwyg/enter.ts` | L93, L99, L483, L564, L571 |
| 粘贴（3 处调用） | `app/src/protyle/util/paste.ts` | L459, L561, L630 |
| 文档加载（2 处调用） | `app/src/protyle/util/onGet.ts`<br>`app/src/mobile/util/MobileBackFoward.ts` | L232<br>L92 |
| AI 填充 | `app/src/ai/actions.ts` | L25 |
| 预览面板 | `app/src/protyle/preview/index.ts` | L193 |
| 导出流程 | `app/src/protyle/export/util.ts` | L166 |
| AV 画廊 | `app/src/protyle/render/av/gallery/render.ts` | L157 |
| 提示面板（2 处） | `app/src/protyle/hint/index.ts`<br>`app/src/protyle/hint/extend.ts` | L865<br>L559 |
| 反链渲染（2 处） | `app/src/protyle/wysiwyg/renderBacklink.ts` | L24, L79 |

---

*文档生成时间：2026-06-15 · 基于 SiYuan 3.6.x 分支代码分析<br>数据说明：processRender 共 47 条 grep 结果，去重后 13 import + 1 定义 + **33 处真实调用**（14 个文件）*
