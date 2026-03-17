---
title: 'Kubernetes Image Promoter 重构：一场无人察觉的 20% 代码删减'
date: '2026-03-18T04:22:00+08:00'
draft: false
tags: ['Kubernetes', '云原生', 'DevOps', '容器镜像', 'SLSA']
author: '千吉'
description: '深度解析 Kubernetes 镜像推广工具 kpromo 的重构历程：如何在保证零用户感知的前提下，删减 20% 代码，提升 10 倍性能，并添加 SLSA 溯源证明。'
---

> **📊 信息来源**
> - 来源平台：Kubernetes Blog
> - 原文链接：[https://kubernetes.io/blog/2026/03/17/image-promoter-rewrite/](https://kubernetes.io/blog/2026/03/17/image-promoter-rewrite/)
> - 热度指数：17.00
> - 生成时间：2026-03-18 04:22

---

# Kubernetes Image Promoter 重构：一场无人察觉的 20% 代码删减

每次你从 `registry.k8s.io` 拉取容器镜像时，这些镜像都是通过 [kpromo](https://github.com/kubernetes-sigs/promo-tools)（Kubernetes 镜像推广工具）到达那里的。它负责将镜像从临时注册表复制到生产注册表，使用 [cosign](https://sigstore.dev) 进行签名，在 20 多个区域镜像站之间复制签名，并生成 [SLSA](https://slsa.dev) 溯源证明。如果这个工具出问题，Kubernetes 发布就会中断。

在过去的几周里，Kubernetes 社区从头重写了它的核心代码，删除了 20% 的代码库，使其性能大幅提升，而没有人注意到——这正是他们的目标。

## ① 历史背景：从内部工具到社区项目

### 起源（2018 年）

Image Promoter 始于 2018 年末，由 [Linus Arver](https://github.com/listx) 作为 Google 内部项目发起。目标很简单：用社区拥有的、基于 GitOps 的工作流，取代手动的、需要 Google 员工审批的镜像复制流程。

```text
工作流程：
推送到临时注册表 → 提交包含 YAML 清单的 PR → 审核并合并 → 自动化处理剩余工作
```

[KEP-1734](https://github.com/kubernetes/enhancements/blob/master/keps/sig-release/1734-k8s-image-promoter/README.md) 正式化了这一提案。

### 发展历程（2019-2025 年）

| 时间 | 里程碑 | 贡献者 |
|------|--------|--------|
| 2019 年初 | 代码迁移到 kubernetes-sigs/k8s-container-image-promoter | Linus Arver |
| 2020-2022 | 整合多个工具（cip、gh2gcs、krel promote-images）为单一 CLI kpromo | Stephen Augustus |
| 2022-2023 | 添加 cosign 签名和 SBOM 支持 | Adolfo Garcia Veytia (Puerco) |
| 2023-2024 | 构建漏洞扫描功能 | Tyler Ferrara |
| 持续维护 | 保持项目健康和可发布状态 | Carlos Panato |

**数据概览：**
- 42 位贡献者
- 约 3,500 次提交
- 超过 60 个版本发布

### 技术债务积累

到 2025 年，代码库承载了七年来自多个 SIG 和子项目的增量添加。README 中直言不讳：

> "你会看到重复的代码、多种实现相同功能的技术，以及多个 TODO。"

生产环境的镜像推广作业经常需要 30 分钟以上，并且频繁因速率限制错误而失败。

## ② 核心问题：为什么需要重构？

### 问题清单

```text
1. 性能问题
   - 生产推广作业耗时 >30 分钟
   - 频繁的速率限制错误导致失败

2. 架构问题
   - 核心推广逻辑变成单体架构
   - 难以扩展（issue #1177）
   - 测试困难

3. 功能迭代受阻
   - 添加溯源证明功能痛苦
   - 漏洞扫描集成困难
```

### SIG Release 路线图中的待办事项

在 [SIG Release 路线图](https://github.com/kubernetes/sig-release/blob/master/roadmap.md) 中，有两个工作项已经搁置一段时间：

1. **"重写 artifact promoter"**
2. **"使 artifact 验证更健壮"**

这些议题在 SIG Release 会议和 KubeCon 上讨论过，[项目面板 #171](https://github.com/orgs/kubernetes/projects/171) 上的开放研究 spike 捕获了在推进之前需要回答的 8 个问题。

## ③ 重构策略：9 个阶段逐步推进

### 总体思路

2026 年 2 月，团队开启了 [issue #1701](https://github.com/kubernetes-sigs/promo-tools/issues/1701)（"重写 artifact promoter 流水线"），在一个跟踪 issue 中回答了所有 8 个 spike 问题。

**关键原则：** 每个阶段独立审查、合并和验证。

### 9 个阶段详解

```text
阶段 1: 速率限制 (#1702)
├─ 重写速率限制逻辑
├─ 对所有注册表操作进行适当的节流
└─ 实现自适应退避策略

阶段 2: 接口抽象 (#1704)
├─ 将注册表和认证操作置于清晰的接口之后
├─ 支持独立替换和测试
└─ 为后续重构奠定基础

阶段 3: 流水线引擎 (#1705)
├─ 构建流水线引擎
├─ 将推广作为独立阶段序列运行
└─ 替代单一大型函数

阶段 4: 溯源证明 (#1706)
├─ 添加 SLSA 溯源验证
├─ 验证临时镜像的完整性
└─ 确保供应链安全

阶段 5: 扫描器和 SBOM (#1709)
├─ 添加漏洞扫描支持
├─ 添加 SBOM（软件物料清单）支持
├─ 默认切换到新流水线引擎
└─ 发布 v4.2.0，在生产环境中浸泡测试

阶段 6: 拆分签名与复制 (#1713)
├─ 将镜像签名与签名复制分离为独立阶段
├─ 消除导致大多数生产失败的速率限制竞争
└─ 签名复制改为独立的周期性 Prow 作业

阶段 7: 移除旧流水线 (#1712)
└─ 完全删除旧代码路径

阶段 8: 移除旧依赖 (#1716)
├─ 删除审计子系统
├─ 删除已弃用工具
└─ 删除 e2e 测试基础设施

阶段 9: 删除单体 (#1718)
└─ 移除旧的单体核心及其支持包
```

### 版本发布节奏

```text
v4.2.0 → 新流水线引擎 + 生产浸泡
v4.3.0 → 完全移除旧代码
v4.4.0 → 所有后续改进 + 默认启用溯源证明
```

**成果：** 阶段 7-9 共删除数千行代码。

## ④ 新流水线架构：7 个清晰分离的阶段

### 流水线图

```text
Setup → Plan → Provenance → Validate → Promote → Sign → Attest
```

### 各阶段职责

| 阶段 | 职责 | 关键操作 |
|------|------|----------|
| **Setup** | 验证选项、预热 TUF 缓存 | 配置校验、缓存初始化 |
| **Plan** | 解析清单、读取注册表、计算需推广的镜像 | 并行读取 1,350 个注册表 |
| **Provenance** | 验证临时镜像的 SLSA 证明 | 供应链完整性检查 |
| **Validate** | 检查 cosign 签名、干跑模式退出点 | 签名验证 |
| **Promote** | 服务器端复制镜像、保留摘要 | 跨注册表复制 |
| **Sign** | 使用无密钥 cosign 签名推广的镜像 | 签名生成 |
| **Attest** | 使用专用 in-toto 谓词类型生成推广溯源证明 | 证明生成 |

### 架构优势

```text
1. 顺序执行
   └─ 每个阶段独占完整的速率限制预算
   └─ 不再有竞争问题

2. 签名复制独立
   └─ 作为独立的周期性 Prow 作业运行
   └─ 不阻塞主推广流水线

3. 可测试性
   └─ 每个阶段可独立测试
   └─ 支持本地注册表集成测试 (#1746)
```

## ⑤ 性能优化：从 17 小时到 15 分钟

### 优化措施详解

#### 1. 并行注册表读取 (#1736)

```text
优化前：
└─ 顺序读取 1,350 个注册表
   └─ Plan 阶段耗时：~20 分钟

优化后：
└─ 并行读取 1,350 个注册表
   └─ Plan 阶段耗时：~2 分钟
   └─ 性能提升：10 倍
```

#### 2. 两阶段标签列表 (#1761)

```text
问题：
└─ 需要检查 46,000 个镜像组 × 20+ 镜像站 = 近百万次 API 调用

发现：
└─ 约 57% 的镜像根本没有签名（因为它们在启用签名之前就被推广了）

解决方案：
└─ 第一阶段：仅检查源注册表
└─ 第二阶段：仅对有签名的镜像检查镜像站
└─ API 调用减少：约 50%
```

#### 3. 复制前检查源 (#1727)

```text
优化前：
└─ 遍历所有镜像站检查签名是否存在
└─ 耗时：~17 小时

优化后：
└─ 先检查主注册表上签名是否存在
└─ 稳态下大多数签名已复制
└─ 耗时：~15 分钟
└─ 性能提升：68 倍
```

#### 4. 每请求超时 (#1763)

```text
问题：
└─ 观察到间歇性挂起
└─ 停滞的连接阻塞流水线超过 9 小时

解决方案：
└─ 每个网络操作独立超时
└─ 自动重试瞬态故障
```

#### 5. 连接复用 (#1759)

```text
优化：
└─ 跨操作复用 HTTP 连接和认证状态
└─ 消除冗余的令牌协商
└─ 解决了 2023 年的长期请求 (#842)
```

### 性能对比汇总

| 优化项 | 优化前 | 优化后 | 提升倍数 |
|--------|--------|--------|----------|
| Plan 阶段 | 20 分钟 | 2 分钟 | 10x |
| 签名复制检查 | 17 小时 | 15 分钟 | 68x |
| API 调用 | 100% | ~50% | 2x 减少 |

## ⑥ 最终成果：数据说话

### 代码变化

```text
添加行数：    +10,000 行
删除行数：    -16,000 行
净减少：      -5,000 行（代码库缩小 20%）
```

### 发布成果

```text
合并 PR 数：   40+
发布版本：    3 个（v4.2.0, v4.3.0, v4.4.0）
关闭问题：    19 个长期问题
```

### 新增功能

在代码库缩小 1/5 的同时，获得了：

- ✅ 溯源证明生成
- ✅ 流水线引擎
- ✅ 漏洞扫描集成
- ✅ 并行化操作
- ✅ 重试逻辑
- ✅ 本地注册表集成测试
- ✅ 独立签名复制模式

### 零用户感知变更

**硬性要求：** 对用户完全透明

```text
保持不变的：
├─ kpromo cip 命令接受相同的标志
├─ 读取相同的 YAML 清单
├─ post-k8sio-image-promo Prow 作业持续工作
├─ kubernetes/k8s.io 中的推广清单无需更改
└─ 用户无需更新工作流或配置
```

### 生产环境中的小插曲

```text
问题 1 (#1731)：
├─ 注册表密钥不匹配
├─ 导致所有镜像显示为"丢失"
├─ 修复时间：数小时内

问题 2 (#1733)：
├─ 默认线程数设置为零
├─ 阻塞所有 goroutine
├─ 修复时间：数小时内

应对策略：
└─ 分阶段发布（v4.2.0 → v4.3.0）提供清晰的回滚路径
└─ 幸运的是从未需要回滚
```

## ⑦ 未来规划：下一步做什么？

### 待优化的最大成本点

签名复制仍然是推广周期中最昂贵的部分。

### 方案 1：消除跨镜像站复制 (#1762)

```text
提案：
└─ 让 archeio（registry.k8s.io 重定向服务）将签名标签请求
   路由到单个规范上游，而非每区域后端

收益：
└─ 每次推广周期减少数千次 API 调用
└─ 进一步简化代码库

状态：
└─ 需要与 SIG Release 和基础设施团队进一步讨论
```

### 方案 2：将签名移至注册表基础设施

```text
提案：
└─ 将签名功能更接近注册表基础设施本身

收益：
└─ 减少中间层
└─ 降低复杂性

状态：
└─ 需要进一步讨论
```

## ⑧ 致谢：七年社区协作

这个项目是跨越七年的社区努力。感谢以下贡献者：

- [Linus](https://github.com/listx) - 创始人
- [Stephen](https://github.com/justaugustus) - 工具整合
- [Adolfo](https://github.com/puerco) - 签名和 SBOM
- [Carlos](https://github.com/cpanato) - 持续维护
- [Ben](https://github.com/BenTheElder)
- [Marko](https://github.com/xmudrii)
- [Lauri](https://github.com/lasomethingsomething)
- [Tyler](https://github.com/tylerferrara)
- [Arnaud](https://github.com/ameukam)
- 以及多年来贡献代码、审查和规划的许多人

SIG Release 和 Release Engineering 社区提供了背景、讨论和对每次 Kubernetes 发布所依赖的基础设施重构的耐心。

## ⑨ 总结与启示

### 核心教训

```text
1. 渐进式重构
   └─ 分阶段进行，每阶段独立验证
   └─ 降低风险，便于回滚

2. 零用户感知
   └─ 保持向后兼容
   └─ 用户无需更改工作流

3. 性能优先
   └─ 识别瓶颈（签名复制）
   └─ 针对性优化（并行化、缓存、超时）

4. 代码质量
   └─ 删除 20% 代码反而增加功能
   └─ 清晰的接口和阶段分离
```

### 适用场景

这篇重构经验适用于：

- 🔧 维护长期运行的基础设施项目
- 🔧 需要在不中断服务的情况下进行重大架构变更
- 🔧 处理速率限制和 API 调用优化
- 🔧 实现供应链安全和溯源证明

### 参与方式

如果你想参与：

- Slack: [#release-management](https://kubernetes.slack.com/archives/C2C40FMNF)
- GitHub: [kubernetes-sigs/promo-tools](https://github.com/kubernetes-sigs/promo-tools)

---

**参考资料：**

- [KEP-1734: Kubernetes Image Promoter](https://github.com/kubernetes/enhancements/blob/master/keps/sig-release/1734-k8s-image-promoter/README.md)
- [promo-tools 仓库](https://github.com/kubernetes-sigs/promo-tools)
- [SLSA 规范](https://slsa.dev)
- [cosign 文档](https://sigstore.dev)
- [in-toto 证明框架](https://in-toto.io)
