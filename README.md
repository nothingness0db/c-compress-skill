# c-compress-skill

把 ChatWise 导出的聊天记录（HTML / MD / TXT）压缩成结构化摘要或语义压缩文本，输出 Markdown 报告。

核心是一个 Claude Code 自定义 skill（`/compress`），只需要一个文件。

---

## 安装

把 `.claude/commands/compress.md` 复制到你项目的 `.claude/commands/` 目录下：

```
your-project/
  .claude/commands/compress.md   ← 唯一需要的文件
  input/test/                    ← 放聊天记录文件
```

其他所有文件（`assemble.py`、`state.json`、`output/`）由 skill 首次运行时自动创建。

---

## 使用

1. 把聊天记录文件放到 `input/test/`（支持 `.html`、`.md`、`.txt`）
2. 在 Claude Code 里运行：

```
/compress              # zip 模式（默认）：高保真语义压缩
/compress --mode=zip   # 同上
/compress --mode=events  # events 模式：提取关键事件，生成时间线
```

3. 查看 `output/output.md`

---

## 两种模式

### zip 模式（默认）

对每段对话做高保真语义压缩：保留所有实质信息，删除重复表述和无信息量的回应，目标压缩率 30-50%。

输出：`output/output.md`（按源文件分节）

### events 模式

从对话中提取关键事件，每个事件包含时间、标题、事实、行动、心理、结论六个字段。

输出：
- `output/timeline.md` — 跨文件时间线
- `output/events.md` — 按源文件分组的事件详情
- `output/weekly.md` — 月度汇总
- `output/output.md` — 以上三份拼装

---

## 增量处理

state.json 存储每个文件的 chunk 级 SHA-256 哈希。再次运行时：

- 文件未变 → 整个跳过
- 文件有新增内容 → 只处理变化的 chunk，已缓存的部分直接复用
- 删除了某文件 → 报告但不影响已有输出

删除 `state.json` 可强制全量重跑。

## 补全询问（events 模式）

events 模式压缩完成后，自动扫描时间字段为空或模糊（"未标明具体时间"、纯月份范围等）的事件，整理成编号清单问你：

```
1. 「主角开始对一位现实中的Coser进行长期单向关注」
   当前：未标明具体时间
   → 大约发生在什么时候？有没有要补充的背景？
```

你可以回答时间（粗略也行，如"去年12月"、"上个月"），也可以顺带补充当时的背景。跳过的事件保持原样。答完后 Claude 直接更新 JSON 并重新生成报告。

数据来源不只是原始文件——你作为当事人补充的信息同样写入归档。

---

## 架构

```
input/test/*.html|md|txt
        │
        ▼ assemble.py prepare --mode=MODE
  按 1500 字切 chunk，对比 chunk_cache
  只有未缓存的 chunk 进入 pending
        │
        ▼ 每 5 个 chunk 一个 batch，并行子 Agent
  Agent 输出 ===CHUNK=== / ===EVENT=== 结构化文本
        │
        ▼ assemble.py merge-results
  解析结果，写入 chunk_cache，重建文件级输出
        │
        ▼ assemble.py find-unclear（仅 events 模式）
  找时间不明确的事件 → 问当事人 → patch-event 写回
        │
        ▼ assemble.py assemble --mode=MODE
  output/output.md
```

**为什么不用 JSON 输出**：模型在中文语境下生成含嵌套引号的 JSON 必出解析错误。当前使用固定标记行（`===EVENT===`、`TIME:`、`FACTS: a | b`）作为输出格式，Python 解析几乎不会失败。

---

## 文件说明

| 文件 | 说明 |
|------|------|
| `.claude/commands/compress.md` | skill 本体，含 assemble.py 全文（唯一需要版本控制的文件） |
| `assemble.py` | 自动生成的 Python 工具脚本 |
| `state.json` | 增量处理状态，含 chunk 级缓存 |
| `output/output.md` | 最终报告 |
| `output/json/` | events 模式的中间 JSON |
| `output/zip/` | zip 模式的压缩文本 |
| `input/test/` | 输入文件（私人数据，不入库） |

---

## 注意事项

- `input/` 目录不入库，里面是私人聊天记录
- 整个流程只依赖 Claude Code，不调任何外部 API
- 不要按行数切分文件；分块使用 `assemble.py` 的 `chunk_text()`（按字符 + 哈希）
- Prompt 必须强调白话翻译，否则 events 模式会直接抄原文黑话
