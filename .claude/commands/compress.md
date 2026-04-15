你是 Compression 项目的增量压缩管理器。执行以下完整流程。

**模式选择**：
- 输入参数含 `--mode=zip` → MODE = `zip`，跳过询问直接开始
- 输入参数含 `--mode=events` → MODE = `events`，跳过询问直接开始
- **无参数时**：向用户展示以下提示，等待回复：

---
选择压缩模式（直接回车 = Events）：

**1. Events（默认）** — 提取关键事件，生成时间线 + 月度汇总
**2. Zip** — 高保真叙事压缩，保留对话原有语感

---

根据回复确定 MODE：输入 `2`、`zip`、`z` → `zip`；其他一律 `events`。  
将 MODE 记下，后续步骤全程使用。

**所有依赖都内联在本文件中。不读取任何外部 prompt 文件。辅助文件不存在时自动创建。**

---

## 第零步：确保项目基础设施存在

### 0.1 目录

```bash
mkdir -p output/json output/zip output/batches output/batch_results input/test
```

### 0.2 `state.json`

如果不存在，创建：

```json
{"schema_version": 2, "last_run": "", "chunk_cache": {}, "files": {}}
```

### 0.3 `assemble.py`

如果不存在，用 Write 工具创建，完整内容如下：

```python
import json
import os
import re
import hashlib
import sys
from collections import defaultdict
from datetime import datetime

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
INPUT_DIR = os.path.join(BASE_DIR, "input", "test")
OUTPUT_DIR = os.path.join(BASE_DIR, "output")
JSON_DIR = os.path.join(OUTPUT_DIR, "json")
ZIP_DIR = os.path.join(OUTPUT_DIR, "zip")
BATCH_DIR = os.path.join(OUTPUT_DIR, "batches")
RESULTS_DIR = os.path.join(OUTPUT_DIR, "batch_results")
STATE_FILE = os.path.join(BASE_DIR, "state.json")
PENDING_FILE = os.path.join(OUTPUT_DIR, "pending_batches.json")

CHUNK_SIZE = 1500
BATCH_SIZE = 5


def compute_hash(data: bytes) -> str:
    return "sha256:" + hashlib.sha256(data).hexdigest()


def file_hash(path: str) -> str:
    return compute_hash(open(path, "rb").read())


def chunk_text(text: str) -> list:
    chunks = []
    for i in range(0, len(text), CHUNK_SIZE):
        t = text[i:i + CHUNK_SIZE]
        chunks.append({"offset": i, "text": t, "hash": compute_hash(t.encode("utf-8"))})
    return chunks


def load_state() -> dict:
    if os.path.exists(STATE_FILE):
        return json.load(open(STATE_FILE, encoding="utf-8"))
    return {"schema_version": 2, "last_run": "", "chunk_cache": {}, "files": {}}


def save_state(state: dict):
    state["last_run"] = datetime.now().isoformat(timespec="seconds")
    json.dump(state, open(STATE_FILE, "w", encoding="utf-8"), ensure_ascii=False, indent=2)


def scan_input() -> dict:
    result = {}
    if not os.path.isdir(INPUT_DIR):
        return result
    for f in sorted(os.listdir(INPUT_DIR)):
        if f.startswith("."):
            continue
        fp = os.path.join(INPUT_DIR, f)
        if os.path.isfile(fp):
            result[f] = file_hash(fp)
    return result


def prepare(mode: str = "zip"):
    for d in (BATCH_DIR, RESULTS_DIR):
        os.makedirs(d, exist_ok=True)

    state = load_state()
    state.setdefault("chunk_cache", {})
    cache = state["chunk_cache"]
    current = scan_input()
    old_files = state.get("files", {})
    batches, unchanged, pending_summary, deleted = [], [], [], []

    for fname, fhash in current.items():
        old = old_files.get(fname, {})
        if old.get("file_hash") == fhash:
            unchanged.append(fname)
            continue

        with open(os.path.join(INPUT_DIR, fname), encoding="utf-8", errors="replace") as f:
            text = f.read()

        chunks = chunk_text(text)
        pending = [c for c in chunks if c["hash"] not in cache]

        if not pending:
            old["file_hash"] = fhash
            old["chunk_hashes"] = [c["hash"] for c in chunks]
            state["files"][fname] = old
            unchanged.append(fname)
            continue

        pending_summary.append({"file": fname, "pending": len(pending), "total": len(chunks)})
        state["files"][fname] = {
            "file_hash": fhash,
            "chunk_hashes": [c["hash"] for c in chunks],
            "status": "pending",
        }

        slug = re.sub(r"[^\w.-]", "_", fname)
        for bi, start in enumerate(range(0, len(pending), BATCH_SIZE)):
            bc = pending[start:start + BATCH_SIZE]
            lines = []
            for c in bc:
                lines.append(f"=== CHUNK {c['hash']} ===")
                lines.append(c["text"])
                lines.append("")
            batch_id = f"{slug}_batch_{bi:03d}"
            tf = os.path.join(BATCH_DIR, batch_id + ".txt")
            rf = os.path.join(RESULTS_DIR, batch_id + ".txt")
            open(tf, "w", encoding="utf-8").write("\n".join(lines))
            batches.append({
                "id": batch_id,
                "file": fname,
                "batch_num": bi,
                "text_file": tf,
                "result_file": rf,
                "chunk_hashes": [c["hash"] for c in bc],
            })

    for fname in list(old_files.keys()):
        if fname not in current:
            deleted.append(fname)

    json.dump({"mode": mode, "batches": batches},
              open(PENDING_FILE, "w", encoding="utf-8"),
              ensure_ascii=False, indent=2)
    save_state(state)

    if unchanged:
        print(f"已缓存（跳过）：{len(unchanged)} 个文件")
    if pending_summary:
        print(f"需要处理：{len(pending_summary)} 个文件，{len(batches)} 个 batch")
        for p in pending_summary:
            print(f"  - {p['file']} ({p['pending']}/{p['total']} chunks pending)")
    if deleted:
        print(f"已移除：{len(deleted)} 个文件")
    if not batches:
        print("无待处理内容，直接组装。")


def parse_events_text(text: str) -> list:
    events = []
    for block in re.split(r"={3,}EVENT={3,}", text):
        block = block.strip()
        if not block or re.match(r"=+\s*(END|CHUNK)", block):
            continue
        ev = {}
        for line in block.splitlines():
            line = line.strip()
            if line.startswith("TIME:"):
                ev["time"] = line[5:].strip()
            elif line.startswith("TITLE:"):
                ev["title"] = line[6:].strip()
            elif line.startswith("FACTS:"):
                ev["facts"] = [f.strip() for f in line[6:].split("|") if f.strip()]
            elif line.startswith("ACTION:"):
                ev["action"] = line[7:].strip()
            elif line.startswith("PSYCH:"):
                ev["psych"] = line[6:].strip()
            elif line.startswith("CONCLUSION:"):
                ev["conclusion"] = line[11:].strip()
        if ev.get("time") and ev.get("title"):
            for k in ("action", "psych", "conclusion"):
                ev.setdefault(k, "")
            ev.setdefault("facts", [])
            events.append(ev)
    return events


def parse_chunk_results(text: str, mode: str) -> dict:
    result = {}
    parts = re.split(r"={3}\s*CHUNK\s+(sha256:\w+)\s*={3}", text)
    i = 1
    while i + 1 < len(parts):
        chash = parts[i].strip()
        body = re.sub(r"={3,}END={3,}", "", parts[i + 1]).strip()
        if mode == "events":
            result[chash] = {"type": "events", "events": parse_events_text(body)}
        else:
            result[chash] = {"type": "zip", "text": body}
        i += 2
    return result


def merge_results() -> dict:
    if not os.path.exists(PENDING_FILE):
        print("没有 pending_batches.json，跳过。")
        return {}

    pb = json.load(open(PENDING_FILE, encoding="utf-8"))
    mode = pb.get("mode", "zip")
    state = load_state()
    cache = state.setdefault("chunk_cache", {})
    failed, merged_files = [], set()

    for batch in pb["batches"]:
        rf = batch["result_file"]
        if not os.path.exists(rf):
            failed.append(batch["id"])
            continue
        result_text = open(rf, encoding="utf-8").read().strip()
        if not result_text:
            failed.append(batch["id"])
            continue

        chunk_results = parse_chunk_results(result_text, mode)
        if chunk_results:
            cache.update(chunk_results)
        else:
            # Fallback: no chunk markers, assign to first hash
            if mode == "events":
                events = parse_events_text(result_text)
                for i, chash in enumerate(batch["chunk_hashes"]):
                    cache[chash] = {"type": "events", "events": events if i == 0 else []}
            else:
                for i, chash in enumerate(batch["chunk_hashes"]):
                    cache[chash] = {"type": "zip", "text": result_text if i == 0 else ""}

        merged_files.add(batch["file"])

    os.makedirs(JSON_DIR, exist_ok=True)
    os.makedirs(ZIP_DIR, exist_ok=True)
    processed = []

    for fname, info in state["files"].items():
        if fname not in merged_files:
            continue
        chunk_hashes = info.get("chunk_hashes", [])
        now = datetime.now().isoformat(timespec="seconds")

        if mode == "events":
            events = []
            for chash in chunk_hashes:
                events.extend(cache.get(chash, {}).get("events", []))
            base = os.path.splitext(fname)[0]
            out = os.path.join(JSON_DIR, base + ".json")
            json.dump({"source": fname, "events": events},
                      open(out, "w", encoding="utf-8"),
                      ensure_ascii=False, indent=2)
            info.update({"status": "done", "processed_at": now,
                         "output": f"output/json/{base}.json",
                         "event_count": len(events)})
            processed.append(f"{fname}: {len(events)} 个事件")
        else:
            texts = [cache[h]["text"] for h in chunk_hashes
                     if cache.get(h, {}).get("text")]
            base = os.path.splitext(fname)[0]
            out = os.path.join(ZIP_DIR, base + ".txt")
            open(out, "w", encoding="utf-8").write("\n\n".join(texts))
            info.update({"status": "done", "processed_at": now,
                         "output": f"output/zip/{base}.txt"})
            processed.append(f"{fname}: 已压缩")

    save_state(state)
    if failed:
        print(f"失败的 batch（需重跑）：{len(failed)} 个 → {', '.join(failed)}")
    if processed:
        print(f"已合并：{'; '.join(processed)}")
    return {"failed": failed, "processed": processed}


_VAGUE_TIME_RE = re.compile(
    r"未标明|不明|具体时间|^\d{1,2}月-\d{1,2}月$|^早期$|^初期$|^后期$|^某|^不详$",
    re.UNICODE,
)


def find_unclear() -> list:
    if not os.path.isdir(JSON_DIR):
        print("没有 JSON 数据，跳过补全扫描。")
        return []
    unclear = []
    for fname in sorted(os.listdir(JSON_DIR)):
        if not fname.endswith(".json"):
            continue
        data = json.load(open(os.path.join(JSON_DIR, fname), encoding="utf-8"))
        for i, ev in enumerate(data.get("events", [])):
            issues = []
            t = ev.get("time", "").strip()
            if not t or _VAGUE_TIME_RE.search(t):
                issues.append(f"时间：{t or '（空）'}")
            for field, label in [("action", "行动"), ("psych", "心理"), ("conclusion", "结论")]:
                if not ev.get(field, "").strip():
                    issues.append(f"{label}：（空）")
            if not ev.get("facts"):
                issues.append("事实：（空）")
            if issues:
                unclear.append({
                    "json_file": fname,
                    "source": data.get("source", fname),
                    "event_index": i,
                    "title": ev.get("title", ""),
                    "current_time": t or "（空）",
                    "issues": issues,
                })
    out_path = os.path.join(OUTPUT_DIR, "unclear_events.json")
    json.dump(unclear, open(out_path, "w", encoding="utf-8"), ensure_ascii=False, indent=2)
    if unclear:
        print(f"找到 {len(unclear)} 个待补全事件 → {out_path}")
    else:
        print("所有事件信息完整，无需补全。")
    return unclear


def patch_event(json_file: str, event_index: int, updates: dict):
    path = os.path.join(JSON_DIR, json_file)
    data = json.load(open(path, encoding="utf-8"))
    ev = data["events"][event_index]
    if "time" in updates and updates["time"]:
        ev["time"] = updates["time"]
    if "facts_append" in updates:
        extra = updates["facts_append"]
        if isinstance(extra, str):
            extra = [extra]
        ev.setdefault("facts", [])
        ev["facts"].extend(extra)
    for k in ("action", "psych", "conclusion"):
        if k in updates and updates[k]:
            ev[k] = updates[k]
    json.dump(data, open(path, "w", encoding="utf-8"), ensure_ascii=False, indent=2)


def _sort_key(time_str: str):
    t = str(time_str).strip()

    # YYYY-MM-DD (full date)
    if m := re.search(r"(20\d{2})-(\d{1,2})-(\d{1,2})", t):
        return int(m.group(1)), int(m.group(2)), int(m.group(3)), t

    # YYYY-MM (year + month, e.g. "2025-12", "2025-12 起")
    if m := re.search(r"(20\d{2})-(\d{1,2})(?!\d)", t):
        return int(m.group(1)), int(m.group(2)), 99, t

    # YYYY only (e.g. "2024-10 至 2025-12" → use first year)
    if m := re.search(r"(20\d{2})", t):
        return int(m.group(1)), 99, 99, t

    # MM-DD (no year, e.g. "03-26", "03-27 22:53 - 03-30")
    # Must start at string start or after whitespace to avoid matching HH:MM:SS fragments
    if m := re.search(r"(?:^|\s)(\d{1,2})-(\d{1,2})", t):
        mo = int(m.group(1))
        if 1 <= mo <= 12:
            # Data spans 2025-07 ~ 2026-06: months ≤6 → 2026, months ≥7 → 2025
            year = 2026 if mo <= 6 else 2025
            return year, mo, int(m.group(2)), t

    # Pure times or unrecognised → sort to end
    return 9999, 99, 99, t


def assemble(mode: str = "zip"):
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    if mode == "zip":
        _assemble_zip()
    else:
        _assemble_events()


def _assemble_zip():
    if not os.path.isdir(ZIP_DIR):
        print("没有 zip 输出，跳过。")
        return
    parts = []
    for f in sorted(os.listdir(ZIP_DIR)):
        if not f.endswith(".txt"):
            continue
        text = open(os.path.join(ZIP_DIR, f), encoding="utf-8").read().strip()
        if text:
            parts.append(f"## {f[:-4]}\n\n{text}")
    out = "# 语义压缩报告\n\n" + "\n\n---\n\n".join(parts) + "\n"
    open(os.path.join(OUTPUT_DIR, "output.md"), "w", encoding="utf-8").write(out)
    print(f"zip 模式组装完成：{len(parts)} 个源文件 → output/output.md")


def _assemble_events():
    if not os.path.isdir(JSON_DIR):
        print("没有 JSON 数据，跳过。")
        return
    all_data = []
    for f in sorted(os.listdir(JSON_DIR)):
        if not f.endswith(".json"):
            continue
        data = json.load(open(os.path.join(JSON_DIR, f), encoding="utf-8"))
        for ev in data.get("events", []):
            for k in ("time", "title", "action", "psych", "conclusion"):
                ev.setdefault(k, "")
            if not isinstance(ev.get("facts"), list):
                ev["facts"] = [str(ev.get("facts", ""))]
        all_data.append(data)
    if not all_data:
        print("没有事件数据，跳过。")
        return

    def build_timeline():
        evs = [(ev["time"], d["source"], ev)
               for d in all_data for ev in d.get("events", [])]
        evs.sort(key=lambda x: _sort_key(x[0]))
        lines = ["# 时间线\n"]
        for ts, src, ev in evs:
            lines.append(f"- **{ts}** [{src}] {ev['title']}")
        return "\n".join(lines) + "\n"

    def build_events():
        lines = ["# 重要事件\n"]
        for d in all_data:
            lines.append(f"## {d['source']}\n")
            for ev in d.get("events", []):
                lines.append(f"### {ev['time']} — {ev['title']}")
                lines.append("- **事实**：")
                for fact in ev.get("facts", []):
                    lines.append(f"  - {fact}")
                lines.append(f"- **行动**：{ev['action']}")
                lines.append(f"- **心理**：{ev['psych']}")
                lines.append(f"- **结论**：{ev['conclusion']}\n")
        return "\n".join(lines)

    def build_weekly():
        buckets = defaultdict(list)
        for d in all_data:
            for ev in d.get("events", []):
                t = ev["time"]
                if m := re.search(r"(20\d{2})-(\d{1,2})", t):
                    bucket = f"{m.group(1)}年{int(m.group(2))}月"
                elif m := re.search(r"(\d{1,2})-\d", t):
                    mo = int(m.group(1))
                    bucket = f"{'2026' if mo <= 4 else '2025'}年{mo}月"
                else:
                    bucket = "其他"
                buckets[bucket].append(ev)

        def bkey(n):
            if m := re.search(r"(\d{4})年(\d{1,2})月", n):
                return int(m.group(1)), int(m.group(2))
            return 9999, 99

        lines = ["# 月度汇总\n"]
        for b in sorted(buckets, key=bkey):
            titles = [ev["title"] for ev in buckets[b]]
            lines.append(f"## {b}\n共 {len(titles)} 个事件：{'；'.join(titles)}。\n")
        return "\n".join(lines)

    tl, ev, wk = build_timeline(), build_events(), build_weekly()
    for name, content in [("timeline.md", tl), ("events.md", ev), ("weekly.md", wk)]:
        open(os.path.join(OUTPUT_DIR, name), "w", encoding="utf-8").write(content)
    out = "# 对话压缩报告\n\n---\n\n" + tl + "\n---\n\n" + ev + "\n---\n\n" + wk
    open(os.path.join(OUTPUT_DIR, "output.md"), "w", encoding="utf-8").write(out)
    total = sum(len(d.get("events", [])) for d in all_data)
    print(f"events 模式组装完成：{len(all_data)} 个源文件，{total} 个事件 → output/output.md")


def legacy_update_state():
    state = load_state()
    current = scan_input()
    n = 0
    for fname, fhash in current.items():
        base = os.path.splitext(fname)[0]
        jf = os.path.join(JSON_DIR, base + ".json")
        if os.path.exists(jf):
            data = json.load(open(jf, encoding="utf-8"))
            state["files"][fname] = {
                "file_hash": fhash,
                "status": "done",
                "processed_at": datetime.now().isoformat(timespec="seconds"),
                "output": f"output/json/{base}.json",
                "event_count": len(data.get("events", [])),
            }
            n += 1
    save_state(state)
    print(f"state.json 已更新：{n} 个文件")


def main():
    cmd = sys.argv[1] if len(sys.argv) > 1 else "run"
    mode = next((a.split("=")[1] for a in sys.argv if a.startswith("--mode=")), "events")

    if cmd == "prepare":
        prepare(mode)
    elif cmd == "merge-results":
        merge_results()
    elif cmd == "assemble":
        assemble(mode)
    elif cmd == "update-state":
        legacy_update_state()
    elif cmd == "find-unclear":
        find_unclear()
    elif cmd == "patch-event":
        json_file = sys.argv[2]
        event_index = int(sys.argv[3])
        updates = json.loads(sys.argv[4])
        patch_event(json_file, event_index, updates)
        print(f"已更新 {json_file} 第 {event_index} 个事件")
    elif cmd == "status":
        state = load_state()
        current = scan_input()
        pending = [f for f, h in current.items()
                   if state.get("files", {}).get(f, {}).get("file_hash") != h]
        if pending:
            print(f"需要处理：{len(pending)} 个文件")
            for f in pending:
                print(f"  - {f}")
        else:
            print(f"所有 {len(current)} 个文件均已缓存，无待处理内容。")
    else:
        prepare(mode)
        merge_results()
        assemble(mode)


if __name__ == "__main__":
    main()
```

如果 `assemble.py` 已存在且内容一致则跳过，不同则完整覆盖。

---

## 第一步：扫描与差异对比

运行：

```bash
python3 assemble.py prepare --mode=MODE
```

（将 `MODE` 替换为实际模式：`zip` 或 `events`）

- 无 pending → 直接跳到第四步。
- 有 pending（输出 "需要处理：N 个文件，M 个 batch"）→ 继续第二步。

---

## 第二步：并行压缩 pending batches

读取 `output/pending_batches.json`，获取 `batches` 数组。

对每个 batch，启动一个后台 Agent（general-purpose，`run_in_background: true`），根据 MODE 使用以下 prompt：

将 `{{TEXT_FILE}}` 替换为 `batch.text_file`（绝对路径）。  
将 `{{RESULT_FILE}}` 替换为 `batch.result_file`（绝对路径）。

---

### ZIP 模式 Agent prompt

```
你是一名文本压缩员。

使用 Read 工具读取文件：{{TEXT_FILE}}

该文件包含若干文本片段，每段以 `=== CHUNK sha256:... ===` 开头标记。

对每个 CHUNK 片段进行高保真语义压缩：
- 保留所有实质信息（具体细节、情绪、因果链条、具体数据）
- 删除重复表述、纯问候语、无信息量的回应
- 目标压缩率 30-50%
- 不添加任何标题、编号、格式标记

输出格式（严格遵守，每个 chunk 对应一个输出块）：

=== CHUNK sha256:原文中的完整hash值 ===
该 chunk 的压缩文本

=== CHUNK sha256:原文中的完整hash值 ===
该 chunk 的压缩文本

===END===

注意：原样复制输入文件中的每个 `=== CHUNK sha256:... ===` 标记，不要修改 hash 值。

使用 Write 工具将上述结果写入：{{RESULT_FILE}}
```

---

### EVENTS 模式 Agent prompt

```
你是一名事件提取员。

使用 Read 工具读取文件：{{TEXT_FILE}}

该文件包含若干文本片段，每段以 `=== CHUNK sha256:... ===` 开头标记。

对每个 CHUNK 片段，提取其中的关键事件。

输出格式（严格遵守）：

=== CHUNK sha256:原文中的完整hash值 ===
===EVENT===
TIME: MM-DD HH:MM（或日期范围）
TITLE: 一句话描述（第三方视角）
FACTS: 事实1 | 事实2 | 事实3
ACTION: 主角采取的具体行动
PSYCH: 心理转折或动机
CONCLUSION: 核心结论

===EVENT===
[下一个事件，格式同上]

=== CHUNK sha256:原文中的完整hash值 ===
[下一个 chunk 的事件，格式同上]

===END===

语言规则（必须遵守）：
- 严禁将原文黑话、诗意隐喻、内部代号直接填入任何字段
- 必须翻译成普通第三方能读懂的白话中文
- 如某 chunk 无值得提取的事件，仍输出 CHUNK 标记，内容留空

注意：原样复制输入文件中的每个 `=== CHUNK sha256:... ===` 标记，不要修改 hash 值。

使用 Write 工具将上述结果写入：{{RESULT_FILE}}
```

---

所有 Agent 并行启动（`run_in_background: true`）。

---

## 第三步：收集结果并更新状态

等待所有 Agent 完成。运行：

```bash
python3 assemble.py merge-results
```

如有失败的 batch（result_file 不存在或为空）：
- 用更严格的指令重新启动对应 Agent（传入相同的 TEXT_FILE 和 RESULT_FILE）
- 等待完成后再次运行 `merge-results`

---

## 第四步：补全询问（仅 events 模式）

**zip 模式跳过此步骤。**

运行：

```bash
python3 assemble.py find-unclear
```

- 输出"所有事件信息完整" → 跳到第五步。
- 输出"找到 N 个待补全事件" → 继续以下流程。

读取 `output/unclear_events.json`，将清单一次性展示给用户：

---

以下事件有缺失信息，**按编号说一遍就行**，不知道的说"跳过"，可以语音输入：

[按编号列出，每条格式：]
N. 「{title}」（来源：{source}）
   {issues 列表，每行一条，如：时间：2025-12 起（具体？）/ 结论：（空）/ 心理：（空）}

---

**等待用户一次性回复全部内容。**

收到回复后，逐条解析并调用 `patch-event`：

1. 时间支持模糊描述（"去年十月" → `2025-10`，"三月底" → `03-下旬`，"上个月" → 根据当前日期推算）
2. 用户提供的背景/补充信息作为 `facts_append`
3. 用户直接填写的行动/心理/结论直接替换对应字段

```bash
python3 assemble.py patch-event <json_file> <event_index> '<json_updates>'
```

`<json_updates>` 支持字段：`time`、`facts_append`（数组或字符串）、`action`、`psych`、`conclusion`

用户说"跳过"的保持原样。全部更新完成后继续。

---

## 第五步：组装输出

```bash
python3 assemble.py assemble --mode=MODE
```

生成 `output/output.md`。events 模式同时生成 `output/timeline.md`、`output/events.md`、`output/weekly.md`。

---

## 第六步：报告

告诉用户：
- 使用的模式（zip / events）
- 处理了几个文件、几个 batch
- chunk 缓存命中了多少（state.json 里 chunk_cache 的条目数）
- 输出文件路径（`output/output.md`）

不省略任何步骤。出错时报告错误并尝试修复。
