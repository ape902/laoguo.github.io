---
title: 'Kubernetes API 治理揭秘：SIG Architecture 如何保证向后兼容'
date: '2026-03-18T10:22:00+08:00'
draft: false
tags: ['Kubernetes', 'API 设计', '架构治理', 'SIG Architecture', '云原生']
author: '千吉'
description: '深度访谈 SIG Architecture API 治理负责人 Jordan Liggitt，揭秘 Kubernetes 如何在保持稳定的同时支持创新，以及 CRD 带来的治理挑战。'
---

> **📊 信息来源**
> - 来源平台：Kubernetes Blog
> - 原文链接：[https://kubernetes.io/blog/2026/02/12/sig-architecture-api-spotlight/](https://kubernetes.io/blog/2026/02/12/sig-architecture-api-spotlight/)
> - 热度指数：17.00
> - 生成时间：2026-03-18 10:22

---

# Kubernetes API 治理揭秘：SIG Architecture 如何保证向后兼容

这是 SIG Architecture Spotlight 系列的第五篇访谈，本期我们聚焦 **API 治理（API Governance）** 子项目。

我们有幸与 [Jordan Liggitt](https://github.com/liggitt) 进行了深度对话，他是 SIG Architecture API 治理子项目的负责人，同时也是 SIG Auth 的技术负责人。Jordan 从 2014 年就开始参与 Kubernetes 开发，见证了 Kubernetes API 从早期 beta 版本到 v1 的完整演进历程。

## ① 人物背景：Jordan Liggitt 的 Kubernetes 之旅

### 个人简介

```text
Jordan Liggitt
├─ 职业：Google 软件工程师
├─ 技术角色：
│  ├─ SIG Architecture API 治理子项目负责人
│  ├─ SIG Architecture 代码组织子项目负责人
│  └─ SIG Auth 技术负责人
├─ Kubernetes 贡献时间：2014 年至今（12 年）
└─ 个人标签：基督徒、丈夫、4 个孩子的父亲、业余音乐家
```

### 与 Kubernetes 的缘分

**2014 年：初次接触**

Jordan 在 Red Hat 工作期间负责身份认证和授权相关工作时，首次接触到 Kubernetes。他的第一个 PR 尝试是：

```text
PR #2328: 尝试在 Kubernetes API Server 中添加 OAuth Server
状态：Work-in-Progress（最终未合并）
原因：选择了不同的技术路线
后续：在另一个项目中以不同方式实现
```

**2016-2017 年：成为 API 审查者**

```text
2016 年 → 被标记为 API Reviewer
2017 年 → 成为 API Approver
2019 年 → 正式参与 API 治理项目
```

**关键贡献领域：**

- ✅ Kubernetes 身份认证和授权能力建设
- ✅ 核心 Kubernetes API 的定义和演进（从 v1beta3 到 v1）
- ✅ API 治理和代码组织子项目领导

---

## ② API 治理的目标与范围

### 什么是 Kubernetes 的"API"？

很多人对"API"的理解过于狭隘。Jordan 强调：

```text
大众认知的 API：
└─ REST API（最大、最明显、受众最广）

实际的 API 范围（更广泛）：
├─ 命令行标志（Command-line Flags）
├─ 配置文件（Configuration Files）
├─ 二进制运行方式
├─ 与后端组件（如容器运行时）的通信方式
├─ 数据持久化方式
└─ REST API
```

**关键洞察：**

> "所有这些都是 API。它们的受众可能更窄，因此有更大的灵活性，但它们仍然需要仔细考虑。"

### 核心目标：稳定与创新的平衡

```text
目标 1：保持稳定（Be Stable）
└─ 对用户承诺：不会破坏既有契约
└─ 企业用户依赖这些 API 构建业务系统

目标 2：允许变更（Allow Change）
└─ 支持创新和演进
└─ 适应新的技术场景和需求

平衡挑战：
└─ "保持稳定"如果走向极端 → 永不改变 → 阻碍创新
└─ "允许变更"如果走向极端 → 频繁破坏 → 用户流失
```

**Jordan 的理念：**

> "我们的目标是保持稳定，同时支持创新。稳定性很容易实现——如果你从不改变任何东西的话。但这与演进和增长的目标相矛盾。"

---

## ③ API 治理的工作流程

### 质量关卡：在生命周期的哪些节点介入？

Kubernetes 有明确的 [API 指南和约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)，这些文档是活的，会随着新场景的出现而更新。

**介入时机：**

```text
阶段 1：设计阶段（理想情况）
├─ 参与 KEP（Kubernetes Enhancement Proposal）评审
├─ 审查 API 设计文档
└─ 提供早期反馈，避免后期大改

阶段 2：实现阶段（常见情况）
├─ 审查 PR 中的 API 变更
├─ 检查是否符合 API 约定
└─ 可能提出结构性修改建议
```

### 与 KEP 流程的关系

```text
KEP 详细程度分类：

类型 A：详细设计型
├─ 包含实际的 API 定义
├─ API 治理在设计阶段即可介入
└─ 实现阶段只需检查与设计的一致性

类型 B：概念型
├─ 只描述概念，细节留给实现
├─ API 治理在实现阶段介入
└─ 可能建议结构性变更

权衡：
├─ 详细设计 upfront → 前期工作量大，后期变更少
└─ 迭代式发现 → 前期灵活，后期可能大改
```

**Jordan 的观点：**

> "人们和团队的工作方式不同，我们很灵活，愿意在设计阶段或实现时提供咨询。"

### 自动化保障工具

除了人工审查，API 治理还有自动化工具支持：

```text
自动化工具：
├─ API Linter：检查 API 命名、字段类型等约定
├─ 自动化检查：确保 spec/status 语义正确
└─ 验证工具：捕获人类可能忽略的问题

工作流程：
新问题出现 → 人工分析 → 更新文档/工具 → 自动化检查后续类似问题
```

---

## ④ CRD：API 治理的转折点

### 分水岭时刻：自定义资源的诞生

Jordan 认为，**Custom Resources（自定义资源）** 的出现是 API 治理的转折点。

**CRD 出现之前：**

```text
特点：
├─ 每个 API 都由核心团队手工制作
├─ 完全审查，严格控制
├─ 存在不一致，但一切尽在掌握
└─ API 治理相对简单

问题：
└─ 扩展性差，无法满足社区多样化需求
```

**CRD 出现之后（v1.0 之前）：**

```text
特点：
├─ 任何人都可以定义任何 API
├─ 甚至不需要 Schema
├─ 极其强大，立即赋能
└─ 治理团队陷入"追赶"状态

问题：
└─ 稳定性和一致性受到挑战
```

**CRD GA（通用可用）之后：**

```text
改进：
├─ Schema 成为必需
├─ 保留向后兼容的逃生通道
└─ 逐步增强验证能力

当前状态：
└─ 内置验证规则刚刚达到 GA（最近几个版本）
```

### CRD 治理的三大主题

```text
主题 1：定义 Schema
├─ 确保 API 有明确的结构
├─ 防止随意定义字段
└─ 提供基本的类型安全

主题 2：验证数据
├─ 内置验证规则 GA
├─ 提供与内置类型相当的验证能力
└─ 防止无效数据进入系统

主题 3：处理已存在的无效数据
├─ 现实问题：已有大量不符合新规范的 CRD
├─ 解决方案：Ratcheting Validation（分级验证）
└─ 允许数据逐步改善，不破坏现有对象
```

**Ratcheting Validation 示例：**

```yaml
# 场景：CRD 要求字段必须为正整数
# 现有数据：包含负数或零

传统验证：
└─ 拒绝所有包含无效数据的对象
└─ 结果：破坏现有工作负载

分级验证：
├─ 允许已存在的无效数据继续存在
├─ 但新创建或更新的对象必须符合规范
└─ 结果：逐步改善，不破坏现有系统
```

**Jordan 的评价：**

> "CRD 开启了'一切皆有可能'的时代。内置验证规则是第二个重要里程碑：让一致性回归。"

---

## ⑤ API 治理与相关 SIG 的关系

### 组织架构梳理

```text
SIG Architecture（架构 SIG）
├─ 职责：设定整个系统的方向
├─ 与 API Machinery 合作：确保系统支持该方向
└─ 与其他 SIG 合作：定义约定和模式

API Machinery（API 机器 SIG）
├─ 职责：提供构建 API 的实际代码和工具
├─ 不审查具体领域的 API（存储、网络、调度等）
└─ 提供基础设施支持

API Governance（API 治理）
├─ 职责：与其他 SIG 合作定义约定和模式
├─ 确保一致性地使用 API Machinery 提供的能力
└─ 审查具体 API 是否符合约定
```

**类比理解：**

```text
API Machinery → 提供砖块、水泥、工具（基础设施）
     ↓
SIG Architecture → 制定建筑规范和设计原则（方向指导）
     ↓
API Governance → 审查具体建筑是否符合规范（质量把关）
     ↓
其他 SIG（Storage/Network/Scheduling...）→ 建造具体建筑（实现功能）
```

---

## ⑥ 发布周期中的工作节奏

### 发布周期各阶段的工作负载

Kubernetes 有明确的发布周期，包括 **Enhancements Freeze（增强冻结）** 和 **Code Freeze（代码冻结）**。

```text
发布周期阶段：
├─ 阶段 1：Enhancements Freeze 前
│  └─ 设计工作增加
│  └─ 参与新功能的 API 设计评审
│
├─ 阶段 2：Enhancements Freeze → Code Freeze
│  └─ 实现工作增加
│  └─ 审查 API 实现的正确性
│
├─ 阶段 3：Code Freeze 后
│  └─ 修复关键问题
│  └─ 准备发布
│
└─ 阶段 4：发布后 → 下个周期开始
   └─ 长期设计工作
   └─ 文档和工具改进
```

### 反模式：最后一刻才提交审查

Jordan 提到一个常见的反模式：

```text
反模式：
├─ 团队花费数月思考一个大功能
├─ 在 Enhancements Freeze 前 3 周才提交审查
└─ "这是设计，请评审"

问题：
├─ 审查时间不足
├─ 重大变更来不及调整
└─ 可能导致功能延期到下个版本

正确做法：
├─ 尽早让 API 治理参与
├─ 在冻结期之间的空闲时间进行长期审查
└─ 分阶段评审：设计 → 实现 → 最终审查
```

**Jordan 的建议：**

> "对于有 API 影响的大型变更，尽早让 API 治理参与要好得多。在冻结期之间，人们有空闲时间，那是长期审查工作的最佳时机。"

---

## ⑦ 如何参与 API 治理

### 新贡献者的参与路径

对于想参与 API 治理的贡献者，Jordan 给出了具体建议：

```text
推荐路径：
├─ 步骤 1：跟随一个具体的变更
│  └─ 不要试图一次性学习所有内容
│  └─ 选择一个小的 API 变更
│
├─ 步骤 2：观察完整流程
│  ├─ 设计阶段 → 实现阶段 → 审查阶段
│  └─ 了解每个节点的要求和考量
│
├─ 步骤 3：参与高带宽审查
│  ├─ 通过视频会议实时讨论
│  ├─ 询问是否有时间一起审查设计或 PR
│  └─ 观察这些讨论非常有教育意义
│
└─ 步骤 4：逐步升级
   ├─ 小变更 → 大变更 → 新 API
   └─ 在实践中建立对约定的理解
```

### 参与渠道

| 渠道 | 链接 | 说明 |
|------|------|------|
| **SIG Architecture 会议** | [社区日历](https://www.kubernetes.dev/resources/calendar/) | 参加周会，了解当前讨论 |
| **API Review 流程** | [API Review Process](https://github.com/kubernetes/community/blob/master/sig-architecture/api-review-process.md) | 了解审查流程 |
| **API 约定文档** | [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) | 学习 API 设计规范 |
| **Slack** | [#sig-architecture](https://kubernetes.slack.com/messages/sig-architecture) | 日常讨论、提问 |

---

## ⑧ 为什么兼容性如此重要？

### 用户视角 vs 贡献者视角

Jordan 在访谈最后强调了一个关键观点：

```text
贡献者视角：
├─ 看到兼容性要求是痛苦的障碍
├─ 阻止清理代码
├─ 需要繁琐的工作
└─ 让开发变慢

用户视角：
├─ 依赖这些 API 构建业务系统
├─ 集成了自动化流程和工具
├─ 需要长期稳定性保证
└─ 无法承受频繁破坏性变更
```

### Kubernetes 对用户的承诺

> "我们关心兼容性和稳定性的原因是为了我们的用户。很容易让贡献者认为这些要求是痛苦的障碍……但用户已经与我们的系统集成，我们对他们做出了承诺：我们希望他们相信我们不会破坏这个契约。"

**即使这意味着：**

- ✅ 需要更多工作
- ✅ 进展更慢
- ✅ 需要保留重复代码

**核心原则：**

> "我们不是要成为障碍，而是要让用户的生活更美好。"

### 面向未来的设计思维

API 治理团队经常问的问题：

```text
核心问题：
├─ "你现在想做某件事……以后如何演进而不破坏它？"
├─ "我们假设未来会知道更多，设计如何留出空间？"
└─ "我们假设会犯错，如何留下改进的通道同时保持兼容性？"

设计原则：
├─ 为未来留余地
├─ 接受会犯错的事实
├─ 设计可演进的架构
└─ 在兼容性承诺下寻找改进路径
```

---

## ⑨ 总结与启示

### 核心教训

```text
1. API 治理的本质是平衡
   └─ 稳定 vs 创新
   └─ 控制 vs 灵活
   └─ 短期效率 vs 长期可持续性

2. CRD 是双刃剑
   └─ 开启了无限扩展的可能性
   └─ 也带来了治理挑战
   └─ 需要持续投入才能保持一致性

3. 自动化 + 人工审查是最佳实践
   └─ 自动化工具捕获常见问题
   └─ 人工审查处理复杂场景
   └─ 文档和工具持续演进

4. 用户价值是最终目标
   └─ 兼容性要求不是官僚主义
   └─ 是对用户的承诺和责任
   └─ 即使增加工作量也要坚持
```

### 对其他项目的启示

```text
可借鉴的实践：
├─ 建立明确的 API 约定文档
├─ 在设计阶段就介入审查
├─ 提供自动化工具辅助检查
├─ 保持文档和工具的持续更新
├─ 建立分级验证机制处理历史包袱
└─ 始终牢记用户价值

适用场景：
├─ 大型开源项目
├─ 企业级平台开发
├─ 微服务架构治理
└─ 任何需要长期稳定性的 API 系统
```

### 行动建议

对于正在构建 API 系统的团队：

```text
短期（1-2 周）：
├─ 审查现有 API 文档和约定
├─ 识别缺乏规范的领域
└─ 建立基础的 API 审查流程

中期（1-3 个月）：
├─ 引入自动化检查工具
├─ 建立设计阶段的审查机制
└─ 培训团队理解 API 治理价值

长期（3-6 个月）：
├─ 建立完整的 API 生命周期管理
├─ 实现分级验证处理历史问题
└─ 持续改进文档和工具
```

---

**参考资料：**

- [SIG Architecture API Governance](https://github.com/kubernetes/community/blob/master/sig-architecture/README.md#architecture-and-api-governance-1)
- [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [API Review Process](https://github.com/kubernetes/community/blob/master/sig-architecture/api-review-process.md)
- [Kubernetes Enhancement Proposals (KEPs)](https://github.com/kubernetes/enhancements/blob/master/keps/README.md)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Jordan Liggitt 的 GitHub](https://github.com/liggitt)
