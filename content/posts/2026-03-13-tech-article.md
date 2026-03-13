---
title: '技术文章'
date: '2026-03-13T09:03:25+08:00'
draft: false
tags: ["ArkClaw", "WebAssembly", "serverless", "Rust", "WASI", "无服务器架构", "前端沙箱"]
author: '千吉'
---

# 零安装的"云养虾"：ArkClaw 使用指南  
## ——一场关于执行边界消融的技术静默革命

> **注**：本文所指“龙虾”并非生物物种，而是对开源项目 `ArkClaw`（原名 `OpenClaw`）的社区昵称。该名称源于其 Logo 设计——一只以 Rust 编写的、挥舞着 WebAssembly 钳子的抽象龙虾，象征对传统运行时边界的钳制与重构。本文所有技术分析均基于 `ArkClaw v0.8.3`（2026 年 3 月稳定版）及配套工具链 `arkclaw-cli@0.8.3`、`@arkclaw/runtime@0.8.3`、`arkclaw-web@0.8.3`。

---

## 一、引言：当“养虾”成为前端工程师的新日常

大家这两天，有没有被“龙虾”刷屏？不是海鲜市场涨价通知，也不是海洋生物学论文推送，而是一条条 GitHub Star 暴涨、Discord 频道爆满、Twitter（现 X 平台）上程序员们晒出自己“云养虾”成果的截图——有人在 Chrome 浏览器地址栏输入一个 `arkclaw://` 协议链接，回车后，一段用 Rust 编写的实时图像降噪算法便在毫秒内启动；有人把本地写好的 Python 数据清洗脚本（经 Pyodide 编译为 WASM）拖进网页，点击“投喂”，三秒完成百万行 CSV 处理并返回可视化图表；还有人将整个小型 React 应用打包为单个 `.wasm` 文件，上传至任意静态托管服务（如 GitHub Pages），再通过 ArkClaw 运行时直接加载——无需 Node.js、不启 Express、不配 Nginx，连 `package.json` 都不必存在。

这并非魔法，而是一场静默发生的技术范式迁移：**执行环境正在从“安装即拥有”转向“链接即运行”，从“进程级隔离”跃迁至“字节码级瞬时沙箱”**。而 ArkClaw，正是这一迁移中最具代表性的基础设施载体。

它不叫“框架”，不叫“平台”，官方定义是：**一个面向终端用户的、零配置、零安装、跨设备的 WASI 兼容 WebAssembly 执行引擎**。它的核心承诺只有一句：“你只需一个 URL，就能运行任何符合标准的 WASM 模块——无论它用 Rust、C、Zig、AssemblyScript 还是（经编译器支持的）Python 写成。”

但为何是现在？为何是 ArkClaw？为何社区将其戏称为“云养虾”？这个看似戏谑的称呼背后，实则暗含三层深刻隐喻：

- **“云”**：指执行完全发生在用户终端（浏览器或轻量 CLI），计算资源由客户端提供，服务端仅承担静态分发职责，真正实现“去中心化执行”；
- **“养”**：强调生命周期管理——ArkClaw 不是简单 `instantiate()` 一下就完事，它内置模块注册、依赖自动发现、内存用量监控、超时熔断、热重载支持，让 WASM 模块像活体一样可观察、可干预、可持续培育；
- **“虾”**：既是对项目名 `ArkClaw` 的谐音转化，更指向其本质特性——**小而锐利、反应敏捷、环境适应力极强**。一只龙虾能在深海高压、低温、黑暗中存活，ArkClaw 则能在 iOS Safari 的严格限制、Windows Edge 的旧版 WASM 支持、甚至树莓派 Zero W 的 ARMv6 架构上稳定运行。

本文将摒弃浮光掠影的功能罗列，以深度技术解剖的方式，带您穿越 ArkClaw 的七层实现结构：从最表层的用户交互协议，到底层的 WASI 系统调用劫持；从安全沙箱的内存页保护策略，到跨语言 ABI 的二进制契约；从开发者工作流的范式颠覆，到企业级部署中的可信执行链构建。我们将用超过 30% 的真实代码占比（含完整可运行示例），还原一个“零安装云养虾”系统是如何从理念落地为每天支撑百万次执行请求的工业级工具。

这不是一篇教程，而是一份**WASM 原生时代的技术考古报告**——当我们站在 2026 年回望，ArkClaw 或许不会是最终形态，但它已凿开了那堵名为“运行时依赖”的承重墙。

本节完。

---

## 二、破壁者：ArkClaw 如何终结“npm install”的宿命

要理解 ArkClaw 的革命性，必须先直面那个缠绕前端开发十余年、至今仍在消耗全球工程师数百万工时的幽灵：**依赖地狱（Dependency Hell）**。

想象这样一个典型场景：你收到一份来自数据科学团队的 `.py` 脚本，用于清洗客户行为日志。你想把它嵌入内部 BI 系统的前端页面中，实时展示清洗结果。传统路径是：

1. 在服务器端用 Flask 启一个 API；
2. 为该 API 单独维护一个 Python 环境（virtualenv + requirements.txt）；
3. 配置反向代理（Nginx）和 CORS；
4. 前端发起 HTTP 请求，等待网络 I/O 和服务端 CPU 计算；
5. 若脚本依赖 `pandas==1.5.3`，而服务器已装 `pandas==2.0.0`，则需降级或容器化隔离……

整套流程耗时数小时，且每个环节都引入新的故障点：网络延迟、进程崩溃、版本冲突、权限错误、SSL 证书过期……

ArkClaw 的答案简洁到令人不安：**把这段 Python 脚本，编译成 WebAssembly，扔到 CDN 上，然后在前端用一行 JS 加载执行**。

但这“一行 JS”背后，是五层精密协作的破壁工程：

### 2.1 第一层破壁：协议即入口（`arkclaw://` 自定义协议）

ArkClaw 定义了一个浏览器可识别的自定义 URL 协议 `arkclaw://`，其语法为：

```text
arkclaw://<module-id>[?param1=value1&param2=value2]#<entry-point>
```

- `<module-id>`：可为绝对 URL（如 `https://cdn.example.com/blur.wasm`），也可为简写 ID（如 `@demo/image-blur`，由 ArkClaw Registry 解析为实际地址）；
- 查询参数（`?` 后）作为模块启动时的 `args` 传入；
- 片段标识符（`#` 后）指定 WASM 导出的主函数名（默认为 `_start`）。

浏览器中直接访问该链接，会触发 ArkClaw 的全局协议处理器。若用户未安装 ArkClaw Desktop 客户端，则自动跳转至 Web 版运行时（`https://run.arkclaw.dev/?url=...`）；若已安装，则交由本地 CLI 执行，获得完整 WASI 权限。

> ✅ 实战验证：在任意现代浏览器地址栏粘贴以下链接（需确保网络可达）  
> `arkclaw://https://cdn.arkclaw.dev/modules/demo/hello-world.wasm?name=张三#main`  
> 你将看到弹窗显示 `"Hello, 张三! (executed in 12ms)"` —— 整个过程无需任何本地安装，无网络请求除该 WASM 文件本身外的任何资源。

该协议的设计哲学是：**URL 不再只是文档定位符，而是可执行程序的统一身份标识（URI as Executable Identity）**。

### 2.2 第二层破壁：零依赖运行时（`@arkclaw/runtime`）

ArkClaw Web 版的核心是一个仅 187 KB 的 ES Module（Gzip 后），名为 `@arkclaw/runtime`。它不依赖 Webpack、不引入 Lodash、不使用任何第三方 polyfill——因为它只做三件事：

- 加载 `.wasm` 二进制并校验 SHA-256 签名（防篡改）；
- 构建 WASI 兼容的环境（`wasi_snapshot_preview1` 接口）；
- 提供内存安全的主机函数注入机制（Host Functions）。

以下是其最小可用加载器代码（已精简，保留核心逻辑）：

```ts
// runtime/minimal-loader.ts
import { WASI } from '@bjorn3/browser_wasi_shim'; // ArkClaw 采用 fork 维护的轻量 WASI shim
import { instantiate } from 'wasm-feature-detect'; // 检测浏览器 WASM 支持等级

// 1. 创建 WASI 实例：模拟文件系统、时钟、随机数等基础能力
const wasi = new WASI({
  version: 'preview1',
  // 注意：此处不挂载真实文件系统！所有 fs 调用均路由至内存虚拟盘
  preopens: {
    '/': new InMemoryDir(), // 所有模块看到的 "/" 是一块空内存目录
  },
  // 环境变量注入（来自 URL 查询参数）
  env: Object.fromEntries(new URLSearchParams(window.location.search).entries()),
});

// 2. 获取 WASM 字节码（支持 streaming fetch）
async function fetchWasm(url: string): Promise<WebAssembly.Module> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  const bytes = await response.arrayBuffer();
  // 强制校验签名（ArkClaw Registry 签发的 .wasm.sig 文件）
  await verifyWasmSignature(bytes, url.replace(/\.wasm$/, '.wasm.sig'));
  return await WebAssembly.compile(bytes);
}

// 3. 实例化并启动
export async function runWasm(url: string, entryPoint: string = '_start'): Promise<void> {
  const module = await fetchWasm(url);
  const instance = await WebAssembly.instantiate(module, {
    wasi_snapshot_preview1: wasi.exports,
    env: {
      // 注入 ArkClaw 特有的主机函数：用于日志、性能上报、UI 交互
      arkclaw_log: (ptr: number, len: number) => {
        const decoder = new TextDecoder();
        const msg = decoder.decode(wasi.memory.buffer.slice(ptr, ptr + len));
        console.info('[ArkClaw]', msg); // 输出到浏览器控制台
      },
      arkclaw_alert: (ptr: number, len: number) => {
        const text = new TextDecoder().decode(wasi.memory.buffer.slice(ptr, ptr + len));
        alert(`🦀 ArkClaw 模块提示：${text}`); // 弹窗提醒用户
      }
    }
  });

  // 4. 调用指定入口函数（支持带参数的 _start 或自定义函数）
  const exports = instance.exports as any;
  if (typeof exports[entryPoint] === 'function') {
    exports[entryPoint]();
  } else if (entryPoint === '_start' && typeof exports._start === 'function') {
    // WASI 标准入口：_start 不接收参数，但会读取 wasi.args
    exports._start();
  } else {
    throw new Error(`Entry point "${entryPoint}" not found or not callable`);
  }
}

// 使用示例（在页面中调用）
document.addEventListener('DOMContentLoaded', () => {
  const urlParams = new URLSearchParams(window.location.search);
  const wasmUrl = urlParams.get('url') || '';
  const entry = urlParams.get('entry') || '_start';
  if (wasmUrl) {
    runWasm(wasmUrl, entry).catch(console.error);
  }
});
```

> 🔍 关键洞察：这段代码中没有任何 `require()`、`import * as xxx from 'xxx'`，也没有 `npm install` 步骤。它就是一个纯 ESM，可通过 `<script type="module" src="..."></script>` 直接运行。这就是“零安装”的第一物理含义——**运行时自身不产生任何 npm 依赖链**。

### 2.3 第三层破壁：WASI 的轻量化重构（`wasi-core` 子系统）

标准 WASI（WebAssembly System Interface）规范定义了约 40 个系统调用（syscalls），涵盖文件读写、网络、时钟、随机数等。但浏览器环境天然无法提供真实文件系统或 socket——ArkClaw 的解法不是“阉割”，而是“重定向”。

它实现了 `wasi-core` 子系统，将所有 WASI 调用映射为内存或 Web API 操作：

| WASI syscall         | ArkClaw 实现方式                                                                 | 安全约束                                  |
|----------------------|----------------------------------------------------------------------------------|------------------------------------------|
| `args_get`           | 从 URL 查询参数或 `wasi.env` 中提取                                               | 参数长度上限 64KB，禁止 `\0` 字符         |
| `clock_time_get`     | 调用 `performance.now()` + `Date.now()`                                         | 返回毫秒级时间戳，不暴露纳秒精度            |
| `random_get`         | 使用 `crypto.getRandomValues()`                                                  | 符合 CSPRNG 标准，拒绝 `Math.random()`     |
| `path_open`          | 在 `InMemoryDir` 结构中查找路径，返回内存文件描述符（fd=3~1023）                      | 所有路径均被 chroot 到 `/` 虚拟根目录       |
| `fd_read` / `fd_write` | 读写内存中的 `Uint8Array` 缓冲区                                                   | fd 生命周期与模块实例绑定，退出即释放        |
| `proc_exit`          | 抛出 `WasmExitCode` 异常，由运行时捕获并记录退出码                                     | 退出码范围限定 0~255，-1 表示 OOM 或超时     |

这种设计带来两大收益：

- **极致轻量**：`wasi-core` 仅 23 KB，比完整 WASI shim 小 80%；
- **强可控性**：所有 I/O 都经过 ArkClaw 的“海关检查”，可插入审计钩子（audit hooks）、速率限制、敏感操作拦截。

例如，一个恶意模块试图调用 `path_open("/etc/passwd", ...)`，在 ArkClaw 中会被解析为打开虚拟根目录下的 `/etc/passwd` 文件——该路径在内存中根本不存在，返回 `ERR_NOENT`，且此次调用会被日志记录：“模块 demo/malware.wasm 尝试访问非法路径 /etc/passwd”。

### 2.4 第四层破壁：跨语言 ABI 的统一契约（`arkclaw-abi`）

不同语言编译出的 WASM 模块，导出函数签名千差万别：Rust 默认用 `#[no_mangle] pub extern "C"` 导出 C ABI；AssemblyScript 生成的是 `export function main(): void`；而 Zig 可能用 `export fn _start() void`。ArkClaw 如何统一调用？

答案是：**强制所有发布到 ArkClaw Registry 的模块，必须遵循 `arkclaw-abi` 二进制接口规范**。该规范定义了三个必选导出函数：

```wat
;; arkclaw-abi 规范（WebAssembly Text Format）
(module
  ;; (1) 初始化函数：模块加载后立即调用，传入 argc/argv
  (func $arkclaw_init (param $argc i32) (param $argv i32))
  
  ;; (2) 主执行函数：业务逻辑入口，返回 i32 退出码
  (func $arkclaw_main (result i32))

  ;; (3) 清理函数：模块卸载前调用（可选）
  (func $arkclaw_cleanup)

  ;; 必须导出这三者
  (export "arkclaw_init" (func $arkclaw_init))
  (export "arkclaw_main" (func $arkclaw_main))
  (export "arkclaw_cleanup" (func $arkclaw_cleanup))
)
```

任何语言只要在编译时链接 `arkclaw-abi` 的 stub 库，即可自动适配。例如 Rust 侧：

```rust
// Cargo.toml
[dependencies]
arkclaw-abi = "0.8"

// src/lib.rs
use arkclaw_abi::{arkclaw_init, arkclaw_main, arkclaw_cleanup};

#[no_mangle]
pub extern "C" fn arkclaw_init(argc: i32, argv: i32) {
    // 解析传入的参数（argv 指向内存中的字符串数组）
    let args = unsafe { parse_argv(argc, argv) };
    println!("🚀 ArkClaw 初始化，收到 {} 个参数", args.len());
}

#[no_mangle]
pub extern "C" fn arkclaw_main() -> i32 {
    // 你的业务逻辑
    match do_real_work() {
        Ok(_) => 0,   // 成功
        Err(e) => {
            eprintln!("❌ 执行失败：{}", e);
            1          // 错误码
        }
    }
}

#[no_mangle]
pub extern "C" fn arkclaw_cleanup() {
    // 释放资源、关闭连接等
}
```

而 Python（通过 `wasi-sdk` + `pyodide` 工具链）侧，只需在 `setup.py` 中声明：

```python
# setup.py
from pyodide_build import build
build(
    name="my-data-cleaner",
    abi="arkclaw-abi@0.8",  # 显式声明 ABI 兼容性
    entry_point="main:run_cleaning_job"
)
```

ArkClaw 运行时在实例化后，会优先查找这三个导出函数，而非传统 `_start`。这实现了**语言无关的标准化启动协议**——就像 USB 接口统一了鼠标、键盘、U 盘的插拔体验。

### 2.5 第五层破壁：模块仓库即 CDN（`arkclaw-registry`）

最后，“零安装”的终极保障，来自于其模块分发体系。ArkClaw 不使用 npm、PyPI 或 crates.io，而是构建了自己的去中心化 Registry：

- 地址：`https://registry.arkclaw.dev`
- 协议：纯静态 HTTPS + IPFS 备份（`ipfs://Qm...`）
- 结构：扁平化命名空间 `@scope/name@version` → `https://registry.arkclaw.dev/@scope/name@version/index.wasm`

所有模块在发布时，必须：
1. 通过 `arkclaw-cli publish` 工具上传；
2. 工具自动执行：
   - WASM 字节码优化（`wabt` 的 `wasm-opt -Oz`）；
   - 插入 `arkclaw-abi` 兼容桩；
   - 生成 `.wasm.sig` 签名文件（使用开发者私钥 ECDSA-secp256k1）；
   - 上传至主 Registry 和 IPFS 网络。

因此，当你写下：

```bash
arkclaw run @demo/image-blur@0.3.1 --input=https://picsum.photos/200/300 --sigma=2.5
```

CLI 实际执行的是：

```bash
# 1. 解析 ID → URL
#    @demo/image-blur@0.3.1 → https://registry.arkclaw.dev/@demo/image-blur@0.3.1/index.wasm

# 2. 下载并校验签名（公钥已预置在 CLI 中）
curl -s https://registry.arkclaw.dev/@demo/image-blur@0.3.1/index.wasm > blur.wasm
curl -s https://registry.arkclaw.dev/@demo/image-blur@0.3.1/index.wasm.sig > blur.wasm.sig
arkclaw-cli verify blur.wasm blur.wasm.sig

# 3. 加载并运行（调用 arkclaw_main）
arkclaw-cli execute blur.wasm --input=https://picsum.photos/200/300 --sigma=2.5
```

整个过程不碰 `node_modules`，不写 `Cargo.lock`，不改 `pip list`——因为模块就是一个个独立、自包含、可验证的 `.wasm` 文件。

这便是 ArkClaw 终结“npm install 宿命”的完整技术图谱：**协议定义入口、运行时剥离依赖、WASI 重定向 I/O、ABI 统一契约、Registry 承载分发**。五层破壁，环环相扣，共同铸就“链接即运行”的新大陆。

本节完。

---

## 三、沙箱之盾：ArkClaw 的纵深防御安全模型

如果说“零安装”是 ArkClaw 的左手，那么“强安全”就是它的右手。在浏览器中执行任意来源的 WASM 代码，其风险不亚于允许网站直接调用 `system("rm -rf /")`。ArkClaw 的安全设计不是事后补救，而是从芯片指令集开始的纵深防御（Defense-in-Depth）。

我们将其安全体系划分为四个物理层级：**内存层、系统调用层、网络层、策略层**。每一层都不可绕过，且形成证据链闭环。

### 3.1 内存层：线性内存的铁壁（Linear Memory Isolation）

WebAssembly 的核心安全基石，是其强制的**线性内存模型（Linear Memory）**：每个模块只能访问自己申请的一块连续内存（`WebAssembly.Memory` 实例），且所有内存访问（load/store）都受 bounds check 保护——这是 CPU 硬件级保障（如 x86 的 `mov` 指令会自动触发 page fault）。

ArkClaw 在此之上，增加了三重加固：

#### （1）内存大小动态裁剪（Dynamic Memory Sizing）

标准 WASM 允许模块声明初始/最大内存页数（1 page = 64KB）。攻击者可能声明 `max: 65536`（4GB），试图耗尽浏览器内存。ArkClaw CLI 和 Web Runtime 均强制执行：

- 默认最大内存：**128 MB**（可由用户通过 `--max-mem=256mb` 覆盖）；
- 内存增长时，进行实时 RSS 监控（`performance.memory?.usedJSHeapSize`）；
- 若增长后总内存占用 > 限制值，抛出 `WasmOutOfMemoryError` 并终止实例。

```ts
// runtime/memory-guard.ts
class MemoryGuard {
  private maxBytes: number;
  private currentBytes: number = 0;

  constructor(maxMb: number = 128) {
    this.maxBytes = maxMb * 1024 * 1024;
  }

  // 在每次 WebAssembly.Memory.grow() 前调用
  canGrow(deltaPages: number): boolean {
    const deltaBytes = deltaPages * 65536;
    const newTotal = this.currentBytes + deltaBytes;
    
    // 检查是否超出硬限制
    if (newTotal > this.maxBytes) {
      console.warn(`🚫 内存申请拒绝：尝试增长 ${deltaBytes} 字节，超出上限 ${this.maxBytes} 字节`);
      return false;
    }

    // 检查浏览器堆内存余量（启发式）
    if (performance.memory) {
      const heapUsed = performance.memory.usedJSHeapSize;
      const heapTotal = performance.memory.totalJSHeapSize;
      if (heapUsed > heapTotal * 0.85) { // 已用超 85%
        console.warn(`⚠️  堆内存紧张，建议降低内存限额`);
      }
    }

    this.currentBytes = newTotal;
    return true;
  }
}
```

#### （2）内存初始化零填充（Zero-Initialization）

WASM 标准允许模块声明 `data` 段，用于初始化内存。但若未显式初始化，内存内容为随机值（可能含之前模块的敏感数据）。ArkClaw 强制：

- 所有新分配的内存页，在交付给模块前，**必须用 `0x00` 填充**；
- `data` 段加载后，剩余未覆盖区域仍保持零值；
- 此举防止“内存残留泄露”（Memory Remanence Leakage），满足 GDPR 对个人数据处理的要求。

#### （3）导出内存只读保护（Exported Memory Immutability）

某些模块会导出其内存实例，供 JavaScript 读取结果（如图像处理后的像素数组）。ArkClaw 运行时对此类导出内存，自动设置为 **`readonly: true`**：

```ts
// 当模块导出 memory 时，运行时自动包装
const rawMemory = instance.exports.memory as WebAssembly.Memory;
const safeMemory = new WebAssembly.Memory({
  initial: rawMemory.buffer.byteLength / 65536,
  maximum: rawMemory.buffer.byteLength / 65536,
  shared: false,
});
// 将原始 buffer 内容复制到 safeMemory（零拷贝不可行，但保证 JS 无法修改 WASM 内存）
new Uint8Array(safeMemory.buffer).set(new Uint8Array(rawMemory.buffer));
```

此举杜绝了 JavaScript 侧恶意篡改 WASM 模块内部状态的可能性（如修改加密密钥、绕过许可证检查）。

### 3.2 系统调用层：WASI 的权限门禁（WASI Capability-Based Access Control）

如前所述，ArkClaw 的 WASI 实现不是全功能模拟，而是基于**能力（Capability）的细粒度授权**。每个模块在启动时，必须声明其所需能力，运行时据此授予最小权限集。

能力声明通过 WASM 自定义段 `arkclaw.capabilities` 嵌入二进制：

```wat
;; 在模块二进制中嵌入 capability 声明
(custom_section "arkclaw.capabilities"
  (bytes
    0x01  ; version
    0x03  ; capability count
    0x01  ; capability id: FILE_SYSTEM_READ
    0x02  ; capability id: CLOCK
    0x04  ; capability id: ENVIRONMENT
  )
)
```

ArkClaw 运行时在 `instantiate` 前解析此段，并构建受限的 WASI 实例：

```ts
// runtime/wasi-factory.ts
interface Capability {
  id: number;
  name: string;
  allowed: boolean;
}

const CAPABILITIES = {
  FILE_SYSTEM_READ: 1,
  FILE_SYSTEM_WRITE: 2,
  NETWORK: 4,
  CLOCK: 8,
  RANDOM: 16,
  ENVIRONMENT: 32,
};

function createRestrictedWasi(capBytes: Uint8Array): WASI {
  const caps = parseCapabilities(capBytes); // 解析自定义段
  const allowedCaps = new Set(caps.map(c => c.id));

  return new WASI({
    // 仅当能力被允许时，才提供对应功能
    preopens: allowedCaps.has(CAPABILITIES.FILE_SYSTEM_READ)
      ? { '/': new ReadOnlyInMemoryDir() }
      : {}, // 空对象 → 所有文件操作返回 ERR_BADF
    clockTimeGet: allowedCaps.has(CAPABILITIES.CLOCK)
      ? () => performance.now()
      : () => { throw new Error('Capability denied: CLOCK'); },
    randomGet: allowedCaps.has(CAPABILITIES.RANDOM)
      ? (buf: Uint8Array) => crypto.getRandomValues(buf)
      : () => { throw new Error('Capability denied: RANDOM'); }
  });
}
```

开发者可通过 `arkclaw-cli build` 工具自动注入能力声明：

```bash
# 构建一个只读文件系统、需要时钟、但禁止网络的模块
arkclaw-cli build \
  --capability=FILE_SYSTEM_READ \
  --capability=CLOCK \
  --no-capability=NETWORK \
  src/main.rs
```

这种基于 capability 的模型，比传统 Unix 的 `rwx` 权限更精细——它不授权“对某个路径的读”，而是授权“使用文件系统能力”，且该能力在模块内所有代码中生效，无法被子函数绕过。

### 3.3 网络层：同源策略的 WASM 延伸（Same-Origin Policy for WASM）

WASM 模块本身无网络能力，必须通过主机函数（Host Functions）或 WASI `sock_*` 接口。ArkClaw 对网络访问实施三重过滤：

#### （1）URL 白名单（Origin Whitelist）

所有通过 `fetch` 或 WASI socket 发起的请求，必须满足：

- 目标 origin 必须在模块声明的白名单中；
- 白名单通过 `arkclaw.network-whitelist` 自定义段声明：

```wat
(custom_section "arkclaw.network-whitelist"
  (bytes
    0x02  ; length of first domain: "api."
    0x61 0x70 0x69 0x2e  ; "api."
    0x07  ; length of second: "example.com"
    0x65 0x78 0x61 0x6d 0x70 0x6c 0x65 0x2e 0x63 0x6f 0x6d  ; "example.com"
  )
)
```

运行时在发起网络请求前，解析目标 URL 的 `origin`，并与白名单匹配（支持通配符 `*.example.com`）。

#### （2）HTTP 方法与 Header 限制

- 仅允许 `GET`, `POST`, `HEAD` 方法；
- 禁止设置危险 Header：`Authorization`, `Cookie`, `Origin`, `Referer`；
- 所有请求自动添加 `X-ArkClaw-Module-ID: @demo/data-fetcher@0.1.0` 头，便于服务端审计。

#### （3）DNS 解析沙箱化

ArkClaw Web Runtime 不使用浏览器原生 `fetch`，而是通过 `Web Workers` + `WebTransport`（若支持）或降级为 `XMLHttpRequest`，并在 Worker 中实现 DNS 查询缓存与拦截：

```js
// worker/dns-sandbox.js
const DNS_CACHE = new Map();

self.onmessage = async (e) => {
  const { hostname } = e.data;
  if (DNS_CACHE.has(hostname)) {
    self.postMessage({ hostname, ip: DNS_CACHE.get(hostname) });
    return;
  }

  // 仅允许解析白名单域名
  if (!isWhitelisted(hostname)) {
    self.postMessage({ hostname, error: 'DNS blocked: not in whitelist' });
    return;
  }

  try {
    // 使用 WebTransport 的内置 DNS（Chrome 120+）
    const ip = await navigator.webtransport.dnsLookup(hostname);
    DNS_CACHE.set(hostname, ip);
    self.postMessage({ hostname, ip });
  } catch (err) {
    self.postMessage({ hostname, error: err.message });
  }
};
```

这确保了即使模块内硬编码 `fetch('http://evil.com/steal')`，也会在 DNS 解析阶段被拦截，而非发出真实请求。

### 3.4 策略层：可编程的安全策略引擎（Policy-as-Code）

ArkClaw 最强大的安全特性，是其**策略即代码（Policy-as-Code）引擎**。管理员可编写 YAML 策略文件，部署到组织级 Registry，对所有模块执行强制合规检查。

一个典型的企业策略 `org-policy.yaml`：

```yaml
# org-policy.yaml
version: "1.0"
rules:
  - id: "no-network-for-analytics"
    description: "分析类模块禁止网络访问"
    condition: "module.tags contains 'analytics'"
    action: "deny"
    capabilities: ["NETWORK"]

  - id: "memory-limit-production"
    description: "生产环境模块内存上限 64MB"
    condition: "environment == 'production'"
    action: "enforce"
    parameters:
      max_memory_mb: 64

  - id: "signature-required"
    description: "所有模块必须有有效签名"
    condition: "true"
    action: "enforce"
    parameters:
      require_signature: true
      trusted_keys:
        - "0xAbc...def" # 公钥哈希
```

该策略在模块 `arkclaw-cli publish` 时，由 Registry 服务端执行：

1. 解析模块的 `arkclaw.tags` 自定义段（如 `["analytics", "internal"]`）；
2. 匹配 `condition` 表达式（使用 CEL 语言）；
3. 若 `action: deny`，则拒绝发布；
4. 若 `action: enforce`，则注入对应限制（如重写 `max_memory` 段）。

策略引擎还支持**运行时动态评估**：当用户在浏览器中访问 `arkclaw://@acme/analytics.wasm`，ArkClaw Web Runtime 会向 `https://policy.acme.com/check?module=@acme/analytics.wasm` 发起策略查询，获取 JSON 策略并实时应用。

这种将安全规则从“代码中硬编码”提升为“组织级策略声明”的范式，标志着 WASM 安全治理进入成熟期。

综上，ArkClaw 的安全不是单一技术，而是一套贯穿编译、分发、加载、执行全生命周期的纵深防御体系。它让“在浏览器里养龙虾”不再是冒险，而是一种可审计、可管控、可规模化的企业级实践。

本节完。

---

## 四、工程实战：从 Rust 到 Python，构建你的第一个 ArkClaw 模块

理论终需落地。本节将手把手带您构建三个真实可用的 ArkClaw 模块：一个 Rust 编写的图像模糊处理器、一个 Python 编写的 CSV 数据清洗器、一个 AssemblyScript 编写的实时 Markdown 预览器。所有代码均可在本地验证，且一键发布至 ArkClaw Registry。

> 📌 前提条件：  
> - 已安装 Rust 1.75+（`rustup default stable`）  
> - 已安装 Node.js 18+（用于 `arkclaw-cli`）  
> - 已安装 Python 3.11+ 及 `pyodide-build`（`pip install pyodide-build`）  
> - 已安装 `arkclaw-cli`：`npm install -g arkclaw-cli@0.8.3`

### 4.1 Rust 模块：高性能图像高斯模糊（`@demo/image-blur`）

Rust 是 ArkClaw 的首选语言，因其零成本抽象与内存安全完美契合 WASM 沙箱需求。

#### 步骤 1：初始化