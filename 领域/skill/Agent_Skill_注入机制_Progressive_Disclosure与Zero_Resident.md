# Agent Skill 注入机制：Progressive Disclosure 与 Zero Resident Skill

## 1. 背景

随着 Agent 中 Skill 数量增加，如何让模型使用合适的能力成为一个核心问题。

最简单的方式是将所有 Skill 全部放入 System Prompt，但会导致：

- Context 长度增加
- Token 消耗增加
- KV Cache 压力增大
- 无关能力干扰模型推理

因此出现了两种主要的 Skill 注入方式：

1. **LLM 驱动的 Progressive Disclosure（渐进式披露）**
2. **Runtime/Harness 驱动的 Zero Resident Skill（零常驻注入）**

---

# 2. 模式一：LLM 驱动的 Skill 注入（Progressive Disclosure）

## 核心思想

让 LLM 负责 Skill 的发现和选择。

模型先知道有哪些 Skill，需要时再加载完整 Skill 内容。

---

## 工作流程

```text
启动 Agent

    |
    v

Skill Loader 扫描 skills/

    |
    v

读取 SKILL.md 元信息

(name + description)

    |
    v

生成 Skill Catalog

    |
    v

注入 System Prompt

    |
    v

用户请求

    |
    v

LLM 判断需要某个 Skill

    |
    v

调用 load_skill()

    |
    v

Skill 全文作为 tool result 加入上下文
```

---

## 示例

Skill：

```
skills/

 docker/
    SKILL.md

 pdf/
    SKILL.md
```

SKILL.md：

```yaml
---
name: docker
description: Manage docker containers
---
```

启动后：

```
Available Skills:

docker:
Manage docker containers

pdf:
Process PDF documents
```

用户：

```
帮我部署这个服务
```

模型：

```
需要 docker skill
```

调用：

```
load_skill("docker")
```

返回：

```
Docker Skill 完整内容
```

作为 tool message 加入上下文。

---

# 3. Progressive Disclosure 优缺点

## 优点

### 1. 模型具备自主发现能力

LLM 可以根据自然语言理解选择能力。

例如：

```
优化网站性能
```

可能选择：

- frontend skill
- database skill
- nginx skill
- security skill

---

### 2. Skill 扩展简单

新增：

```
skills/

 terraform/
    SKILL.md
```

重新扫描即可。

无需修改 Agent Runtime。

---

### 3. 泛化能力强

适合：

- 通用 Agent
- 开放任务
- 多 Skill 组合任务

---

## 缺点

### 1. Skill Catalog 会占用上下文

即使只加载：

```
name + description
```

大量 Skill 仍然会增加：

- Prompt Token
- KV Cache 前缀长度

---

### 2. Skill 选择依赖模型

模型可能：

- 不知道某 Skill 存在
- 选择错误 Skill
- 忽略可用能力

---

# 4. 模式二：Runtime 驱动的 Zero Resident Skill

## 核心思想

Skill 不进入模型上下文。

由 Agent Runtime / Harness 根据事件动态决定何时注入 Skill。

---

## 工作流程

```text
启动 Agent

    |
    v

Runtime 注册 Skill

    |
    v

Skill 不进入 LLM Context

    |
    v

用户请求

    |
    v

Harness Router / Hook 判断

    |
    v

动态注入 Skill

    |
    v

LLM 继续执行
```

---

## 与传统方式区别

传统：

```
LLM:

我看到 docker skill

↓

我要加载 docker
```

Zero Resident：

```
Runtime:

检测当前任务

↓

发现需要 docker

↓

注入 docker skill

↓

LLM 执行
```

核心变化：

> Skill 选择权从 LLM 转移到 Runtime。

---

# 5. Zero Resident 如何发现 Skill？

零常驻并不是没有 Skill 信息。

Skill 信息存在 Runtime，而不是 Context。

---

## 5.1 Tool Binding

工具和 Skill 绑定。

例如：

```
browser.open()

        |
        v

browser skill
```

调用工具时：

```
tool call

↓

hook

↓

inject skill
```

---

## 5.2 Rule Router

根据：

- 用户关键词
- 文件类型
- 工具调用
- 任务类型

匹配 Skill。

例如：

```
.py 文件

↓

python skill
```

---

## 5.3 Skill Metadata Registry

Skill 自带元数据：

```yaml
name: terraform

trigger:
  keywords:
    - cloud
    - infrastructure
```

Runtime 自动扫描：

```
Skill Registry

        |
        v

Router Index
```

新增 Skill 不需要修改 Harness。

---

## 5.4 Embedding Router

将 Skill 描述建立向量索引：

```
Skill Database

docker
pdf
latex

        |
        v

Embedding Search

        |
        v

匹配 Skill
```

---

# 6. Zero Resident 优缺点

## 优点

### 1. Context 零污染

模型启动时：

```
没有 Skill Catalog
```

减少：

- Token 消耗
- KV Cache 压力

---

### 2. 支持大量 Skill

10000 个 Skill：

不会全部进入 System Prompt。

---

### 3. 更像插件系统

类似 IDE：

不是启动时加载所有插件，而是在需要时加载。

---

## 缺点

### 1. Runtime 复杂度提高

原来：

```
LLM 判断 Skill
```

现在：

```
Harness 判断 Skill
```

需要设计：

- Router
- Hook
- Trigger
- Metadata

---

### 2. 泛化能力可能下降

如果 Runtime 没识别需求：

```
用户任务

↓

没有匹配 Skill

↓

Skill 不会加载
```

---

# 7. 两种模式对比

| |Progressive Disclosure|Zero Resident|
|-|-|-|
|Skill Catalog|进入上下文|不进入上下文|
|Skill Body|按需加载|按需加载|
|选择者|LLM|Runtime|
|上下文消耗|较高|最低|
|模型自主性|高|较低|
|工程复杂度|低|高|
|适合规模|中小规模 Skill|大规模 Skill|

---

# 8. 实际工程趋势：混合模式

当前更合理的方案：

```
                User Task

                    |
                    v

          Runtime 初筛

                    |
        +-----------+-----------+

        明确匹配             不确定

           |                   |

       自动注入              LLM 查看 Skill Catalog

```

即：

- Runtime 处理高确定性场景
- LLM 处理复杂语义场景

---

# 9. 总结

Skill 注入的发展：

## 第一代：全部加载

```
Skill 全文

↓

System Prompt
```

问题：

Context 爆炸。

---

## 第二代：渐进式披露

```
Skill 摘要

↓

LLM 选择

↓

加载全文
```

问题：

Skill Catalog 随规模增长。

---

## 第三代：Runtime Skill

```
Runtime Router

↓

动态注入

↓

LLM 执行
```

问题：

需要更强 Harness 编排能力。

---

最终区别：

> Progressive Disclosure：
> 让模型决定“我需要什么能力”。

> Zero Resident：
> 让系统决定“什么时候给模型什么能力”。

两者解决的是同一个问题：

**如何在有限上下文中管理越来越多的 Agent 能力。**
