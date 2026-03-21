```markdown
---
name: does-it-age-verify
description: Track and contribute to the age verification compliance status list for open source operating systems
triggers:
  - "check age verification status of linux distros"
  - "update does it age verify list"
  - "add OS to age verification tracking"
  - "what distros support age verification"
  - "contribute to age verification compliance list"
  - "track OS age verification laws"
  - "which operating systems implement age verification"
  - "age verification status open source OS"
---

# DoesItAgeVerify

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

**DoesItAgeVerify** is a community-maintained reference list tracking whether open source operating systems (Linux distros, BSDs, etc.) have implemented, are planning to implement, or have refused to implement age verification — as required by laws passed in Brazil and California, and proposed in Colorado, Illinois, and New York.

---

## What This Project Does

This is a **documentation/data repository** (not a code library). It maintains a structured Markdown table in `README.md` categorizing open source OSes into three groups:

1. **Not Implementing** — Refused or blocking access in affected regions
2. **Planning to Implement** — Publicly committed but not yet done
3. **Already Implemented** — Fully compliant (currently empty as of last update)

---

## Repository Structure

```
DoesItAgeVerify/
├── README.md        # The main tracking list (primary artifact)
└── LICENSE          # BSD-2-Clause
```

---

## Contributing: Adding or Updating an OS Entry

All contributions are edits to `README.md`. Follow the existing table format precisely.

### Status Emoji Key

| Emoji | Meaning |
|---|---|
| `:no_entry:` | Not implementing / blocking access |
| `:building_construction:` | Planning to implement |
| `:white_check_mark:` | Fully implemented |

### Table Row Format

```markdown
| :no_entry: | **OS Name** | [Source description](https://link-to-statement) |
```

---

## Example Contributions

### Adding a new "Not Implementing" OS

```markdown
| :no_entry: | **Alpine Linux** | [Developer statement](https://example.com/statement) |
```

### Adding a new "Planning to Implement" OS

```markdown
| :building_construction: | **Fedora** | [Red Hat statement planning compliance](https://example.com/fedora-av) |
```

### Adding a fully compliant OS (first ever!)

Add a new section if needed:

```markdown
### Operating Systems Which Have Already Implemented Age Verification

| &nbsp; | Operating System | Notes |
| - | - | - |
| :white_check_mark: | **ExampleOS** | [Compliance announcement](https://example.com/compliant) |
```

---

## Editing README.md Programmatically

Since this is a Markdown data file, you can parse and update it with standard tooling.

### Python: Parse current status entries

```python
import re

def parse_av_table(readme_path: str) -> list[dict]:
    """Parse OS entries from DoesItAgeVerify README tables."""
    with open(readme_path, "r", encoding="utf-8") as f:
        content = f.read()

    # Match table rows: | emoji | **OS Name** | notes |
    pattern = r"\|\s*(:[a-z_]+:)\s*\|\s*\*\*(.+?)\*\*\s*\|\s*(.+?)\s*\|"
    entries = []

    status_map = {
        ":no_entry:": "not_implementing",
        ":building_construction:": "planning",
        ":white_check_mark:": "implemented",
    }

    for match in re.finditer(pattern, content):
        emoji, os_name, notes = match.groups()
        entries.append({
            "os": os_name.strip(),
            "status": status_map.get(emoji.strip(), "unknown"),
            "notes": notes.strip(),
        })

    return entries


if __name__ == "__main__":
    entries = parse_av_table("README.md")
    for e in entries:
        print(f"[{e['status']:20s}] {e['os']}")
```

### Python: Add a new OS entry to the correct section

```python
def add_os_entry(
    readme_path: str,
    os_name: str,
    status: str,  # "not_implementing" | "planning" | "implemented"
    notes_markdown: str,
) -> None:
    """
    Insert a new OS row into the correct section of README.md.
    
    Args:
        readme_path: Path to README.md
        os_name: Name of the OS (e.g. "Fedora Linux")
        status: One of 'not_implementing', 'planning', 'implemented'
        notes_markdown: Markdown string for the Notes column
    """
    emoji_map = {
        "not_implementing": ":no_entry:",
        "planning": ":building_construction:",
        "implemented": ":white_check_mark:",
    }
    section_map = {
        "not_implementing": "### Operating Systems Not Implementing Age Verification",
        "planning": "### Operating Systems Planning to Implement Age Verification",
        "implemented": "### Operating Systems Which Have Already Implemented Age Verification",
    }

    emoji = emoji_map[status]
    section_header = section_map[status]
    new_row = f"| {emoji} | **{os_name}** | {notes_markdown} |"

    with open(readme_path, "r", encoding="utf-8") as f:
        content = f.read()

    if os_name in content:
        print(f"WARNING: '{os_name}' may already exist in the list.")

    # Find the section and insert before the closing blank line after the table
    section_idx = content.find(section_header)
    if section_idx == -1:
        raise ValueError(f"Section not found: {section_header}")

    # Find the last table row in this section
    section_content = content[section_idx:]
    table_end = section_content.rfind("| :") 
    if table_end == -1:
        table_end = section_content.rfind("| :building_construction:")
    
    # Find end of that last row
    insert_after = section_idx + table_end
    line_end = content.find("\n", insert_after)

    updated = content[: line_end + 1] + new_row + "\n" + content[line_end + 1 :]

    with open(readme_path, "w", encoding="utf-8") as f:
        f.write(updated)

    print(f"Added: {new_row}")


# Usage
add_os_entry(
    "README.md",
    "Fedora Linux",
    "planning",
    "[Red Hat statement](https://example.com/fedora-av-statement)",
)
```

### Bash: Quick count of each status

```bash
#!/bin/bash
# Count OS entries by status in DoesItAgeVerify README

README="README.md"

echo "=== Age Verification Status Count ==="
echo "Not Implementing : $(grep -c ':no_entry:' "$README")"
echo "Planning         : $(grep -c ':building_construction:' "$README")"
echo "Implemented      : $(grep -c ':white_check_mark:' "$README")"
echo ""
echo "=== All Tracked OSes ==="
grep -oP '\*\*[^*]+\*\*' "$README" | tr -d '*' | sort -u
```

---

## Cloning and Local Setup

```bash
# Clone the repository
git clone https://github.com/BryanLunduke/DoesItAgeVerify.git
cd DoesItAgeVerify

# No build steps required — this is a documentation project
# Edit README.md directly
```

---

## Submitting a Pull Request

```bash
# Fork on GitHub first, then:
git clone https://github.com/YOUR_USERNAME/DoesItAgeVerify.git
cd DoesItAgeVerify

git checkout -b add-fedora-linux

# Edit README.md to add your entry
# Follow the table format exactly

git add README.md
git commit -m "Add Fedora Linux (planning to implement)"
git push origin add-fedora-linux

# Open PR at: https://github.com/BryanLunduke/DoesItAgeVerify/pulls
```

### PR checklist
- [ ] Entry is in the correct section
- [ ] OS name is bolded: `**OS Name**`
- [ ] Emoji matches status
- [ ] Notes column links to a **primary source** (official statement, blog post, GitHub issue, mailing list)
- [ ] Link is a real, publicly accessible URL

---

## Relevant Laws Being Tracked

| Jurisdiction | Status | Reference |
|---|---|---|
| Brazil | **Passed** | OS-level age verification required |
| California | **Passed** | OS-level age verification required |
| Colorado | Proposed | Pending |
| Illinois | Proposed | Pending |
| New York | Proposed | Pending |

---

## Common Issues

### "My OS doesn't fit any category"
If an OS has not made any statement, do not add it. Only add entries with a **citable source** (developer statement, official blog, mailing list, etc.).

### Table formatting is off
GitHub Markdown tables require `|` on every cell. Use exactly this pattern:
```markdown
| :no_entry: | **OS Name** | [Link text](https://url) |
```

### Link is to a social media post that may disappear
Prefer linking to official project pages, mailing lists, or GitHub issues when possible. Social media links (x.com, mastodon) are acceptable if that's the only source.

---

## License

BSD-2-Clause — See [LICENSE](https://github.com/BryanLunduke/DoesItAgeVerify/blob/main/LICENSE)
```
