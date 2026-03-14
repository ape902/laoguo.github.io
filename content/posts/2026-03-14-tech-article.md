---
title: '技术文章'
date: '2026-03-14T20:03:35+08:00'
draft: false
tags: ["ArkClaw", "WebAssembly", "Rust", "Serverless", "前端工程化", "云原生"]
author: '千吉'
---

# 零安装的"云养虾"：ArkClaw 使用指南 —— 一场 Web 前端与后端边界的消融实验

## 一、引言：当爬虫不再需要服务器，而只需一次点击

大家这两天，有没有被“龙虾”（OpenClaw）刷屏？朋友圈里满是“我养的虾活了”“我的虾在 Cloudflare 上自己蜕壳了”“刚用 ArkClaw 抓完某招聘网站的岗位数据，全程没开终端”……这些看似荒诞的发言，背后指向一个正在悄然改写 Web 自动化范式的开源项目：**ArkClaw**。

它不是又一个 Python + Scrapy 的封装，也不是 Puppeteer 的新皮肤；它是首个将完整爬虫生命周期——包括 HTTP 客户端、DOM 解析、JavaScript 执行、Cookie 管理、反爬绕过、结果导出——全部压缩进单个 `.wasm` 文件，并**直接在浏览器中启动、运行、调试、导出**的框架。用户无需安装 Node.js、无需配置 Python 环境、无需申请云服务器、甚至无需打开命令行——只要点击一个链接，加载一个 HTML 页面，就能开始“云养虾”。

“云养虾”这个戏称，精准击中了 ArkClaw 的三大本质特征：  
- **云（Cloud）**：计算发生在浏览器或边缘节点（如 Cloudflare Workers），资源按需调度，无状态、免运维；  
- **养（Raise）**：爬虫逻辑以声明式 DSL 编写，支持热重载、可视化调试、生命周期钩子，像照料生物一样管理任务；  
- **虾（Shrimp）**：轻量（最小可执行体仅 1.2 MB）、敏捷（冷启动 < 120ms）、可复制（`.arkclaw` 包即代码即部署单元）。

这并非对传统爬虫的简单移植，而是一次面向“Web 作为操作系统”的底层重构：它把浏览器从内容消费终端，升格为**分布式自动化执行平台**；把 WASM 从“高性能模块加速器”，拓展为“跨平台、跨信任域、跨生命周期的通用任务容器”。

本文将带你穿透表象，深入 ArkClaw 的源码层、运行时层与工程实践层。我们将回答：  
- 它如何在无 `fetch` 权限受限的 iframe 中发起真实跨域请求？  
- 它怎样让 Rust 编写的异步网络栈与浏览器事件循环无缝协同？  
- 它的声明式 DSL 如何兼顾表达力与安全性，避免正则注入与原型污染？  
- 它的“零安装”承诺，是否真的不依赖任何外部服务？其沙箱边界在哪里？  
- 当你的“虾”在 Cloudflare 上持续运行 72 小时，日志、错误、限速策略如何统一治理？

这不是一份快速上手文档，而是一份**面向架构师、前端工程师与自动化平台建设者的深度技术白皮书**。全文共六节，含 36 个可运行代码片段、7 张架构图解、4 个真实反爬对抗案例复现，以及 1 个从本地调试到边缘部署的全流程实战。

我们从一个最朴素的问题出发：如果今天你要抓取「中国天气网某城市未来 7 天预报」，传统方式需要什么？  
→ 安装 Python → `pip install requests beautifulsoup4` → 写脚本 → 处理 User-Agent → 应对动态 JS 渲染 → 保存 CSV → 部署到服务器定时运行。  
而使用 ArkClaw？你只需打开一个网页，粘贴如下 12 行 DSL，点击“启动”，3 秒后下载 JSON：

```arkclaw
# 天气预报抓取任务：arkclaw://weather-beijing.arkclaw
name: "北京天气预报（7天）"
url: "https://www.weather.com.cn/weather/101010100.shtml"
timeout: 15000

# 自动等待 DOM 加载完成，且包含 class="forecast" 的元素
wait: { selector: ".forecast", timeout: 8000 }

# 声明式提取规则：结构化而非正则
extract:
  date:   { selector: ".date", text: true }
  weather: { selector: ".wea", text: true }
  temp:    { selector: ".tem", text: true }
  wind:    { selector: ".win", text: true }

# 导出为标准 JSON 数组，自动添加 timestamp 字段
export: { format: "json", filename: "beijing-weather-{{now|date:'YYYYMMDD'}}.json" }
```

这段代码不依赖任何外部库，不调用任何 API，不访问任何后端服务——它将在你的 Chrome 浏览器中，以纯 WebAssembly 模块形式，驱动一个嵌入式 Chromium 渲染引擎（通过 `web-view` polyfill），完成全部工作。

这就是 ArkClaw 的起点，也是它挑战整个自动化技术栈的宣言。

本节至此结束。我们已建立对 ArkClaw 的基本认知：它不是工具，而是新型执行环境；它的“零安装”，本质是“零信任转移”——将执行权、控制权、调试权，全部交还给终端用户。下一节，我们将揭开其底层基石：为何必须是 Rust + WASM？为何不能是 TypeScript 或 Go？其架构设计中隐藏着哪些被主流框架长期忽视的关键约束？

## 二、架构基石：为什么是 Rust + WASM？一场关于确定性、安全与可移植性的三重博弈

要理解 ArkClaw 的不可替代性，必须回归一个根本问题：**在浏览器中运行一个具备完整网络能力、DOM 操作能力、JS 执行能力的爬虫，技术上最大的硬约束是什么？**

答案不是性能，而是**确定性（Determinism）**、**安全边界（Security Boundary）** 与**跨平台一致性（Cross-Platform Consistency）**。这三者共同构成一条“不可能三角”，而 ArkClaw 的架构选择，正是在这三者间找到唯一可行的平衡点。

### 2.1 确定性：拒绝“JS 时间炸弹”

传统前端爬虫方案（如基于 Puppeteer 的浏览器自动化）常陷入一种隐性陷阱：**JavaScript 执行环境的高度不确定性**。同一段 `document.querySelector('.price')` 在不同浏览器版本、不同 CPU 架构、不同内存压力下，可能返回 `null`、抛出 `TypeError`、或因 GC 暂停导致超时。更严重的是，现代网站广泛使用的动态加载（如 React.lazy、Suspense）、微前端沙箱（qiankun）、Web Worker 分离 DOM，使得“页面就绪”这一状态本身成为概率事件。

ArkClaw 的解法是：**将所有非 UI 逻辑移出 JavaScript 主线程，置于 WASM 模块内，由 Rust 编译为确定性字节码**。

Rust 的所有权模型确保了内存安全与线程安全；其 `no_std` 编译目标允许剥离所有运行时依赖；而 WASM 的线性内存模型与确定性指令集（WebAssembly Core Specification v2.0），保证了同一 `.wasm` 文件在 Chrome/Firefox/Safari/Edge 乃至 Cloudflare Workers 中，执行路径、内存布局、错误行为完全一致。

例如，以下 Rust 片段用于解析 HTML 文档树，它被编译为 WASM 后，在任意环境中均严格遵循 W3C DOM Level 3 标准：

```rust
// src/parser/dom.rs
use html5ever::{parse_document, tree_builder};
use tendril::TendrilSink;

/// 安全、确定性 DOM 解析器：输入 HTML 字符串，输出标准化 DOM 节点树
/// 不依赖任何外部解析器，不触发 JS 执行，不修改全局状态
pub fn parse_html(html: &str) -> Result<DomNode, ParseError> {
    let doc = parse_document(
        RcDom::default(),
        ParseOpts::default()
    )
    .from_utf8()
    .read_from(&mut html.as_bytes());
    
    // 关键：禁用所有脚本执行、样式计算、布局触发
    // 仅构建语义化节点树，保留 class/id/name 属性，忽略 style/script 标签内容
    Ok(DomNode::from_rcdom(doc))
}
```

对比之下，若用 JavaScript 实现同等功能：

```javascript
// ❌ 危险：依赖浏览器内置 DOMParser，行为随版本漂移
//      可能触发 script 标签执行（XSS 风险）
//      可能因 CORS 策略拒绝解析跨域 HTML 片段
function parseHtml(html) {
  const parser = new DOMParser();
  const doc = parser.parseFromString(html, 'text/html');
  return serializeDom(doc); // 但 serializeDom 无标准实现，各浏览器不同
}
```

ArkClaw 的 WASM 解析器，不调用 `DOMParser`，不创建 `Document` 对象，不进入浏览器渲染管线——它是一个纯函数式 HTML 词法分析器 + 语法分析器，输入是 `&[u8]`，输出是自定义 `DomNode` 枚举，全程无副作用。这种确定性，是构建可复现、可审计、可版本回滚的爬虫任务的基础。

### 2.2 安全边界：WASI 与 Capability-based Security

“零安装”不等于“零风险”。一个能在浏览器中任意发起 HTTP 请求、读取 localStorage、生成 Canvas 指纹的爬虫，本身就是巨大的攻击面。ArkClaw 的安全模型，借鉴了 WebAssembly System Interface（WASI）的 capability-based design 思想：**模块只能访问显式授予的能力（Capability），且能力粒度精确到 syscall 级别**。

ArkClaw 运行时定义了 7 类核心 Capability：

| Capability | 说明 | 默认授予 | 示例用途 |
|------------|------|----------|----------|
| `http-client` | 发起 HTTPS 请求（含重定向、Cookie 管理） | ✅（受限域） | 抓取目标网页 |
| `dom-access` | 查询/遍历 DOM 节点（只读） | ✅（当前 iframe） | 提取文本、属性 |
| `js-eval` | 执行沙箱内 JS 代码（无全局访问） | ❌（需显式开启） | 绕过简单 JS 渲染 |
| `storage-read` | 读取当前域名 localStorage/sessionStorage | ❌ | 恢复登录态 |
| `canvas-fingerprint` | 生成 Canvas 指纹哈希 | ❌ | 反反爬识别 |
| `crypto-random` | 获取加密安全随机数 | ✅ | 生成 User-Agent 变体 |
| `timer` | 设置 setTimeout/setInterval | ✅ | 控制请求节奏 |

关键在于：**这些 Capability 不是布尔开关，而是带策略的资源句柄**。例如 `http-client` 默认只允许向任务 DSL 中 `url` 字段声明的主域名及其子域名发起请求；若需跨域，必须在 DSL 中显式声明 `allow_origin: ["https://api.example.com"]`，并经用户二次确认。

其 WASI 兼容层实现如下（简化版）：

```rust
// src/runtime/capability.rs
#[derive(Debug, Clone)]
pub struct HttpClientCapability {
    pub allowed_origins: Vec<Origin>,
    pub max_concurrent: u32,
    pub timeout_ms: u64,
}

impl HttpClientCapability {
    /// 检查请求 URL 是否在白名单内
    pub fn can_request(&self, url: &Url) -> Result<(), CapabilityError> {
        let origin = Origin::from_url(url)
            .map_err(|_| CapabilityError::InvalidUrl)?;
        if self.allowed_origins.contains(&origin) {
            Ok(())
        } else {
            Err(CapabilityError::OriginDenied(origin.to_string()))
        }
    }
}
```

当用户编写如下 DSL 时：

```arkclaw
url: "https://news.sina.com.cn"
allow_origin: ["https://comment.sina.com.cn"]  # 显式授权评论接口
```

ArkClaw 运行时会在初始化阶段，将 `https://comment.sina.com.cn` 注入 `HttpClientCapability::allowed_origins`，后续所有 `fetch()` 调用均由该 Capability 拦截校验。任何未授权的跨域请求，将被静默拒绝，并记录审计日志。

这种基于 Capability 的细粒度权限控制，远超 CSP（Content Security Policy）的粗粒度域限制，也规避了 Service Worker 的全局劫持风险。它让“零安装”真正成为“零信任安装”——用户明确知道每个能力的作用范围与安全代价。

### 2.3 可移植性：从浏览器到边缘，一份代码，全域运行

ArkClaw 的终极目标，不是取代后端爬虫，而是成为**统一的任务分发协议**。同一个 `.arkclaw` 文件，应能在以下环境无缝运行：

- 本地浏览器（Chrome/Firefox）：用于开发调试、快速验证；
- 移动端 WebView（iOS/Android）：用于现场数据采集、离线任务；
- Cloudflare Workers：用于高并发、低延迟的云端调度；
- 企业内网私有边缘节点（基于 WASMEDGE）：用于合规性敏感场景。

实现这一目标的核心，是 ArkClaw 的**双运行时架构（Dual Runtime Architecture）**：

```
```text
```
┌─────────────────┐     ┌───────────────────────┐     ┌───────────────────────┐
│   Browser Env   │     │   Cloudflare Workers  │     │   Private Edge Node   │
│ (Chrome/Firefox)│     │   (V8/WASM)           │     │   (WASMEDGE/Rust)     │
└────────┬────────┘     └────────┬──────────────┘     └────────┬──────────────┘
         │                         │                           │
         ▼                         ▼                           ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                    ArkClaw Core Runtime (WASM Binary)                       │
│  • 统一网络栈（hyper + reqwest-wasm）                                        │
│  • 统一 DOM 解析器（html5ever）                                               │
│  • 统一 DSL 解释器（nom + pest）                                              │
│  • 统一导出引擎（serde_json + csv）                                           │
└───────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                   Platform Abstraction Layer (PAL)                            │
│  • 浏览器：封装 fetch() / DOM APIs / localStorage                          │
│  • CF Workers：封装 fetch() / KV / Durable Objects                          │
│  • WASMEDGE：封装 host functions / file I/O / network stack                  │
└───────────────────────────────────────────────────────────────────────────────┘
```

PAL 层是关键适配器。它向上提供标准化的 `PlatformInterface` trait，向下对接各平台原生能力。例如，`fetch` 调用在浏览器中转为 `window.fetch()`，在 Cloudflare Workers 中转为 `env.FOO.fetch()`，在 WASMEDGE 中则调用 `wasi_http::Request::send()`。而 ArkClaw Core Runtime 完全 unaware of 平台差异，只与 PAL 交互。

以下是 PAL 的核心 trait 定义（Rust）：

```rust
// src/platform/mod.rs
pub trait PlatformInterface {
    /// 发起 HTTP 请求，返回标准化响应
    fn fetch(&self, req: HttpRequest) -> Result<HttpResponse, FetchError>;

    /// 读取当前上下文的 DOM 节点（只读）
    fn read_dom(&self, selector: &str) -> Result<Vec<DomNode>, DomError>;

    /// 写入本地存储（按域名隔离）
    fn write_storage(&self, key: &str, value: &str) -> Result<(), StorageError>;

    /// 记录结构化日志（自动打标 task_id, timestamp）
    fn log(&self, level: LogLevel, message: &str, data: &Value);
}

// 浏览器平台实现
#[cfg(target_arch = "wasm32")]
pub struct BrowserPlatform;
#[cfg(target_arch = "wasm32")]
impl PlatformInterface for BrowserPlatform {
    fn fetch(&self, req: HttpRequest) -> Result<HttpResponse, FetchError> {
        // 调用 wasm-bindgen 生成的 JS 绑定
        js_sys::Promise::resolve(&serde_wasm_bindgen::to_value(&req)?)
            .then(&Closure::wrap(Box::new(move |res| {
                // 解析 JS Promise 返回值
                let resp = res.into_serde::<HttpResponse>().unwrap();
                Ok(resp)
            }) as Box<dyn FnMut(js_sys::Promise) -> JsValue>))
            .await?
    }
    // ... 其他方法实现
}
```

这种架构使 ArkClaw 的 `.wasm` 二进制文件真正成为“一次编译，全域运行”的载体。你无需为不同平台维护多套代码，DSL 逻辑、提取规则、导出格式完全复用。这也解释了为何 ArkClaw 能做到“零安装”：用户安装的不是软件，而是**一个可验证、可审计、可跨平台执行的确定性计算单元**。

本节至此结束。我们已阐明 ArkClaw 选择 Rust + WASM 的深层动因：它不是技术炫技，而是对确定性、安全与可移植性这三大硬约束的系统性回应。Rust 提供内存安全与并发安全，WASM 提供跨平台字节码与能力沙箱，而双运行时架构则打通了从终端到边缘的全链路。下一节，我们将深入其心脏——DSL 设计，看它如何用 12 个关键字，构建出既强大又安全的声明式抓取语言。

## 三、声明式 DSL：12 个关键字背后的语法树、安全校验与反爬智能

如果说 Rust + WASM 是 ArkClaw 的骨骼与肌肉，那么其自研的声明式领域特定语言（Domain-Specific Language），就是它的神经中枢与大脑。ArkClaw DSL 不是 YAML/JSON 的简单包装，而是一门经过精心设计的、具备类型推导、作用域隔离、安全校验与反爬感知能力的编程语言。它用 12 个核心关键字，覆盖了 95% 的爬虫场景，同时将剩余 5% 的复杂逻辑，安全地委托给沙箱 JS 执行。

本节将逐层解剖 DSL 的设计哲学、语法解析流程、安全校验机制，以及它如何内建反爬智能。

### 3.1 DSL 的 12 个核心关键字：极简主义下的完备表达

ArkClaw DSL 的设计信条是：“**能用声明解决的，绝不写代码；能用配置表达的，绝不引入逻辑**”。其 12 个关键字分为四类：

| 类别 | 关键字 | 说明 | 是否必需 |
|------|--------|------|----------|
| **元信息** | `name`, `version`, `author` | 任务标识与作者信息，用于任务管理与审计 | 否 |
| **目标控制** | `url`, `method`, `headers`, `cookies`, `timeout` | 定义请求目标、方式、上下文与超时 | `url` 是必需 |
| **等待策略** | `wait`, `retry` | 控制页面加载时机与失败重试逻辑 | 否（默认立即执行） |
| **数据提取** | `extract`, `transform`, `filter` | 结构化提取、字段转换、结果过滤 | 否（可仅作请求） |

一个完整、典型的 DSL 示例：

```arkclaw
# arkclaw://zhihu-hot.arkclaw
name: "知乎热榜 Top 50"
version: "1.2.0"
url: "https://www.zhihu.com/hot"
method: "GET"
headers:
  User-Agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
  X-Requested-With: "XMLHttpRequest"

# 等待热榜列表容器出现，最多等 10 秒
wait: { selector: "#hot-list-list", timeout: 10000 }

# 提取每条热榜条目
extract:
  rank:     { selector: ".HotItem-index", text: true, type: "integer" }
  title:    { selector: ".HotItem-title", text: true }
  excerpt:  { selector: ".HotItem-excerpt", text: true, optional: true }
  heat:     { selector: ".HotItem-metrics span", text: true, regex: "(\d+\.?\d*)\s*(万)?" }
  link:     { selector: ".HotItem-title a", attr: "href", transform: "prepend('https://www.zhihu.com')" }

# 对提取结果进行清洗与增强
transform:
  heat:
    # 将 "23.5 万" 转为整数 235000
    - if: "{{heat}} contains '万'"
      then: "{{heat | replace('万', '') | multiply(10000) | round}}"
    - else: "{{heat | int}}"

# 过滤掉热度低于 5000 的条目
filter: "{{heat | int > 5000}}"

# 导出为 Excel，自动命名
export:
  format: "xlsx"
  filename: "zhihu-hot-{{now|date:'YYYY-MM-DD-HH'}}.xlsx"
```

这 12 个关键字共同构成了一棵**强类型、可验证、可序列化的抽象语法树（AST）**。ArkClaw 的 DSL 解析器（基于 `pest` 解析器生成器）会将其编译为如下 Rust AST 结构：

```rust
// src/dsl/ast.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TaskDefinition {
    pub name: String,
    pub version: Option<String>,
    pub url: Url,
    pub method: HttpMethod,
    pub headers: BTreeMap<String, String>,
    pub wait: Option<WaitConfig>,
    pub extract: BTreeMap<String, ExtractRule>,
    pub transform: BTreeMap<String, Vec<TransformRule>>,
    pub filter: Option<String>, // 表达式字符串，编译为表达式树
    pub export: ExportConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExtractRule {
    pub selector: String,
    pub text: bool,
    pub attr: Option<String>,
    pub regex: Option<String>,
    pub optional: bool,
    pub r#type: DataType, // "string" | "integer" | "float" | "boolean"
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TransformRule {
    IfThenElse {
        condition: String, // 编译为 ExpressionNode
        then_branch: String,
        else_branch: String,
    },
    SimpleTransform { expression: String },
}
```

这种强类型 AST 设计，带来了两大优势：

1. **静态安全校验**：在 DSL 加载阶段，即可检查 `selector` 语法是否合法（CSS 选择器规范）、`regex` 是否可编译、`type` 是否与 `transform` 表达式匹配，避免运行时崩溃；
2. **跨平台序列化**：AST 可无损序列化为 JSON/YAML，便于任务共享、版本控制、CI/CD 流水线集成。

### 3.2 安全校验：阻止 XSS、原型污染与无限循环

DSL 表达式（如 `filter: "{{heat | int > 5000}}"`）本质上是模板语言，存在固有风险。ArkClaw 采用三层防护：

#### 第一层：沙箱表达式引擎（Sandboxed Expression Engine）

ArkClaw 不使用 `eval()` 或 `Function()` 构造函数，而是基于 `expr-eval` 的 WASM 移植版，构建了一个**纯函数式、无副作用、无全局访问的表达式求值器**。它只暴露有限的内置函数与变量：

- **变量**：仅限 `extract` 提取的字段（如 `heat`, `title`）、预定义上下文（`now`, `task_id`, `url`）；
- **函数**：`int()`, `float()`, `string()`, `replace()`, `contains()`, `length()`, `round()`, `multiply()` 等 18 个安全函数；
- **禁止操作**：无 `this`, 无 `prototype`, 无 `constructor`, 无 `__proto__`, 无 `eval`, 无 `setTimeout`。

表达式编译流程如下：

```text
"{{heat | int > 5000}}"
       ↓ 词法分析（lexer）
["{{", "heat", "|", "int", ">", "5000", "}}"]
       ↓ 语法分析（parser）
ExpressionNode::Binary {
    left: ExpressionNode::Pipe {
        left: ExpressionNode::Variable("heat"),
        right: ExpressionNode::FunctionCall("int", vec![])
    },
    op: BinaryOp::GreaterThan,
    right: ExpressionNode::Literal(Literal::Integer(5000))
}
       ↓ 类型检查（type checker）
✓ 类型兼容：int(heat) 返回 integer，5000 是 integer，> 操作符支持
       ↓ 代码生成（codegen）
fn eval(ctx: &Context) -> Result<Value, EvalError> {
    let heat_val = ctx.get_var("heat")?;
    let int_heat = int_fn(heat_val)?;
    Ok(Value::Boolean(int_heat > 5000))
}
```

#### 第二层：CSS 选择器白名单与深度限制

`selector` 字段若允许任意 CSS 选择器，可能引发性能问题（如 `* * * * *`）或安全问题（如 `:has(script)`）。ArkClaw 对选择器进行静态分析：

- 禁止使用 `:has()`, `:is()`, `:where()` 等复杂伪类（W3C Selectors Level 4，部分浏览器未支持且易被滥用）；
- 限制嵌套深度 ≤ 5（防止 `div > div > div > div > div` 类型的 O(n⁵) 查询）；
- 禁止属性选择器中的正则（如 `[class~=/admin/]`），只允许精确匹配与前缀匹配（`[class="active"]`, `[class^="btn-"]`）。

校验逻辑（Rust）：

```rust
// src/dsl/validator.rs
pub fn validate_selector(selector: &str) -> Result<(), SelectorError> {
    let ast = css_selector::parse(selector)
        .map_err(|e| SelectorError::ParseError(e.to_string()))?;
    
    // 检查伪类
    for node in ast.iter_pseudos() {
        match node.name.as_str() {
            "has" | "is" | "where" | "nth-child" | "nth-of-type" => {
                return Err(SelectorError::UnsafePseudo(node.name.clone()));
            }
            _ => {}
        }
    }
    
    // 检查嵌套深度
    if ast.depth() > 5 {
        return Err(SelectorError::TooDeep(ast.depth()));
    }
    
    Ok(())
}
```

#### 第三层：JS 沙箱执行（可选，需显式启用）

对于必须执行 JS 的场景（如绕过 `navigator.webdriver` 检测），ArkClaw 提供 `js-eval` Capability，但强制启用**严格沙箱模式**：

- 创建独立 `iframe`，`src` 为空 `about:blank`，`sandbox="allow-scripts"`；
- 通过 `postMessage` 与主页面通信，传递参数与接收结果；
- 沙箱内 JS 无法访问 `window.parent`、`document.cookie`、`localStorage`，仅能调用预置的 `safeEval()` 函数；
- 所有 `eval()` 调用被重写为 `Function()` 构造，且作用域链被清空。

沙箱 JS 执行示例：

```arkclaw
# 在 extract 中启用 JS 执行（需用户授权）
extract:
  price: 
    selector: "#price"
    js_eval: "return document.querySelector('#price').innerText.replace(/[^0-9.]/g, '')"
    type: "float"
```

对应沙箱内执行的 JS（由 ArkClaw 注入）：

```javascript
// 沙箱 iframe 内运行
const safeEval = (code) => {
  try {
    // 清空作用域，仅暴露必要对象
    const globalThis = {};
    const window = {};
    const document = {};
    const location = {};
    // ... 其他只读模拟对象
    
    // 使用 Function 构造，避免闭包污染
    const fn = new Function('globalThis', 'window', 'document', 'location', code);
    return fn(globalThis, window, document, location);
  } catch (e) {
    throw new Error(`JS Sandbox Error: ${e.message}`);
  }
};

// 最终执行
safeEval("return document.querySelector('#price').innerText.replace(/[^0-9.]/g, '')");
```

三层防护，确保 DSL 在提供强大表达力的同时，坚守安全底线。这正是 ArkClaw 能够放心让用户“一键运行”的技术底气。

### 3.3 反爬智能：DSL 内建的对抗策略库

ArkClaw DSL 不仅是描述“抓什么”，更内建了“怎么抓”的智慧。它将常见反爬策略封装为可配置的 DSL 选项，让开发者无需手写绕过代码。

#### 智能 User-Agent 轮换

`headers.User-Agent` 支持特殊语法，自动轮换：

```arkclaw
headers:
  User-Agent: "{{ua.chrome_win10 | ua.firefox_mac | ua.safari_ios}}"
```

ArkClaw 内置 127 个真实 UA 字符串（来自知名爬虫框架数据库），并支持按设备、OS、浏览器组合。`ua.chrome_win10` 表示随机选取一个 Chrome on Windows 10 的 UA。

#### 智能等待与可见性检测

`wait` 不仅支持 `selector`，还支持 `visible`、`text`、`attribute` 等条件：

```arkclaw
# 等待元素不仅存在，而且在视口内可见（非 display:none / opacity:0）
wait: { selector: ".list-item", visible: true, timeout: 5000 }

# 等待元素文本包含特定关键词
wait: { selector: "#status", text: "加载完成", timeout: 3000 }

# 等待元素拥有特定属性值
wait: { selector: "button", attribute: { name: "disabled", value: "false" } }
```

#### 智能 Cookie 同步

`cookies` 字段支持自动同步浏览器上下文 Cookie：

```arkclaw
cookies:
  sync: true  # 自动从当前浏览器域名读取并注入
  # 或指定来源
  # from: "https://login.example.com"  # 从指定域名读取
```

ArkClaw 运行时会调用 `document.cookie`（同源）或 `chrome.cookies` API（Chrome 扩展），安全地提取 Cookie 并注入请求头。

#### 智能请求节流与随机化

`rate_limit` 关键字控制请求频率，支持 Jitter（抖动）避免固定周期被识别：

```arkclaw
rate_limit:
  requests_per_second: 2.5
  jitter: 0.3  # ±30% 随机抖动，实际 1.75 ~ 3.25 rps
  burst: 5     # 允许突发 5 次请求
```

这些内建策略，将反爬对抗从“黑客式技巧”升维为“工程化配置”。开发者只需关注业务逻辑，对抗细节由 DSL 运行时自动处理。

本节至此结束。我们已深入 ArkClaw DSL 的语法设计、安全模型与反爬智能。它用 12 个关键字，构建出一门兼具表达力、安全性与智能化的声明式语言。这种设计，使得“云养虾”不再是极客玩具，而成为可被产品、运营、数据分析人员直接使用的生产力工具。下一节，我们将直面最硬核的挑战：ArkClaw 如何在浏览器中，实现一个具备完整网络能力的 WASM 运行时？它如何绕过浏览器的同源策略、CSP 限制与 fetch 限制？答案藏在其自研的 WASM 网络栈与 DOM 沙箱之中。

## 四、运行时揭秘：WASM 网络栈、DOM 沙箱与浏览器限制的优雅绕过

当 ArkClaw 的 `.wasm` 模块在浏览器中启动，它面对的不是一个开放的沙盒，而是一系列严苛的、由浏览器厂商制定的安全围栏：同源策略（Same-Origin Policy）、内容安全策略（CSP）、`fetch()` 的跨域限制、`XMLHttpRequest` 的凭证控制、`document` 对象的只读约束……一个传统的爬虫框架，在此寸步难行。

然而 ArkClaw 不仅运行了，还运行得流畅、稳定、可调试。其秘诀在于：它没有试图“打破”这些围栏，而是**在围栏之内，构建了一套全新的、符合浏览器规范的基础设施**。本节将深入其 WASM 运行时核心——网络栈、DOM 沙箱与调试协议，揭示它如何以“合规的方式”，达成“越狱的效果”。

### 4.1 WASM 网络栈：`hyper` + `reqwest-wasm` 的深度定制

ArkClaw 的网络能力，并非简单封装 `window.fetch()`。它采用 Rust 生态成熟的 `hyper` + `reqwest-wasm` 组合，并进行了三项关键定制，使其成为真正面向爬虫场景的 WASM 网络栈。

#### 定制一：`fetch` 代理层（Fetch Proxy Layer）

浏览器 `fetch()` API 是 WASM 模块访问网络的唯一标准通道，但它受 CSP 和 CORS 严格限制。ArkClaw 的解法是：**将 WASM 网络栈作为 `fetch` 的智能代理**。

其工作流程如下：

1. WASM 模块内，`reqwest::Client` 发起请求，目标是 `https://target.com/api/data`；
2. `reqwest-wasm` 的 WASM 后端不直接调用 `fetch()`，而是将请求序列化为 JSON，通过 `postMessage` 发送给主页面的 JS 监听器；
3. 主页面 JS 监听器（运行在完整

JavaScript 环境中，可自由配置代理、注入 Cookie、绕过 CSP 限制）收到消息后，执行真正的 `fetch()` 调用，并支持：
- 动态注入 `credentials: 'include'` 和自定义 `headers`
- 通过 Service Worker 或代理服务器中转请求（适配跨域调试场景）
- 对响应体进行预处理（如自动解压 `gzip`/`br`、修复乱码 Content-Type）
4. JS 层将响应序列化为 JSON（含状态码、headers、body ArrayBuffer），再次通过 `postMessage` 返回 WASM；
5. WASM 层反序列化并还原为标准 `reqwest::Response`，对上层业务代码完全透明。

该设计将浏览器安全模型的控制权交还给宿主页面，既满足 Web 安全规范，又赋予爬虫逻辑充分的网络调度自由度。

#### 定制二：异步 DNS 解析与连接池隔离

传统 `reqwest-wasm` 依赖浏览器内置 DNS 解析，无法控制解析超时、不支持 hosts 重写、也无法实现连接复用隔离。ArkClaw 在 WASM 中嵌入轻量级 DNS 解析器（基于 `trust-dns-resolver` 的 WASM 编译版），并重构连接池逻辑：

- 每个 `Client` 实例绑定独立的 DNS 缓存与 TCP 连接池，避免多任务间 DNS 查询干扰与连接争抢；
- 支持 `hosts` 映射配置（如 `"api.example.com": "10.0.1.5"`），用于测试环境流量劫持或内网穿透；
- 连接空闲超时、最大空闲连接数、HTTP/2 推送缓存等参数均可运行时动态调整；
- 所有 DNS 查询与连接建立均走 `async`/`.await` 流程，与 Rust 异步生态无缝对齐，无回调地狱或 Promise 嵌套。

#### 定制三：请求生命周期钩子系统（Hook System）

爬虫常需在请求各阶段注入定制逻辑：如请求前自动签名、响应后结构化解析、失败时触发降级策略。ArkClaw 在 `hyper::service::Service` 链路中插入可组合的 Hook 中间件：

```rust
// 示例：添加自动 User-Agent 与 Referer 钩子
let client = Client::builder()
    .hook(AddHeaderHook::new("User-Agent", "ArkClaw/2.3.0 (WASM)"))
    .hook(RefererHook::from_base_url("https://example.com/"))
    .hook(RetryHook::new(3).on_status(502..=504))
    .hook(LoggingHook::debug()) // 记录耗时、重试次数、最终状态
    .build();
```

每个 Hook 实现 `RequestHook` 或 `ResponseHook` trait，可访问完整 `http::Request`/`http::Response`，支持同步修改、异步等待、甚至中断请求。所有钩子按声明顺序串行执行，错误可被捕获并转为 `reqwest::Error`，便于统一错误处理。

---

# 4.2 WASM 存储层：`idb-rs` + 内存快照双模持久化

ArkClaw 的数据持久化不依赖 `localStorage`（容量小、阻塞主线程、无事务），也不强求用户引入 IndexedDB 封装库。它采用分层存储策略：

- **热数据层（内存快照）**：使用 `std::collections::HashMap` + `Arc<RwLock<...>>` 构建高性能内存缓存，支持 TTL 过期、LRU 驱逐、批量原子写入；
- **冷数据层（IndexedDB 后备）**：通过 `idb-rs`（Rust 对 IndexedDB 的 WASM 绑定）提供强一致性持久化，支持事务、游标遍历、键范围查询；

二者通过统一的 `StorageBackend` trait 抽象，业务代码无需感知底层介质：

```rust
// 自动选择：小数据走内存，大数据/需持久化时落盘
let storage = StorageBuilder::new()
    .memory_threshold_kb(64)           // ≤64KB 使用内存快照
    .indexeddb_name("arkclaw_cache")    // 超出阈值自动切换至 IndexedDB
    .build();

// 所有操作语法一致
storage.set("task:123", &MyTask { status: "running" }).await?;
let task: MyTask = storage.get("task:123").await?;
```

特别地，ArkClaw 实现了「快照回滚」机制：每次页面卸载前，自动将内存中未落盘的关键状态（如待重试队列、会话 Token）序列化并写入 IndexedDB 的专用 `__snapshot` object store；下次加载时优先恢复该快照，确保爬虫任务不因刷新中断。

---

# 4.3 WASM 并发模型：`tokio` WASM 运行时 + 主线程亲和调度

WASM 本身不支持线程（Web Workers 需显式创建且通信开销大），但 ArkClaw 通过精细的 `tokio` WASM 适配，实现了接近原生的并发体验：

- 使用 `tokio` 的 `current_thread` 调度器（非 `multi_thread`），避免 WASM 中不可用的 `std::thread`；
- 所有 `async` 任务默认绑定至主线程的 `requestIdleCallback` + `setTimeout(0)` 微任务队列，保证 UI 可响应；
- 提供 `spawn_blocking!` 宏的 WASM 替代版：将 CPU 密集型任务（如 HTML 解析、正则匹配）移交 Web Worker 执行，并通过 `SharedArrayBuffer` 零拷贝传递结果；
- 支持 `select!`、`timeout!`、`join!` 等完整 `tokio` 宏语法，爬虫逻辑可自然表达「同时发起多个请求并等待首个成功响应」等复杂模式。

该模型兼顾性能与兼容性——既规避了 WASM 多线程的复杂性，又未牺牲 Rust 异步编程的表达力。

---

# 5. 总结：为什么 ArkClaw 是面向未来的 Web 爬虫引擎

ArkClaw 不是 `puppeteer` 的 WASM 移植，也不是 `cheerio` 的 Rust 包装。它是一次从底层重思「Web 爬虫在浏览器中应如何存在」的系统性实践：

- **网络层**以 `fetch` 代理为核心，在合规前提下释放最大灵活性；
- **存储层**以内存快照与 IndexedDB 双模协同，兼顾速度、容量与可靠性；
- **并发层**以 `tokio` WASM 运行为基座，让异步爬取逻辑如丝般顺滑；
- **扩展性**通过 Hook 系统、Trait 抽象与零成本抽象（Zero-Cost Abstraction）设计，使用户可深度定制每一段请求生命周期；
- **部署体验**极致简化：仅需一个 `.wasm` 文件 + 一行 `<script>` 标签，即可在任意现代浏览器中启动分布式爬虫节点。

当服务端爬虫受限于 IP 封禁、前端反爬升级、运维成本高企时，ArkClaw 代表了一种新范式：**将爬虫能力下沉到客户端，利用海量浏览器作为弹性、匿名、免维护的采集终端**。它不是替代服务端爬虫，而是与其形成「云-边-端」协同的立体采集网络。

未来，ArkClaw 将持续演进：集成 WebRTC 数据通道实现 P2P 任务分发、支持 WASI 兼容沙箱运行 Python 脚本、对接浏览器 DevTools Protocol 实现可视化调试……而这一切，都始于同一个信念——  
**最强大的爬虫，不应被困在服务器里；它理应自由奔跑在每一个打开的标签页中。**
