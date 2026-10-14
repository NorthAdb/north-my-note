# JSON Schema 与 Agent Tool Calling

## 1. JSON Schema 是什么？

**JSON Schema 是描述 JSON 数据结构与约束的规范**，例如字段类型、必填项和数值范围。

```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "Path to read."
    },
    "offset": {
      "type": "number",
      "description": "从第几行开始，默认 1。"
    },
    "limit": {
      "type": "number",
      "description": "最多读取多少行，默认 2000。"
    }
  },
  "required": ["file_path"]
}
```

这段 Schema 表示：
- 最外层必须是 `object`。
- `file_path` 是必填的字符串。
- `offset`、`limit` 是可选数字。
- `description` 用来说明字段含义；它本身通常不构成严格的数据校验规则。
- 写着“默认值为 1/2000”的描述不等于 Schema 自动设置了默认值，默认行为通常由工具实现负责。

## 2. Tool Definition 和 JSON Schema 的关系

**Tool Definition（工具定义）是整体；JSON Schema 通常只是其中的参数定义部分。**

```text
Tool Definition
├── name          工具名称，例如 read
├── description   工具用途
└── input_schema  参数结构与约束（JSON Schema）
```

可以这样记：
- `description`：告诉模型工具能做什么。
- `JSON Schema`：告诉模型调用时要传哪些参数、参数是什么类型、哪些必填。
- 工具实现：真正执行读取文件等操作的代码或服务。

因此，Schema 能帮助 LLM 按正确格式构造工具参数，但它不等于完整的工具定义，也不负责执行工具。

## 3. JSON、JSON Schema、Tool Definition 的区别

| 概念 | 作用 |
|---|---|
| JSON | 实际传递的数据，例如 `{"file_path":"PRACTICES.md"}` |
| JSON Schema | 规定数据应满足的结构与约束 |
| Tool Definition | 告诉模型工具的名称、用途和参数格式等信息 |

## 4. Schema 合法，为什么工具仍可能失败？

示例参数：

```json
{
  "file_path": "PRACTICES.md"
}
```

它符合上面的 Schema：必填的 `file_path` 存在且为字符串，其他字段可以省略。

但一次工具调用通常还要经过权限或策略审查，Schema 合法不代表执行一定获准：

```text
LLM 生成 Tool Call
        ↓
参数 / Schema 校验
        ↓
权限、审批或 Auto Review
        ↓
工具执行
        ↓
返回工具结果
```

此前截图中的报错是：

```text
Auto review of tool "read" failed;
its body was not executed: auto-review.
```

这说明工具主体代码**没有执行**，失败发生在 Auto Review 阶段，而不是从报错本身就能判断为 Schema 不合法。具体是哪条策略拒绝了调用，仅凭截图无法确定。

## 5. 核心结论

- **JSON Schema 是参数契约，不是执行授权。**
- **Tool Calling 不等于 Tool Execution。**
- Schema 解决“参数格式对不对”；权限、审批与策略解决“这次调用允不允许执行”。
