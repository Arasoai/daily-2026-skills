```markdown
---
name: chops-ai-skills-manager
description: macOS app to browse, edit, and manage AI agent skills across Claude Code, Cursor, Codex, Windsurf, and Amp
triggers:
  - manage my AI agent skills
  - organize claude code skills
  - add a new skill to cursor
  - browse skills across AI tools
  - edit my agent skills files
  - set up chops on my mac
  - create a new skill file
  - sync skills between AI coding tools
---

# Chops — AI Agent Skills Manager

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Chops is a native macOS app (SwiftUI + SwiftData) that discovers, organizes, and edits AI coding agent skill files across Claude Code, Cursor, Codex, Windsurf, Copilot, Aider, and Amp — all in one place, with real-time file watching, full-text search, and a built-in markdown editor.

## Installation

### Download the App

```bash
# Direct download
curl -L https://github.com/Shpigford/chops/releases/latest/download/Chops.dmg -o Chops.dmg
open Chops.dmg
```

Or visit [chops.md](https://chops.md) and click Download.

### Build from Source

**Prerequisites:**
- macOS 15 (Sequoia) or later
- Xcode with command-line tools
- Homebrew

```bash
# Install dependencies
xcode-select --install
brew install xcodegen

# Clone and build
git clone https://github.com/Shpigford/chops.git
cd chops
xcodegen generate        # generates Chops.xcodeproj from project.yml
open Chops.xcodeproj     # then press Cmd+R to build and run
```

### CLI Build (Headless)

```bash
xcodebuild -scheme Chops -configuration Debug build
```

> **Note:** The `.xcodeproj` is generated from `project.yml`. Always edit `project.yml` and re-run `xcodegen generate` — never edit `.xcodeproj` directly.

## Where Skills Live

Chops scans these directories automatically:

| Tool | Scanned Paths |
|------|--------------|
| Claude Code | `~/.claude/skills/`, `~/.agents/skills` |
| Cursor | `~/.cursor/skills/`, `~/.cursor/rules` |
| Windsurf | `~/.codeium/windsurf/memories/`, `~/.windsurf/rules` |
| Codex | `~/.codex` |
| Amp | `~/.config/amp` |

Copilot and Aider only detect project-level skills (no global paths). Custom paths can be added per tool in Settings.

## Skill File Format

Skills are Markdown files with YAML frontmatter:

```markdown
---
name: my-skill-name
description: One-line description of what this skill does
triggers:
  - phrase users might say
  - another trigger phrase
  - add 6-8 total triggers
---

# Skill Title

Main skill content in Markdown...
```

Cursor uses `.mdc` files with the same frontmatter format. Chops handles both automatically.

## Key Features

### Multi-Tool Support
Chops reads and writes skills for all major AI coding agents. Skills are deduplicated by resolved symlink path — symlinked files appear once with multiple tool badges.

### Built-in Editor
- Monospaced `NSTextView`-based editor
- **Cmd+S** to save
- Frontmatter is parsed and displayed as structured metadata
- Changes are written directly to the source `.md` or `.mdc` file

### Collections
Group skills logically without moving or modifying source files. Collections are stored in SwiftData, not on disk.

### Real-Time File Watching
FSEvents-based watcher detects disk changes instantly and triggers a re-scan. No manual refresh needed.

### Full-Text Search
Search across skill name, description, and full content simultaneously.

### Remote Skill Servers
Connect to remote servers (e.g., [OpenClaw](https://openclaw.ai)) to discover, browse, and install community skills.

## Project Structure

```
Chops/
├── App/
│   ├── ChopsApp.swift         # @main — SwiftData ModelContainer + Sparkle updater
│   ├── AppState.swift          # @Observable singleton — all UI state
│   └── ContentView.swift       # Three-column NavigationSplitView
├── Models/
│   ├── Skill.swift             # @Model — a discovered skill file
│   ├── Collection.swift        # @Model — user-created groupings
│   └── ToolSource.swift        # Enum of tools, their paths and icons
├── Services/
│   ├── SkillScanner.swift      # Probes directories, upserts skills into SwiftData
│   ├── SkillParser.swift       # Routes to FrontmatterParser or MDCParser
│   ├── FileWatcher.swift       # FSEvents listener → triggers re-scan
│   └── SearchService.swift     # In-memory full-text search
├── Utilities/
│   ├── FrontmatterParser.swift # Extracts YAML frontmatter from .md files
│   └── MDCParser.swift         # Parses Cursor .mdc files
└── Views/
    ├── Sidebar/                # Tool filters, collection list
    ├── Detail/                 # Skill editor, metadata display
    ├── Settings/               # Preferences & Sparkle update UI
    └── Shared/                 # ToolBadge, NewSkillSheet, etc.
```

## Development: Common Tasks

### Add a New Tool

Edit `Chops/Models/ToolSource.swift`:

```swift
enum ToolSource: String, CaseIterable, Codable {
    // ... existing cases ...
    case myNewTool

    var displayName: String {
        switch self {
        case .myNewTool: return "My New Tool"
        // ...
        }
    }

    var globalPaths: [String] {
        switch self {
        case .myNewTool: return ["~/.mynewtools/skills"]
        // ...
        }
    }

    var color: Color {
        switch self {
        case .myNewTool: return .purple
        // ...
        }
    }

    var iconName: String {
        switch self {
        case .myNewTool: return "wand.and.stars"
        // ...
        }
    }

    // Optional: return asset name if you add a logo to the asset catalog
    var logoAssetName: String? {
        switch self {
        case .myNewTool: return "MyNewToolLogo"
        default: return nil
        }
    }
}
```

If the tool uses a non-standard file layout, also update `Chops/Services/SkillScanner.swift`.

### Add Custom Frontmatter Parsing

Edit `Chops/Utilities/FrontmatterParser.swift`:

```swift
struct FrontmatterParser {
    static func parse(_ content: String) -> (metadata: [String: Any], body: String) {
        // Frontmatter is delimited by --- on its own line
        // Returns structured metadata dict + remaining markdown body
        guard content.hasPrefix("---") else {
            return ([:], content)
        }
        // ... parsing logic
    }
}
```

### Support a New File Extension

Edit `Chops/Services/SkillParser.swift` to add dispatch logic:

```swift
static func parse(fileURL: URL) -> SkillMetadata {
    switch fileURL.pathExtension {
    case "mdc":
        return MDCParser.parse(fileURL: fileURL)
    case "md", "markdown":
        return FrontmatterParser.parse(fileURL: fileURL)
    case "txt":
        // Add handling for plain text skills
        return PlainTextParser.parse(fileURL: fileURL)
    default:
        return SkillMetadata(name: fileURL.lastPathComponent)
    }
}
```

### Modify the UI Layout

The main layout is a three-column `NavigationSplitView` in `Chops/App/ContentView.swift`:

```swift
NavigationSplitView {
    SidebarView()           // Column 1: tool filters + collections
} content: {
    SkillListView()         // Column 2: filtered/searched skill list
} detail: {
    SkillDetailView()       // Column 3: editor + metadata
}
```

### State Management

`AppState` is the single source of truth for all UI state:

```swift
@Observable
class AppState {
    var selectedTool: ToolSource?     // current sidebar filter
    var selectedSkill: Skill?         // currently open skill
    var searchText: String = ""       // full-text search query
    var sidebarMode: SidebarMode      // .tools or .collections
}
```

Access it in any view via `@Environment`:

```swift
struct MyView: View {
    @Environment(AppState.self) var appState

    var body: some View {
        Text("Searching: \(appState.searchText)")
    }
}
```

## Creating a Skill for Your Own Project

If your project is used with Chops-supported AI tools, create a skill file so agents have instant context:

```bash
# For Claude Code
mkdir -p .claude/skills
cat > .claude/skills/setup.md << 'EOF'
---
name: my-project-setup
description: Architecture and key patterns for MyProject
triggers:
  - how does myproject work
  - set up myproject
  - myproject architecture
---

# MyProject

[Describe your project here for AI agents]
EOF
```

Chops will automatically discover and display this skill.

## Troubleshooting

### App Can't Find My Skills
- Verify the skill files exist at the expected paths (see table above)
- Check that file extensions are `.md` or `.mdc`
- Use **File → Rescan** or restart the app to force a fresh scan
- Chops disables the macOS sandbox intentionally for unrestricted `~/` access — if you built from source, confirm `Chops.entitlements` has `com.apple.security.app-sandbox` set to `false`

### Skills Not Updating After Edit
- The FSEvents watcher should catch changes automatically
- If it doesn't, save the file again or trigger a rescan
- Ensure the file is in a directory Chops watches (custom paths can be added in Settings)

### Frontmatter Not Parsed
- Ensure the file starts with `---` on the very first line (no leading whitespace or BOM)
- YAML must be valid — test with a YAML linter
- Cursor `.mdc` files use the same frontmatter format but a different parser — ensure the extension is correct

### Build Fails After Pulling Changes
```bash
# Regenerate the Xcode project after any project.yml changes
xcodegen generate
```

### Sparkle Update Issues
Sparkle is pulled automatically via Swift Package Manager. If SPM resolution fails:
```bash
# In Xcode: File → Packages → Reset Package Caches
# Or from terminal:
rm -rf ~/Library/Caches/org.swift.swiftpm
xcodebuild -resolvePackageDependencies
```

## Website (Marketing Site)

```bash
cd site
npm install       # first time only
npm run dev       # local dev server at localhost:4321
npm run build     # production build → site/dist/
```

Built with [Astro 6](https://astro.build/).

## Architecture Notes

- **No sandbox** — Required for reading dotfiles across `~/`. Intentional and documented in `Chops.entitlements`.
- **Dedup via symlinks** — Skills identified by resolved symlink path; same file in multiple tool dirs = one skill, multiple tool badges.
- **No automated tests** — Validate manually: build → trigger feature → observe result → test edge cases.
- **SwiftData persistence** — `Skill` and `SkillCollection` models are persisted; scanning upserts (never duplicates) discovered skills.
```
