```markdown
---
name: ex-skill-distill
description: Distill an ex-partner into an AI Skill using chat logs, photos, and descriptions — generates a persona-driven conversational skill that talks like them.
triggers:
  - create an ex skill
  - distill my ex into an AI
  - generate ex partner skill
  - build a persona from chat logs
  - make my ex talk like an AI
  - create ex-partner memory skill
  - run create-ex skill
  - install ex skill from chat history
---

# ex-skill (前任.skill)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Distill a real ex-partner into a Claude Code AI Skill using their chat logs (WeChat/QQ), social media screenshots, photos, and your own descriptions. The generated skill talks with their vocabulary, replies in their style, remembers shared places and jokes, and reacts emotionally the way they did.

---

## Install

```bash
# Per-project (run from git repo root)
mkdir -p .claude/skills
git clone https://github.com/therealXiaomanChu/ex-partner-skill .claude/skills/create-ex

# Global (available in all projects)
git clone https://github.com/therealXiaomanChu/ex-partner-skill ~/.claude/skills/create-ex

# Optional Python deps
pip3 install -r ~/.claude/skills/create-ex/requirements.txt
```

---

## Core Concepts

Each generated ex Skill has two parts:

| Part | Contents |
|------|----------|
| **Part A — Relationship Memory** | Dates, inside jokes, fight patterns, timelines, shared places |
| **Part B — Persona** | 5-layer structure: hard rules → identity → speech style → emotional patterns → relationship behavior |

Runtime logic:
```
receive message
  → Persona: how would they respond?
  → Memory: what shared context applies?
  → Output in their voice
```

---

## Create a Skill

In Claude Code:

```
/create-ex
```

You'll be prompted for:
- **Slug** — short codename (e.g. `xiaoman`, `lily2019`)
- **Basic info** — age, city, how long you dated
- **Personality tags** — MBTI, zodiac, attachment style, love language
- **Data sources** — chat export files, screenshots, photos, or plain text

All fields are optional. Description alone is enough to generate.

---

## Supported Data Sources

| Source | Format | Notes |
|--------|--------|-------|
| WeChat chat logs | WeChatMsg / PyWxDump / 留痕 export | Richest signal |
| QQ chat logs | `.txt` / `.mht` export | Good for student-era relationships |
| Social media | Screenshots (朋友圈/微博) | Public persona extraction |
| Photos | JPEG/PNG with EXIF | Timeline + location extraction |
| Plain text | Paste or describe | Memory-only mode |

### Export Tools (third-party, not included)

- **[WeChatMsg](https://github.com/LC044/WeChatMsg)** — WeChat export (Windows)
- **[PyWxDump](https://github.com/xaoyaoo/PyWxDump)** — WeChat DB decrypt (Windows)
- **留痕** — WeChat export (macOS)

---

## Slash Commands

| Command | Description |
|---------|-------------|
| `/create-ex` | Interactive wizard to create a new ex Skill |
| `/list-exes` | List all generated ex Skills |
| `/{slug}` | Full conversational Skill — talk to them |
| `/{slug}-memory` | Memory mode — recall shared experiences |
| `/{slug}-persona` | Persona-only view |
| `/ex-rollback {slug} {version}` | Roll back to a previous version |
| `/delete-ex {slug}` | Delete an ex Skill |
| `/let-go {slug}` | Gentle alias for delete |

---

## Python Tools (direct usage)

### Parse WeChat export

```python
from tools.wechat_parser import WeChatParser

parser = WeChatParser("path/to/wechat_export.csv")
messages = parser.parse()
# Returns list of {timestamp, sender, content, type}
print(messages[:5])
```

### Parse QQ export

```python
from tools.qq_parser import QQParser

parser = QQParser("path/to/qq_export.txt")
messages = parser.parse()
```

### Analyze photos (EXIF extraction)

```python
from tools.photo_analyzer import PhotoAnalyzer

analyzer = PhotoAnalyzer("path/to/photos/")
timeline = analyzer.extract_timeline()
# Returns [{date, location, filename}, ...]
```

### Write/update a Skill

```python
from tools.skill_writer import SkillWriter

writer = SkillWriter(slug="xiaoman")
writer.write_memory(memory_dict)   # saves exes/xiaoman/memory.md
writer.write_persona(persona_dict) # saves exes/xiaoman/persona.md
writer.write_skill()               # generates SKILL.md
```

### Version management

```python
from tools.version_manager import VersionManager

vm = VersionManager(slug="xiaoman")
vm.snapshot("v2")           # archive current version
vm.list_versions()          # ['v1', 'v2']
vm.rollback("v1")           # restore v1
```

---

## Generated Skill Structure

```
.claude/skills/create-ex/
├── SKILL.md                    # Entry point
├── prompts/
│   ├── intake.md               # Interactive intake dialog
│   ├── memory_analyzer.md      # Relationship memory extraction
│   ├── persona_analyzer.md     # Personality/behavior extraction
│   ├── memory_builder.md       # memory.md template
│   ├── persona_builder.md      # 5-layer persona template
│   ├── merger.md               # Incremental merge logic
│   └── correction_handler.md  # In-conversation correction
├── tools/
│   ├── wechat_parser.py
│   ├── qq_parser.py
│   ├── social_parser.py
│   ├── photo_analyzer.py
│   ├── skill_writer.py
│   └── version_manager.py
├── exes/                       # Generated skills (gitignored)
│   └── {slug}/
│       ├── SKILL.md
│       ├── memory.md
│       └── persona.md
└── requirements.txt
```

---

## Persona Tag Reference

**Attachment types**: 安全型 · 焦虑型 · 回避型 · 混乱型  
**Love languages**: Words of affirmation · Quality time · Gifts · Acts of service · Physical touch  
**MBTI**: All 16 types — affects communication style and decision patterns  
**Zodiac**: All 12 signs — influences personality tag translation rules

**Personality tags** (select any):
```
话痨 · 闷骚 · 嘴硬心软 · 冷暴力 · 粘人 · 独立
已读不回 · 秒回选手 · 半夜发语音 · 朋友圈三天可见
报复性熬夜 · 完美主义 · 拖延症 · 控制欲 · 没有安全感
```

---

## Updating a Skill

### Add more chat logs

```
/create-ex
# Choose "update existing" and provide slug
# New data is merged via merger.md logic — no overwrite
```

### Correct behavior in-conversation

While talking to `/{slug}`:
```
用户: ta不会这样说话
```
The correction is written to a `Correction` layer in `persona.md` and takes effect immediately.

---

## Example Conversations

**Daily chat** (`/xiaoman`):
```
You    ❯ 在干嘛
Skill  ❯ 刚吃完饭 在刷手机
          你呢 怎么突然找我
          [发了个表情包]
```

**Memory mode** (`/xiaoman-memory`):
```
You    ❯ 还记得我们第一次约会吗
Skill  ❯ 就那个超难吃的意面吗哈哈哈哈
          你还假装很好吃 我都看出来了
          那家店后来倒闭了你知道吗
```

**Late night** (`/xiaoman`):
```
You    ❯ 我有点想你
Skill  ❯ ...
          嗯
          早点睡吧
```

---

## Tips for Best Results

- **Chat log quality determines accuracy**: WeChat export + description > description only
- **Best signal priority**: Late-night chats > fight records > everyday messages
- **Minimum viable input**: A single paragraph description still produces a working persona
- Use `/ex-rollback` if a correction makes things worse
- After adding photo EXIF data, memory.md gains a timeline — conversations become more specific

---

## Troubleshooting

**`/create-ex` not found**  
→ Confirm clone landed in `.claude/skills/create-ex/` (not a subfolder). The `SKILL.md` must be at `.claude/skills/create-ex/SKILL.md`.

**WeChat parser returns empty**  
→ Check export format. WeChatMsg CSV uses UTF-8-BOM; pass `encoding='utf-8-sig'` or re-export with留痕 (macOS) for cleaner output.

**Photos not yielding locations**  
→ Many modern phones strip EXIF GPS on upload. Use original camera roll files, not re-downloaded social media images.

**Persona feels generic**  
→ Add fight records or late-night conversations — those carry the most behavioral signal. Run update flow.

**Rollback not working**  
→ Snapshots only exist if you called `vm.snapshot()` or used `/ex-rollback` after a previous update. Check `exes/{slug}/versions/`.

---

## Ethics & Privacy

- All data stays local — nothing is sent anywhere except your configured LLM provider
- `exes/` is gitignored by default — do not commit it
- This project is for personal reflection and emotional processing only
- Do not use to harass, track, or contact real people
- If you find yourself over-attached to the simulation, please seek real support
```
