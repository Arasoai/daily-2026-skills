```markdown
---
name: colleague-skill-creator
description: Generate AI skills that replicate departed colleagues — their technical style, communication patterns, and domain knowledge — from chat logs, docs, and descriptions.
triggers:
  - create a colleague skill
  - generate a colleague persona
  - recreate my coworker as an AI
  - build a skill from chat logs
  - my teammate left and I need to replace them
  - create colleague from feishu messages
  - make an AI version of my coworker
  - colleague skill from documentation
---

# colleague-skill — AI Colleague Skill Generator

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Turn departed colleagues into working AI Skills. Feed in chat logs, docs, emails, or screenshots — get back a Skill that writes code in their style, answers questions in their voice, and knows when they'd deflect blame.

## What It Does

`colleague-skill` is an [AgentSkills](https://agentskills.io)-compatible skill that:

1. **Ingests raw materials** — Feishu/Lark messages, DingTalk docs, emails, PDFs, screenshots, Markdown, or plain text
2. **Extracts two layers** per colleague:
   - **Work Skill** (`work.md`) — tech standards, system knowledge, workflows, domain expertise
   - **Persona** (`persona.md`) — 5-layer personality: hard rules → identity → expression style → decision patterns → interpersonal behavior
3. **Outputs a callable Skill** (`/{slug}`) that runs: `task → Persona judges attitude → Work Skill executes → output in their voice`
4. **Evolves** — append new files, correct via conversation, roll back versions

---

## Installation

### Claude Code (recommended)

```bash
# Project-local (run from git repo root)
mkdir -p .claude/skills
git clone https://github.com/titanwings/colleague-skill .claude/skills/create-colleague

# OR global (available in all projects)
git clone https://github.com/titanwings/colleague-skill ~/.claude/skills/create-colleague
```

### OpenClaw

```bash
git clone https://github.com/titanwings/colleague-skill \
  ~/.openclaw/workspace/skills/create-colleague
```

### Python dependencies (optional — needed for auto-collection)

```bash
cd .claude/skills/create-colleague
pip install -r requirements.txt
```

---

## Configuration

Auto-collection from Feishu/DingTalk requires credentials. Set these environment variables:

```bash
# Feishu / Lark auto-collection
export FEISHU_APP_ID="cli_xxxxxxxxxxxxxxxx"
export FEISHU_APP_SECRET="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# DingTalk auto-collection
export DINGTALK_APP_KEY="dingxxxxxxxxxxxxxxxx"
export DINGTALK_APP_SECRET="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

> **Feishu note**: The App bot must be added to the relevant group chats before collection works.

---

## Basic Usage

### Create a colleague skill

In Claude Code:

```
/create-colleague
```

The interactive intake flow will prompt for:

| Field | Example | Required? |
|-------|---------|-----------|
| Name / slug | `zhang-wei` | Yes |
| Company + level + role | `字节 2-1 后端工程师` | Optional |
| Personality tags | `INTJ, 甩锅高手, 字节范` | Optional |
| Data source | Feishu auto / file upload / paste | Optional |

All fields can be skipped — description alone is enough to generate a basic skill.

### Call a generated skill

```
/zhang-wei                  # Full skill (Persona + Work)
/zhang-wei-work             # Work capability only
/zhang-wei-persona          # Personality only
```

### Management commands

```
/list-colleagues                        # List all generated colleague skills
/colleague-rollback zhang-wei v3        # Roll back to version 3
/delete-colleague zhang-wei             # Delete skill
```

---

## Data Sources

| Source | How to provide |
|--------|---------------|
| Feishu messages + docs | Input colleague name → fully automatic via API |
| DingTalk docs/wiki | Automatic via API; messages require browser fallback |
| PDF documents | Upload file path |
| Images / screenshots | Upload file path (OCR + vision extraction) |
| Feishu JSON export | Upload file path |
| Email `.eml` / `.mbox` | Upload file path |
| Markdown files | Upload file path |
| Plain text / paste | Type or paste directly |

**Quality tip**: Prioritize materials the colleague *wrote themselves* — long-form docs > decision replies > casual messages.

---

## Python Tool Usage

You can also drive the tools directly from Python:

### Feishu auto-collection

```python
import asyncio
from tools.feishu_auto_collector import FeishuAutoCollector

async def collect():
    collector = FeishuAutoCollector(
        app_id=os.environ["FEISHU_APP_ID"],
        app_secret=os.environ["FEISHU_APP_SECRET"],
    )
    # Collect all messages + docs involving "张伟"
    data = await collector.collect_colleague(name="张伟", days_back=180)
    print(f"Collected {len(data['messages'])} messages, {len(data['docs'])} docs")
    return data

asyncio.run(collect())
```

### Email parsing

```python
from tools.email_parser import EmailParser

parser = EmailParser()

# Parse a single .eml file
result = parser.parse_file("handover.eml")
print(result["body"])

# Parse an entire mbox
results = parser.parse_mbox("archive.mbox", sender_filter="zhang.wei@company.com")
for msg in results:
    print(msg["subject"], msg["date"])
```

### Skill file management

```python
from tools.skill_writer import SkillWriter

writer = SkillWriter(base_dir=".claude/skills")

# Write a new colleague skill
writer.write_skill(
    slug="zhang-wei",
    work_content="# Zhang Wei — Work Skill\n\n...",
    persona_content="# Zhang Wei — Persona\n\n...",
)

# Append new material (incremental merge, preserves existing conclusions)
writer.append_material(
    slug="zhang-wei",
    new_content="New chat logs from Q2...",
    source_type="feishu_messages",
)

# Apply a correction from conversation
writer.apply_correction(
    slug="zhang-wei",
    correction="He would never approve PRs without running tests himself — he's not lazy about that",
)
```

### Version management

```python
from tools.version_manager import VersionManager

vm = VersionManager(base_dir=".claude/skills")

# List versions for a colleague
versions = vm.list_versions("zhang-wei")
for v in versions:
    print(f"{v['version']} — {v['created_at']} — {v['note']}")

# Roll back
vm.rollback("zhang-wei", version="v2")
```

---

## Generated Skill Structure

Each colleague generates two files under `colleagues/{slug}/`:

```
colleagues/zhang-wei/
├── zhang-wei.md            # Master skill entry point
├── zhang-wei-work.md       # Work Skill: tech standards, systems, workflows
├── zhang-wei-persona.md    # Persona: 5-layer personality model
├── versions/               # Auto-archived snapshots
│   ├── v1/
│   ├── v2/
│   └── v3/
└── corrections.md          # Conversation-based corrections layer
```

### Persona 5-layer model

```
Layer 1 — Hard Rules      : Non-negotiable behaviors ("never approves without tests")
Layer 2 — Identity        : Role, seniority, company culture embedding
Layer 3 — Expression      : Vocabulary, sentence length, emoji use, response latency
Layer 4 — Decision Model  : How they prioritize, what triggers caution vs. action
Layer 5 — Interpersonal   : Who they defer to, who they push back on, blame patterns
```

---

## Supported Tags

**Personality**: `认真负责` · `甩锅高手` · `完美主义` · `差不多就行` · `拖延症` · `PUA高手` · `阴阳怪气` · `话少` · `只读不回` · `反复横跳` · `向上管理专家` · `职场政治玩家`

**Company culture**: `字节范` · `阿里味` · `腾讯味` · `华为味` · `百度味` · `美团味` · `第一性原理` · `OKR狂热者` · `大厂流水线` · `创业公司派`

**Seniority levels**:
- ByteDance: `2-1` through `3-3+`
- Alibaba: `P5` through `P11`
- Tencent: `T1` through `T4`
- Baidu: `T5` through `T9`
- Meituan: `P4` through `P8`
- Huawei: Level `13` through `21`

---

## Example Interaction

> Input when creating: `字节 2-1 后端工程师, INTJ, 甩锅高手, 字节范`

**Code review scenario:**

```
User       ❯ 帮我看一下这个接口设计

zhang-wei  ❯ 等等，这个接口的 impact 是什么？背景没说清楚。
             （看完后）N+1 查询，改掉。返回结构用统一的
             {code, message, data}，这是规范，不用问为什么。
```

**Blame deflection scenario:**

```
User       ❯ 这个 bug 是你引入的吧

zhang-wei  ❯ 上线时间对上了吗？那个需求改了好几个地方，还有其他变更。
```

---

## Incremental Updates

Colleague skills improve over time without losing existing conclusions:

```python
# In Claude Code — just say:
"给张伟追加这份文档" 
# → append_material() runs, merger.md prompt merges incrementally

# Correct behavior mid-conversation:
"他不会这样，他其实很在意代码质量"
# → correction_handler.md writes to corrections.md, effective immediately
```

---

## Project Layout

```
create-colleague/
├── SKILL.md                          # Skill entry (AgentSkills frontmatter)
├── prompts/
│   ├── intake.md                     # Conversational info intake
│   ├── work_analyzer.md              # Work capability extraction
│   ├── persona_analyzer.md           # Personality extraction + tag translation
│   ├── work_builder.md               # work.md generation template
│   ├── persona_builder.md            # 5-layer persona generation template
│   ├── merger.md                     # Incremental merge logic
│   └── correction_handler.md         # Conversation correction processing
├── tools/
│   ├── feishu_auto_collector.py      # Feishu full-auto collection
│   ├── feishu_browser.py             # Feishu browser fallback
│   ├── feishu_mcp_client.py          # Feishu MCP approach
│   ├── dingtalk_auto_collector.py    # DingTalk collection
│   ├── email_parser.py               # .eml / .mbox parsing
│   ├── skill_writer.py               # Skill file management
│   └── version_manager.py            # Version archive + rollback
├── colleagues/                        # Generated skills (gitignored)
├── requirements.txt
└── LICENSE
```

---

## Troubleshooting

**Feishu collection returns empty:**
- Confirm `FEISHU_APP_ID` and `FEISHU_APP_SECRET` are set
- Ensure the bot has been added to the target group chats
- Try the browser fallback: `feishu_browser.py`

**Skill output sounds generic, not like the person:**
- Raw material quality drives output quality — add longer documents they wrote
- Prioritize: long-form writing > decision-type replies > casual chat
- Add correction: `"他不是这样的，他更倾向于..."`

**`/create-colleague` command not found:**
- Verify you cloned into `.claude/skills/create-colleague` (the directory name matters)
- Run from the git repo root, not a subdirectory
- For global install, confirm `~/.claude/skills/create-colleague/SKILL.md` exists

**Version rollback fails:**
- List available versions first: `vm.list_versions("slug")`
- Versions are only created on write operations — a brand-new skill has only `v1`
```
