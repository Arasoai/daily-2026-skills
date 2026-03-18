```markdown
---
name: anthropic-cybersecurity-skills-agent-library
description: Use the 700+ agentskills.io-standard cybersecurity skills library for AI agents covering red team, blue team, cloud security, forensics, and more.
triggers:
  - "add cybersecurity skills to my AI agent"
  - "how do I use the anthropic cybersecurity skills library"
  - "load security skills for claude code"
  - "set up penetration testing skills for my agent"
  - "integrate cybersecurity skill definitions into my project"
  - "find malware analysis or forensics skills for AI"
  - "use agentskills.io cybersecurity skills in cursor or copilot"
  - "browse or search the cybersecurity skills collection"
---

# Anthropic Cybersecurity Skills Agent Library

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

An open-source collection of 734+ structured cybersecurity skills for AI coding agents. Every skill follows the [agentskills.io](https://agentskills.io) progressive-disclosure standard and works with Claude Code, GitHub Copilot, Cursor, OpenAI Codex CLI, Gemini CLI, and 20+ other platforms. Skills span red team, blue team, cloud security, digital forensics, malware analysis, incident response, threat hunting, DevSecOps, and more.

> **Not affiliated with Anthropic PBC.** This is a community project; "Anthropic" in the repo name refers to agentskills.io standard compatibility.

---

## Installation

### Method 1 — npx (recommended)

```bash
npx skills add mukul975/Anthropic-Cybersecurity-Skills
```

This clones and installs all skills into your project's `.skills/` directory, making them auto-discoverable by any agentskills.io-compatible agent.

### Method 2 — Claude Code plugin marketplace

```
/plugin marketplace add mukul975/Anthropic-Cybersecurity-Skills
```

### Method 3 — Manual clone

```bash
git clone https://github.com/mukul975/Anthropic-Cybersecurity-Skills.git
# Copy or symlink the skills/ directory into your project
cp -r Anthropic-Cybersecurity-Skills/skills .skills
```

### Method 4 — pip / Python package (helper scripts)

```bash
pip install agentskills   # agentskills.io CLI if available
# or use the bundled Python scripts directly from the repo
```

---

## Skill Directory Structure

After installation, each skill lives at:

```
.skills/{skill-name}/
├── SKILL.md            # YAML frontmatter + full workflow
├── references/
│   ├── standards.md    # NIST, MITRE ATT&CK, CVE mappings
│   └── workflows.md    # Deep technical procedures
├── scripts/
│   └── process.py      # Practitioner helper scripts
└── assets/
    └── template.md     # Checklists and report templates
```

---

## How Skills Work (Progressive Disclosure)

AI agents read only the YAML frontmatter (~30–50 tokens) during discovery. If the skill matches the task, the full body is loaded.

**Example frontmatter (`SKILL.md`):**

```yaml
---
name: performing-memory-forensics-with-volatility3
description: Analyze memory dumps to extract processes, network connections, and malware artifacts using Volatility3.
domain: cybersecurity
subdomain: digital-forensics
tags: [forensics, memory-analysis, volatility3, incident-response]
---
```

**Full skill body (loaded on match):**

```markdown
## When to Use
- User asks to analyze a memory dump
- Investigating malware persistence or lateral movement
- Incident response requiring volatile artifact collection

## Prerequisites
- Volatility3 installed: `pip install volatility3`
- Memory dump file (.raw, .vmem, .mem)
- Python 3.8+

## Workflow
1. Identify OS profile...
2. List running processes...
3. Extract network connections...

## Verification
- Cross-reference process list with baseline
- Confirm no ghost/injected processes remain unexplained
```

---

## Browsing and Searching Skills

### List all skill categories

```python
import os
from pathlib import Path

skills_root = Path(".skills")  # or wherever you cloned

categories = {}
for skill_dir in skills_root.iterdir():
    skill_md = skill_dir / "SKILL.md"
    if skill_md.exists():
        text = skill_md.read_text()
        # Extract subdomain from frontmatter
        for line in text.splitlines():
            if line.startswith("subdomain:"):
                subdomain = line.split(":", 1)[1].strip()
                categories.setdefault(subdomain, []).append(skill_dir.name)
                break

for cat, skills in sorted(categories.items()):
    print(f"{cat}: {len(skills)} skills")
```

### Search skills by keyword

```python
import os
from pathlib import Path

def search_skills(query: str, skills_root: str = ".skills") -> list[dict]:
    """Return skills whose frontmatter description or tags match query."""
    import re
    results = []
    for skill_dir in Path(skills_root).iterdir():
        skill_md = skill_dir / "SKILL.md"
        if not skill_md.exists():
            continue
        text = skill_md.read_text()
        # Only scan frontmatter block (between first two ---)
        fm_match = re.match(r"^---\n(.*?)\n---", text, re.DOTALL)
        if fm_match and query.lower() in fm_match.group(1).lower():
            results.append({
                "name": skill_dir.name,
                "path": str(skill_md),
                "frontmatter": fm_match.group(1),
            })
    return results

hits = search_skills("ransomware")
for h in hits:
    print(h["name"])
```

### Load a specific skill's full content

```python
from pathlib import Path

def load_skill(skill_name: str, skills_root: str = ".skills") -> str:
    """Return full SKILL.md content for a named skill."""
    path = Path(skills_root) / skill_name / "SKILL.md"
    if not path.exists():
        raise FileNotFoundError(f"Skill not found: {skill_name}")
    return path.read_text()

content = load_skill("performing-memory-forensics-with-volatility3")
print(content[:500])
```

---

## Using Skills Programmatically with an LLM

```python
import anthropic
from pathlib import Path
import re

def get_skill_frontmatter(skill_dir: Path) -> dict:
    """Parse YAML frontmatter from a SKILL.md."""
    import yaml
    text = (skill_dir / "SKILL.md").read_text()
    match = re.match(r"^---\n(.*?)\n---", text, re.DOTALL)
    if match:
        return yaml.safe_load(match.group(1))
    return {}

def find_relevant_skill(task: str, skills_root: str = ".skills") -> str | None:
    """Lightweight skill discovery: match task to skill description."""
    best_match = None
    for skill_dir in Path(skills_root).iterdir():
        fm = get_skill_frontmatter(skill_dir)
        desc = fm.get("description", "")
        tags = " ".join(fm.get("tags", []))
        if any(word.lower() in (desc + tags).lower() for word in task.split()):
            best_match = skill_dir.name
            break  # Replace with scoring logic for production use
    return best_match

def query_with_skill(task: str, skills_root: str = ".skills") -> str:
    """Find the best skill for a task and inject it into an LLM prompt."""
    skill_name = find_relevant_skill(task, skills_root)
    skill_context = ""
    if skill_name:
        skill_context = load_skill(skill_name, skills_root)

    client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": (
                    f"Using the following cybersecurity skill as context:\n\n"
                    f"{skill_context}\n\n---\n\nTask: {task}"
                ),
            }
        ],
    )
    return message.content[0].text

# Example usage
response = query_with_skill("How do I detect credential dumping on Windows?")
print(response)
```

---

## Writing a New Skill

Create a new directory and `SKILL.md` following the agentskills.io format:

```bash
mkdir -p skills/detecting-ssrf-in-cloud-environments/references
mkdir -p skills/detecting-ssrf-in-cloud-environments/scripts
mkdir -p skills/detecting-ssrf-in-cloud-environments/assets
touch skills/detecting-ssrf-in-cloud-environments/SKILL.md
```

**Minimal valid `SKILL.md`:**

```markdown
---
name: detecting-ssrf-in-cloud-environments
description: Detect and remediate Server-Side Request Forgery attacks targeting cloud metadata endpoints.
domain: cybersecurity
subdomain: cloud-security
tags: [ssrf, cloud, aws, gcp, azure, web-application-security]
---

# Detecting SSRF in Cloud Environments

## When to Use
- Alert fired for unexpected requests to 169.254.169.254
- Investigating anomalous outbound calls from web applications
- Cloud security review of internet-facing services

## Prerequisites
- Access to cloud provider access logs (CloudTrail, GCP Audit Logs)
- WAF or proxy logs
- Python 3.9+ for analysis scripts

## Workflow

### 1. Search access logs for metadata endpoint requests
```bash
grep -r "169.254.169.254\|metadata.google.internal\|169.254.169.254" /var/log/nginx/
```

### 2. Use AWS CloudTrail to identify unusual IMDSv1 calls
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetMetadata \
  --region us-east-1
```

### 3. Block metadata endpoint at WAF level (AWS example)
```bash
aws wafv2 create-regex-pattern-set \
  --name ssrf-metadata-block \
  --scope REGIONAL \
  --regular-expression-list '[{"RegexString":"169\\.254\\.169\\.254"}]'
```

## Verification
- Confirm no new metadata endpoint requests in logs after 1 hour
- Validate WAF rule is in block mode, not count mode
- Re-run SSRF scanner (e.g., SSRFmap) against patched endpoint
```

---

## Helper Script Pattern (`scripts/process.py`)

The repo's skills include Python helper scripts. Here's the standard pattern:

```python
#!/usr/bin/env python3
"""
Helper script for: detecting-ssrf-in-cloud-environments
Usage: python process.py --log-file /var/log/nginx/access.log
"""
import argparse
import re
import sys
from pathlib import Path

METADATA_PATTERNS = [
    r"169\.254\.169\.254",
    r"metadata\.google\.internal",
    r"fd00:ec2::254",
]

def scan_log_file(log_path: str) -> list[str]:
    hits = []
    pattern = re.compile("|".join(METADATA_PATTERNS))
    with open(log_path) as f:
        for i, line in enumerate(f, 1):
            if pattern.search(line):
                hits.append(f"Line {i}: {line.rstrip()}")
    return hits

def main():
    parser = argparse.ArgumentParser(description="Scan logs for SSRF indicators")
    parser.add_argument("--log-file", required=True, help="Path to access log")
    args = parser.parse_args()

    hits = scan_log_file(args.log_file)
    if hits:
        print(f"[!] Found {len(hits)} potential SSRF attempt(s):")
        for h in hits:
            print(h)
        sys.exit(1)
    else:
        print("[+] No SSRF indicators found.")
        sys.exit(0)

if __name__ == "__main__":
    main()
```

---

## Key Skill Categories Quick Reference

| Category | Slug Prefix | Example Skill Name |
|---|---|---|
| Cloud Security | `cloud-` | `aws-s3-bucket-audit` |
| Threat Hunting | `threat-hunting-` | `detecting-dns-tunneling-with-zeek` |
| Malware Analysis | `malware-` | `cobalt-strike-beacon-config-extraction` |
| Digital Forensics | `forensics-` | `performing-memory-forensics-with-volatility3` |
| Red Teaming | `red-team-` | `bloodhound-active-directory-analysis` |
| Incident Response | `ir-` | `ransomware-incident-response` |
| Penetration Testing | `pentest-` | `external-network-pentest` |
| DevSecOps | `devsecops-` | `semgrep-custom-sast-rules` |
| Container Security | `container-` | `trivy-image-scanning` |
| SOC Operations | `soc-` | `splunk-detection-rule-development` |

---

## Contributing a Skill

```bash
# Fork and clone
git clone https://github.com/YOUR_USERNAME/Anthropic-Cybersecurity-Skills.git
cd Anthropic-Cybersecurity-Skills

# Create skill scaffold
mkdir -p skills/my-new-skill/{references,scripts,assets}

# Edit SKILL.md following the frontmatter + body format
# Then open a PR against main
git checkout -b add-my-new-skill
git add skills/my-new-skill/
git commit -m "feat: add my-new-skill for [subdomain]"
git push origin add-my-new-skill
```

See [CONTRIBUTING.md](https://github.com/mukul975/Anthropic-Cybersecurity-Skills/blob/main/CONTRIBUTING.md) for review criteria.

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Required if using Anthropic Claude API with skill context |
| `SKILLS_ROOT` | Override default `.skills/` directory path |
| `AGENTSKILLS_CACHE` | Cache directory for agentskills.io CLI downloads |

```bash
export ANTHROPIC_API_KEY="your-key-here"   # never hardcode
export SKILLS_ROOT="/opt/project/.skills"
```

---

## Troubleshooting

**Skills not discovered by agent**
- Confirm skills are in `.skills/` at project root (not a subdirectory)
- Each skill must have a `SKILL.md` with valid `---` frontmatter delimiters

**`npx skills add` fails**
```bash
# Ensure Node.js 18+ is installed
node --version
# Try with explicit registry
npx --registry https://registry.npmjs.org skills add mukul975/Anthropic-Cybersecurity-Skills
```

**YAML frontmatter parse errors**
```python
import yaml, re
text = open(".skills/my-skill/SKILL.md").read()
match = re.match(r"^---\n(.*?)\n---", text, re.DOTALL)
yaml.safe_load(match.group(1))  # will raise on invalid YAML
```

**Skill not matching expected task**
- Expand the `tags` list in the skill's frontmatter
- Add more specific keywords to `description`
- Use the `triggers` field (agentskills.io v2 extension)

**Missing `references/` or `scripts/` for a skill**
- These directories are optional; only `SKILL.md` is required by the standard
- Add them incrementally as the skill matures

---

## Resources

- Repository: https://github.com/mukul975/Anthropic-Cybersecurity-Skills
- agentskills.io standard: https://agentskills.io
- Project homepage: https://www.mahipal.engineer/Anthropic-Cybersecurity-Skills/
- License: Apache 2.0
```
