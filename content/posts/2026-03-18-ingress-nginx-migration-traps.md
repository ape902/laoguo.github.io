---
title: 'Ingress-NGINX 退役前必知：5 个令人惊讶的行为陷阱'
date: '2026-03-18T08:22:00+08:00'
draft: false
tags: ['Kubernetes', 'Ingress', '网络', '迁移指南', 'Gateway API']
author: '千吉'
description: 'Kubernetes 将于 2026 年 3 月退役 Ingress-NGINX。本文揭示 5 个隐藏的行为陷阱，帮助你在迁移到 Gateway API 时避免生产事故。'
---

> **📊 信息来源**
> - 来源平台：Kubernetes Blog
> - 原文链接：[https://kubernetes.io/blog/2026/02/27/ingress-nginx-before-you-migrate/](https://kubernetes.io/blog/2026/02/27/ingress-nginx-before-you-migrate/)
> - 热度指数：17.00
> - 生成时间：2026-03-18 08:22

---

# Ingress-NGINX 退役前必知：5 个令人惊讶的行为陷阱

正如 [2025 年 11 月宣布](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) 的那样，Kubernetes 将于 **2026 年 3 月退役 Ingress-NGINX**。

尽管 Ingress-NGINX 被广泛使用，但它充满了令人惊讶的默认行为和副作用——这些行为可能今天就存在于你的集群中。本文重点介绍这些行为，帮助你在安全迁移的同时，有意识地决定保留哪些行为。

**核心风险模式：** 看似正确的转换，如果不考虑 Ingress-NGINX 的怪癖，仍然会导致生产事故。

## ① 背景：Ingress-NGINX 退役时间线

### 重要声明

```text
⚠️ 关键信息
├─ 退役时间：2026 年 3 月
├─ 影响范围：约 50% 的云原生环境
├─ 替代方案：Gateway API + 兼容的 Ingress Controller
└─ 紧急程度：高（立即开始迁移评估）
```

### 两个 NGINX Ingress 的区别

很多用户混淆了两个不同的项目：

| 项目 | Ingress-NGINX | NGINX Ingress |
|------|---------------|---------------|
| **维护方** | Kubernetes 社区 | F5 (NGINX 公司) |
| **GitHub** | [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx) | [nginxinc/kubernetes-ingress](https://github.com/nginxinc/kubernetes-ingress) |
| **状态** | 🚫 2026 年 3 月退役 | ✅ 继续维护 |
| **文档** | kubernetes.io | docs.nginx.com |

**本文仅讨论 Ingress-NGINX（社区版本）**。

## ② 陷阱一：正则匹配是前缀匹配且不区分大小写

### 问题场景

假设你想将所有路径仅为三个大写字母的请求路由到 httpbin 服务：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: regex-match-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: regex-match.example.com
    http:
      paths:
      - path: "/[A-Z]{3}"
        pathType: ImplementationSpecific
        backend:
          service:
            name: httpbin
            port:
              number: 8000
```

### 意外行为

```text
预期：只匹配 /ABC、/XYZ 等三个大写字母的路径
实际：匹配任何以三个字母开头的路径（包括小写）

测试：
curl -H "Host: regex-match.example.com" http://<ingress-ip>/uuid
→ 成功路由到 httpbin（因为 /uuid 以三个字母开头）
```

**原因：** Ingress-NGINX 的正则匹配是 **前缀匹配** 且 **不区分大小写**。

### Gateway API 迁移陷阱

如果你直接转换为 Gateway API：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: regex-match-route
spec:
  hostnames:
  - regex-match.example.com
  parentRefs:
  - name: <your-gateway>
  rules:
  - matches:
    - path:
        type: RegularExpression
        value: "/[A-Z]{3}"
    backendRefs:
    - name: httpbin
      port: 8000
```

**问题：** 大多数 Envoy -based 实现（Istio、Envoy Gateway、Kgateway）执行 **完整的大小写敏感匹配**。

```text
结果：
├─ /uuid → 404 Not Found（不匹配）
├─ /ABC → 200 OK（匹配）
└─ 生产事故：原本能访问的路径突然失效
```

### 正确迁移方案

**方案 1：显式指定大小写不敏感和前缀匹配**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: regex-match-route
spec:
  hostnames:
  - regex-match.example.com
  parentRefs:
  - name: <your-gateway>
  rules:
  - matches:
    - path:
        type: RegularExpression
        value: "/[a-zA-Z]{3}.*"  # 显式包含大小写，添加 .* 表示前缀
    backendRefs:
    - name: httpbin
      port: 8000
```

**方案 2：使用 `(?i)` 标志**

```yaml
  rules:
  - matches:
    - path:
        type: RegularExpression
        value: "(?i)/[a-z]{3}.*"  # (?i) 表示大小写不敏感
```

### 检查清单

```text
迁移前检查：
□ 识别所有使用 use-regex: "true" 的 Ingress
□ 测试现有正则的实际匹配行为（可能比你想象的更宽泛）
□ 在 Gateway API 中显式指定大小写和前缀语义
□ 在预发布环境验证所有路径匹配
```

## ③ 陷阱二：use-regex 注解影响同一 host 的所有 Ingress

### 问题场景

假设有两个 Ingress，一个启用了正则，另一个使用精确匹配：

```yaml
# Ingress 1：启用正则
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: regex-match-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: regex-match.example.com
    http:
      paths:
      - path: "<某个正则模式>"
        pathType: ImplementationSpecific
        backend:
          service:
            name: <your-backend>
            port:
              number: 8000

---
# Ingress 2：精确匹配（但有拼写错误）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exact-match-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: regex-match.example.com
    http:
      paths:
      - path: "/Header"  # 拼写错误，应该是 /headers
        pathType: Exact
        backend:
          service:
            name: httpbin
            port:
              number: 8000
```

### 意外行为

```text
预期：/headers 返回 404（因为精确匹配的是 /Header）
实际：/headers 成功路由到 httpbin

原因：
├─ regex-match-ingress 启用了 use-regex: "true"
├─ 同一 host（regex-match.example.com）的所有路径都被视为正则
├─ /Header 作为正则模式，大小写不敏感前缀匹配
└─ /headers 匹配 /Header 模式
```

**测试验证：**

```bash
curl -H "Host: regex-match.example.com" http://<ingress-ip>/headers
# 返回 httpbin 的 /headers 响应（包含请求头）
```

### Gateway API 迁移陷阱

Gateway API **不会** 静默地将 Exact/Prefix 匹配转换为正则：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: migrated-route
spec:
  hostnames:
  - regex-match.example.com
  rules:
  - matches:
    - path:
        type: Exact
        value: "/Header"  # 保留了拼写错误
    backendRefs:
    - name: httpbin
      port: 8000
```

```text
结果：
├─ /headers → 404 Not Found（精确匹配，不匹配）
├─ /Header → 200 OK
└─ 生产事故：依赖 /headers 的应用突然失效
```

### 正确迁移方案

**方案 1：保留大小写不敏感前缀匹配行为**

```yaml
  rules:
  - matches:
    - path:
        type: RegularExpression
        value: "(?i)/Header"  # 显式指定大小写不敏感
    backendRefs:
    - name: httpbin
      port: 8000
```

**方案 2：修复拼写错误（推荐）**

```yaml
  rules:
  - matches:
    - path:
        type: Exact
        value: "/headers"  # 修复错误
    backendRefs:
    - name: httpbin
      port: 8000
```

### 检查清单

```text
迁移前检查：
□ 列出同一 host 下的所有 Ingress
□ 识别哪些 Ingress 启用了 use-regex
□ 测试所有路径的实际匹配行为（可能有隐藏的交叉影响）
□ 审查路径拼写（Ingress-NGINX 可能掩盖了拼写错误）
□ 在 Gateway API 中为每个路由显式指定匹配语义
```

## ④ 陷阱三：rewrite-target 注解隐式启用正则

### 问题场景

```yaml
# Ingress 1：使用路径重写
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-target-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: "/uuid"
spec:
  ingressClassName: nginx
  rules:
  - host: rewrite-target.example.com
    http:
      paths:
      - path: "/IP"  # 拼写错误，应该是 /ip
        pathType: Exact
        backend:
          service:
            name: httpbin
            port:
              number: 8000

---
# Ingress 2：普通精确匹配
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exact-match-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: rewrite-target.example.com
    http:
      paths:
      - path: "/Header"  # 拼写错误，应该是 /headers
        pathType: Exact
        backend:
          service:
            name: httpbin
            port:
              number: 8000
```

### 意外行为

```text
关键发现：rewrite-target 注解隐式添加了 use-regex: "true"！

测试结果：
├─ /ip → 路径重写为 /uuid，路由到 httpbin（因为 /ip 大小写不敏感匹配 /IP）
├─ /headers → 路由到 httpbin（因为 /headers 大小写不敏感匹配 /Header）
└─ 两个拼写错误都被正则的模糊匹配掩盖了
```

**影响范围：**

```text
rewrite-target 注解的副作用：
├─ 静默启用 use-regex: "true"
├─ 同一 host 的所有路径都被视为正则
├─ 所有匹配变为大小写不敏感前缀匹配
└─ 拼写错误被掩盖，迁移时暴露
```

### Gateway API 迁移方案

Gateway API 使用 **HTTP URL Rewrite Filter**，不会静默转换匹配类型：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-target-route
spec:
  hostnames:
  - rewrite-target.example.com
  parentRefs:
  - name: <your-gateway>
  rules:
  - matches:
    - path:
        type: Exact
        value: "/IP"  # 现在是真正的精确匹配
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /uuid
    backendRefs:
    - name: httpbin
      port: 8000
  - matches:
    - path:
        type: Exact
        value: "/Header"
    backendRefs:
    - name: httpbin
      port: 8000
```

```text
迁移后的行为：
├─ /ip → 404 Not Found（精确匹配 /IP 不匹配）
├─ /IP → 200 OK，路径重写为 /uuid
├─ /headers → 404 Not Found
└─ /Header → 200 OK
```

### 正确迁移方案

**方案 1：保留原有行为（显式使用正则）**

```yaml
  rules:
  - matches:
    - path:
        type: RegularExpression
        value: "(?i)/IP.*"  # 大小写不敏感前缀匹配
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /uuid
    backendRefs:
    - name: httpbin
      port: 8000
  - matches:
    - path:
        type: RegularExpression
        value: "(?i)/Header.*"
    backendRefs:
    - name: httpbin
      port: 8000
```

**方案 2：修复拼写错误（推荐）**

```yaml
  rules:
  - matches:
    - path:
        type: Exact
        value: "/ip"  # 修复拼写
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /uuid
    backendRefs:
    - name: httpbin
      port: 8000
  - matches:
    - path:
        type: Exact
        value: "/headers"  # 修复拼写
    backendRefs:
    - name: httpbin
      port: 8000
```

### 检查清单

```text
迁移前检查：
□ 搜索所有使用 rewrite-target 注解的 Ingress
□ 测试实际匹配行为（可能有隐藏的模糊匹配）
□ 识别被正则掩盖的拼写错误
□ 决定是保留行为还是修复错误
□ 在 Gateway API 中显式配置 URL Rewrite Filter
```

## ⑤ 陷阱四：缺少尾部斜杠会自动重定向

### 问题场景

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trailing-slash-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: trailing-slash.example.com
    http:
      paths:
      - path: "/my-path/"
        pathType: Exact
        backend:
          service:
            name: <your-backend>
            port:
              number: 8000
```

### 意外行为

```text
预期：/my-path 返回 404（因为精确匹配的是 /my-path/）
实际：/my-path 返回 301 重定向到 /my-path/

测试：
curl -i -H "Host: trailing-slash.example.com" http://<ingress-ip>/my-path

响应：
HTTP/1.1 301 Moved Permanently
Location: http://trailing-slash.example.com/my-path/
```

**适用范围：**

```text
自动重定向条件：
├─ 路径类型：Exact 或 Prefix
├─ 唯一差异：仅尾部斜杠
├─ 不适用于：正则模式路径
└─ 重定向类型：301 Moved Permanently
```

### Gateway API 迁移陷阱

Gateway API **不会** 静默配置任何重定向：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: trailing-slash-route
spec:
  hostnames:
  - trailing-slash.example.com
  parentRefs:
  - name: <your-gateway>
  rules:
  - matches:
    - path:
        type: Exact
        value: "/my-path/"
    backendRefs:
    - name: <your-backend>
      port: 8000
```

```text
迁移后的行为：
├─ /my-path → 404 Not Found（无自动重定向）
├─ /my-path/ → 200 OK
└─ 生产事故：依赖重定向的客户端收到 404
```

### 正确迁移方案

**显式配置重定向：**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: trailing-slash-route
spec:
  hostnames:
  - trailing-slash.example.com
  parentRefs:
  - name: <your-gateway>
  rules:
  # 规则 1：处理无尾部斜杠的请求，重定向
  - matches:
    - path:
        type: Exact
        value: "/my-path"
    filters:
    - type: RequestRedirect
      requestRedirect:
        statusCode: 301
        path:
          type: ReplaceFullPath
          replaceFullPath: /my-path/
  # 规则 2：处理有尾部斜杠的请求，正常路由
  - matches:
    - path:
        type: Exact
        value: "/my-path/"
    backendRefs:
    - name: <your-backend>
      port: 8000
```

### 检查清单

```text
迁移前检查：
□ 识别所有依赖尾部斜杠重定向的客户端
□ 测试现有 Ingress 的重定向行为
□ 决定是否需要保留重定向（有些客户端可能已修复）
□ 在 Gateway API 中显式配置 RequestRedirect Filter
□ 验证重定向后的行为（包括 HTTPS 重定向场景）
```

## ⑥ 陷阱五：URL 标准化

### 问题场景

Ingress-NGINX 在匹配 Ingress 规则之前会对 URL 进行标准化（URL Normalization）。

URL 标准化遵循 [RFC 3986 Section 6.2](https://datatracker.ietf.org/doc/html/rfc3986#section-6.2)，包括：

```text
标准化操作示例：
├─ 移除单点路径段：/my/./path → /my/path
├─ 处理双点路径段：/my/../path → /path
├─ 去重连续斜杠：/my//path → /my/path
└─ 解码编码字符：/my%2Fpath → /my/path（某些情况）
```

### 意外行为

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-normalization-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: path-normalization.example.com
    http:
      paths:
      - path: "/uuid"
        pathType: Exact
        backend:
          service:
            name: httpbin
            port:
              number: 8000
```

```text
测试结果：
├─ /uuid → 200 OK（精确匹配）
├─ /u/./uid → 200 OK（标准化后为 /uuid）
├─ /x/../uuid → 200 OK（标准化后为 /uuid）
├─ /uuid// → 200 OK（标准化后为 /uuid/，然后尾部斜杠重定向）
└─ 所有标准化后等于 /uuid 的路径都能匹配
```

### Gateway API 迁移陷阱

不同的 Gateway API 实现对 URL 标准化的处理可能不同：

```text
潜在问题：
├─ 某些实现可能不执行相同的标准化
├─ 依赖标准化行为的路径可能失效
├─ 安全策略可能依赖标准化（如 WAF 规则）
└─ 需要验证具体实现的行为
```

### 正确迁移方案

**方案 1：在应用层处理标准化**

```yaml
# 在应用中统一处理路径标准化，不依赖网关行为
# 优点：行为一致，不依赖具体实现
# 缺点：需要修改应用代码
```

**方案 2：使用多个匹配规则**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: normalization-route
spec:
  hostnames:
  - path-normalization.example.com
  parentRefs:
  - name: <your-gateway>
  rules:
  - matches:
    - path:
        type: Exact
        value: "/uuid"
    - path:
        type: RegularExpression
        value: "/u/./uuid"
    - path:
        type: RegularExpression
        value: "/x/../uuid"
    backendRefs:
    - name: httpbin
      port: 8000
```

**方案 3：在网关前添加标准化中间件**

```text
架构：
客户端 → 标准化中间件 → Gateway API → 后端服务

优点：
├─ 统一标准化逻辑
├─ 可审计、可测试
└─ 不依赖具体 Gateway 实现
```

### 检查清单

```text
迁移前检查：
□ 识别依赖 URL 标准化的路径匹配规则
□ 测试各种标准化场景（./、../、//、编码字符）
□ 评估安全影响（标准化可能绕过某些安全检查）
□ 选择目标 Gateway 实现并测试其标准化行为
□ 决定是否需要显式标准化中间件
```

## ⑦ 迁移策略与最佳实践

### 迁移流程

```text
阶段 1：发现与评估（1-2 周）
├─ 扫描所有 Ingress 资源
├─ 识别使用的注解和特性
├─ 测试实际行为（可能和文档不一致）
└─ 记录依赖的隐藏行为

阶段 2：规划与设计（1-2 周）
├─ 选择 Gateway API 实现（Istio、Envoy Gateway、Kgateway 等）
├─ 设计迁移方案（保留行为 vs 修复问题）
├─ 制定回滚计划
└─ 准备监控和告警

阶段 3：预发布验证（2-4 周）
├─ 在预发布环境部署 Gateway API
├─ 并行运行 Ingress-NGINX 和 Gateway API
├─ 对比行为差异
├─ 修复发现的问题

阶段 4：生产迁移（2-4 周）
├─ 分批次迁移（按服务/团队）
├─ 密切监控错误率和延迟
├─ 快速回滚能力
└─ 用户沟通和文档更新

阶段 5：清理与优化（1-2 周）
├─ 移除 Ingress-NGINX 资源
├─ 清理不再需要的配置
├─ 更新文档和 Runbook
└─ 经验总结
```

### 工具推荐

```text
发现与评估：
├─ kubectl grep -r "nginx.ingress.kubernetes.io" --all-namespaces
├─ kubent (Kubernetes No Trouble) - 废弃 API 检测
└─ 自定义脚本扫描 Ingress 注解

迁移自动化：
├─ ingress2gateway - 官方转换工具
├─ 自定义转换脚本（处理边缘情况）
└─ GitOps 工作流（ArgoCD、Flux）

测试验证：
├─ 自动化 E2E 测试
├─ 流量镜像（将生产流量复制到 Gateway API）
└─ 渐进式发布（Canary、Blue-Green）
```

### 常见注解映射表

| Ingress-NGINX 注解 | Gateway API 等价物 | 注意事项 |
|-------------------|-------------------|----------|
| `nginx.ingress.kubernetes.io/use-regex` | `path.type: RegularExpression` | 需显式指定大小写和前缀语义 |
| `nginx.ingress.kubernetes.io/rewrite-target` | `filters[].urlRewrite.path` | 不会隐式启用正则 |
| `nginx.ingress.kubernetes.io/ssl-redirect` | `filters[].requestRedirect` | 需显式配置 |
| `nginx.ingress.kubernetes.io/cors-allow-origin` | 依赖具体实现 | 某些实现支持 CORS Policy |
| `nginx.ingress.kubernetes.io/rate-limit` | 依赖具体实现 | 可能需要额外插件 |

## ⑧ 总结与行动建议

### 核心教训

```text
1. 隐藏行为是最大风险
   └─ Ingress-NGINX 的很多行为没有明确文档化
   └─ 直接转换会导致生产事故

2. 测试是关键
   └─ 不要假设行为一致
   └─ 在预发布环境全面测试

3. 显式优于隐式
   └─ Gateway API 要求显式配置
   └─ 这是优点，不是缺点（行为可预测）

4. 迁移是修复技术债务的机会
   └─ 修复被掩盖的拼写错误
   └─ 清理不再需要的配置
   └─ 标准化路径匹配逻辑
```

### 立即行动

```text
本周：
□ 运行 kubent 或其他工具扫描废弃 API
□ 列出所有使用 Ingress-NGINX 的命名空间
□ 识别关键业务服务（优先迁移）

本月：
□ 选择 Gateway API 实现并搭建测试环境
□ 开始迁移非关键服务（积累经验）
□ 更新团队文档和培训材料

本季度：
□ 完成所有服务迁移
□ 移除 Ingress-NGINX
□ 建立 Gateway API 运维规范
```

### 参考资源

- [Ingress-NGINX 退役公告](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Gateway API 官方文档](https://gateway-api.sigs.k8s.io/)
- [ingress2gateway 转换工具](https://github.com/kubernetes-sigs/ingress2gateway)
- [Gateway API 实现对比](https://gateway-api.sigs.k8s.io/implementations/)

---

**参考资料：**

- [Kubernetes Blog: Before You Migrate: Five Surprising Ingress-NGINX Behaviors](https://kubernetes.io/blog/2026/02/27/ingress-nginx-before-you-migrate/)
- [Ingress-NGINX GitHub](https://github.com/kubernetes/ingress-nginx)
- [Gateway API Specification](https://gateway-api.sigs.k8s.io/reference/spec/)
- [RFC 3986: Uniform Resource Identifier (URI): Generic Syntax](https://datatracker.ietf.org/doc/html/rfc3986)
