```markdown
---
name: phoneclaw-ios-ai-agent
description: Build and extend PhoneClaw, an on-device iOS AI Agent powered by Gemma 4 running fully offline on iPhone
triggers:
  - add a new skill to PhoneClaw
  - how do I set up PhoneClaw on my iPhone
  - create a custom tool for PhoneClaw
  - how does PhoneClaw's skill system work
  - register a new iOS API tool in PhoneClaw
  - how to add calendar or contacts support to PhoneClaw
  - PhoneClaw model download and setup
  - how to extend PhoneClaw with a new capability
---

# PhoneClaw iOS AI Agent

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

PhoneClaw is a fully offline, on-device AI Agent for iPhone. It uses Gemma 4 (E2B or E4B) via MLX for local inference — no cloud, no data upload, no network dependency. It exposes iOS system APIs through a file-based Skill system where each capability is defined in a `SKILL.md` file, meaning new skills can be added or modified without recompiling the app.

---

## Project Structure

```
PhoneClaw/
├── Skills/Library/<skill-id>/SKILL.md   # Built-in skills
├── Tools/
│   ├── ToolRegistry.swift               # Register all tools here
│   └── Handlers/<Name>Handler.swift     # Per-tool iOS API logic
├── LLM/MLX/MLXLocalLLMService.swift     # Model loading & inference
├── Models/                              # Model files (gitignored)
│   ├── gemma-4-e2b-it-4bit/
│   └── gemma-4-e4b-it-4bit/
└── PhoneClaw.xcworkspace
```

---

## Installation & Setup

### Requirements

- macOS + Xcode 16
- iOS 17+ device (real hardware required for MLX inference)
- CocoaPods
- Apple Developer account (free tier works)

### Clone and Install Dependencies

```bash
git clone https://github.com/kellyvv/phoneclaw.git
cd phoneclaw
pod install
open PhoneClaw.xcworkspace  # Always open .xcworkspace, not .xcodeproj
```

### Download Models

#### Option A: Shell Install (Recommended — download on device)
Install the app via Xcode without bundling models. Open the app, go to **Model Settings**, and download E2B or E4B directly on the phone.

#### Option B: Bundle E2B into App

```bash
brew install hf
mkdir -p ./Models/gemma-4-e2b-it-4bit
hf download mlx-community/gemma-4-e2b-it-4bit \
  --local-dir ./Models/gemma-4-e2b-it-4bit
```

Then in Xcode: **Build Phases → Copy Bundle Resources** → add the model directory.

#### Option C: Bundle Both Models

```bash
mkdir -p ./Models/gemma-4-e2b-it-4bit ./Models/gemma-4-e4b-it-4bit
hf download mlx-community/gemma-4-e2b-it-4bit \
  --local-dir ./Models/gemma-4-e2b-it-4bit
hf download mlx-community/gemma-4-e4b-it-4bit \
  --local-dir ./Models/gemma-4-e4b-it-4bit
```

> E2B ≈ 3.58 GB, E4B ≈ 5.22 GB. `Models/` is gitignored.

#### ModelScope Mirror (China, no VPN needed)

Download from:
- E2B: `https://modelscope.cn/models/mlx-community/gemma-4-e2b-it-4bit`
- E4B: `https://modelscope.cn/models/mlx-community/gemma-4-e4b-it-4bit`

### Signing

1. Select PhoneClaw target → **Signing & Capabilities**
2. Choose your Team, set a unique Bundle Identifier
3. Connect iPhone → ⌘R
4. If prompted: **Settings → General → VPN & Device Management → Trust**

---

## The Skill System

Each skill is a Markdown file with YAML frontmatter. Skills live at:

- **Built-in**: `Skills/Library/<skill-id>/SKILL.md`
- **Runtime (sandboxed)**: `Application Support/PhoneClaw/skills/<skill-id>/SKILL.md`

### SKILL.md Structure

```yaml
---
name: MySkill
name-zh: 我的能力                   # Chinese display name (optional)
description: What this skill does
version: "1.0.0"
icon: star                          # SF Symbol name
disabled: false
type: device                        # "device" = calls iOS API; "content" = pure LLM

triggers:
  - phrase the user might say
  - another trigger phrase

allowed-tools:
  - my-tool-name                    # Must match ToolRegistry keys; [] for content type

examples:
  - query: "What a user might say"
    scenario: "Context/scenario description"
---

# Skill Instructions

Instructions telling the model when to call tools, how to form parameters,
and when to answer directly without tool calls.
```

### Skill Types

| Type | Behavior | Use When |
|------|----------|----------|
| `device` | Model emits `<tool_call>` → Swift handler executes iOS API | Need real system access (calendar, contacts, clipboard) |
| `content` | Model processes input and responds directly | Translation, summarization, text manipulation |

---

## Creating a New Skill (Step-by-Step)

### Step 1: Create the SKILL.md

```
Skills/Library/weather-note/SKILL.md
```

```yaml
---
name: WeatherNote
name-zh: 天气备忘
description: Saves weather-related notes to reminders
version: "1.0.0"
icon: cloud.sun
disabled: false
type: device

triggers:
  - remind me about the weather
  - save a weather note

allowed-tools:
  - create-reminder

examples:
  - query: "Remind me to check the weather at 7am"
    scenario: "User wants a weather-related reminder"
---

# WeatherNote Skill Instructions

When the user wants to save a weather-related note or reminder, call the
`create-reminder` tool with the appropriate title and due date.
If no time is specified, schedule for the next morning at 7:00 AM.
Always confirm after creation.
```

### Step 2: Register the Tool in ToolRegistry.swift

```swift
// Tools/ToolRegistry.swift

import Foundation

struct ToolRegistry {
    static let shared = ToolRegistry()

    // All registered tools keyed by name
    private(set) var tools: [String: any ToolHandler] = [:]

    init() {
        register(CreateReminderHandler())
        register(ReadClipboardHandler())
        register(WriteClipboardHandler())
        register(CreateCalendarEventHandler())
        register(SaveContactHandler())
        register(GetDeviceInfoHandler())
        // Register your new tool:
        register(MyCustomHandler())
    }

    mutating func register(_ handler: any ToolHandler) {
        tools[handler.toolName] = handler
    }
}
```

### Step 3: Implement the Tool Handler

```swift
// Tools/Handlers/MyCustomHandler.swift

import Foundation
import EventKit

struct CreateReminderHandler: ToolHandler {
    let toolName = "create-reminder"

    // Describe parameters so the model knows how to call this tool
    var parameterSchema: [String: ToolParameter] {
        [
            "title": ToolParameter(type: .string, description: "Reminder title", required: true),
            "dueDate": ToolParameter(type: .string, description: "ISO 8601 date string", required: false),
            "notes": ToolParameter(type: .string, description: "Additional notes", required: false)
        ]
    }

    func execute(parameters: [String: Any]) async throws -> String {
        guard let title = parameters["title"] as? String else {
            throw ToolError.missingRequiredParameter("title")
        }

        let store = EKEventStore()

        // Request permission
        let granted = try await store.requestFullAccessToReminders()
        guard granted else {
            throw ToolError.permissionDenied("Reminders access denied")
        }

        let reminder = EKReminder(eventStore: store)
        reminder.title = title

        // Find or create PhoneClaw list
        let calendars = store.calendars(for: .reminder)
        let targetCalendar = calendars.first(where: { $0.title == "PhoneClaw" })
            ?? calendars.first(where: { $0.allowsContentModifications })
        
        guard let calendar = targetCalendar else {
            throw ToolError.executionFailed("No writable reminder list found")
        }
        reminder.calendar = calendar

        // Parse optional due date
        if let dueDateString = parameters["dueDate"] as? String {
            let formatter = ISO8601DateFormatter()
            if let date = formatter.date(from: dueDateString) {
                let components = Calendar.current.dateComponents(
                    [.year, .month, .day, .hour, .minute, .second],
                    from: date
                )
                reminder.dueDateComponents = components

                let alarm = EKAlarm(absoluteDate: date)
                reminder.addAlarm(alarm)
            }
        }

        if let notes = parameters["notes"] as? String {
            reminder.notes = notes
        }

        try store.save(reminder, commit: true)
        return "Reminder '\(title)' created successfully."
    }
}
```

### Step 4: Validate

The framework auto-validates `allowed-tools` against `ToolRegistry` at startup. Watch the Xcode console — mismatched tool names are flagged immediately.

---

## Built-In Skills Reference

| Skill ID | Type | What It Does |
|----------|------|-------------|
| `calendar` | device | Create calendar events with title, time, location |
| `reminders` | device | Create timed reminders with system notifications |
| `contacts` | device | Save/update contacts; deduplicates by phone number |
| `clipboard` | device | Read and write system clipboard |
| `device-info` | device | Query device name, OS version, memory, CPU count |
| `text-tools` | device | MD5 hash, text reversal, basic text operations |
| `translate` | content | Pure LLM translation, no tool call needed |

---

## Model Configuration

### Available Models (MLXLocalLLMService.swift)

```swift
// LLM/MLX/MLXLocalLLMService.swift

let availableModels: [ModelConfig] = [
    ModelConfig(
        id: "gemma-4-e2b-it-4bit",
        displayName: "Gemma 4 E2B",
        directoryName: "gemma-4-e2b-it-4bit",
        recommendedFor: "A16 and above, stable default"
    ),
    ModelConfig(
        id: "gemma-4-e4b-it-4bit",
        displayName: "Gemma 4 E4B",
        directoryName: "gemma-4-e4b-it-4bit",
        recommendedFor: "iPhone 15 Pro and above, better quality"
    )
]
```

### Model Selection Guidelines

| Model | Size | Best For |
|-------|------|----------|
| Gemma 4 E2B | 3.58 GB | Default distribution, A16+, stable |
| Gemma 4 E4B | 5.22 GB | iPhone 15 Pro+, higher quality output |

### Inference Budget

Memory is allocated dynamically based on available device RAM. Long prompts and responses are no longer truncated by a fixed budget — the app calculates limits from actual free memory at inference time.

---

## Content-Type Skill Example

For skills that don't need iOS APIs — just LLM processing:

```yaml
---
name: Summarize
name-zh: 摘要
description: Summarize any text the user provides
version: "1.0.0"
icon: text.quote
disabled: false
type: content          # No tool calls — model answers directly

triggers:
  - summarize this
  - give me a summary
  - tl;dr

allowed-tools: []      # Empty for content type

examples:
  - query: "Summarize this article for me"
    scenario: "User pastes a long text and wants a summary"
---

# Summarize Skill Instructions

When the user asks for a summary, read the provided text carefully and
produce a concise summary in 3-5 bullet points. Use the same language
as the input text. Do not call any tools.
```

---

## Multimodal (Image) Usage

PhoneClaw supports image input via camera or photo library. The model processes images fully on-device.

```swift
// Example: Sending an image with a text query
// In your chat view, attach UIImage to the message before sending

let message = ChatMessage(
    role: .user,
    content: "What is in this image?",
    image: selectedUIImage   // Optional UIImage attachment
)
chatViewModel.send(message)
```

Image data never leaves the device — all vision inference runs locally via the multimodal Gemma 4 model.

---

## Voice Input

Voice recording is supported in the chat UI. Tap the microphone icon to record; the audio is transcribed and analyzed on-device.

---

## Session Management

- **New session**: Tap the compose icon in the top bar
- **Switch session**: Access history from the sidebar
- **Delete session**: Swipe to delete in session list
- **Thinking mode**: Toggle from the top-right menu in chat view

---

## Runtime Skill Installation (No Recompile)

Install skills at runtime by writing to the app sandbox:

```swift
// Write a custom SKILL.md to the app's sandbox at runtime
let skillDir = FileManager.default
    .urls(for: .applicationSupportDirectory, in: .userDomainMask)
    .first!
    .appendingPathComponent("PhoneClaw/skills/my-custom-skill")

try FileManager.default.createDirectory(
    at: skillDir,
    withIntermediateDirectories: true
)

let skillContent = """
---
name: MyRuntimeSkill
description: Dynamically installed skill
version: "1.0.0"
icon: wand.and.stars
disabled: false
type: content
triggers:
  - do my custom thing
allowed-tools: []
examples: []
---

# My Runtime Skill
Answer the user's request directly without tools.
"""

let skillFile = skillDir.appendingPathComponent("SKILL.md")
try skillContent.write(to: skillFile, atomically: true, encoding: .utf8)
```

The app loads skills from both the bundle and the sandbox on startup.

---

## Troubleshooting

### Model fails to load after switching

1. Verify the model directory name exactly matches `availableModels` in `MLXLocalLLMService.swift`
2. For shell installs: confirm the model finished downloading on-device before switching
3. For bundled installs: confirm the model folder is in **Copy Bundle Resources**
4. Check available RAM — E4B needs iPhone 15 Pro or above

### Reminder creation fails

The app first tries to reuse an existing writable reminder list. If none exists, it creates a "PhoneClaw" list. If this still fails, the system reminder source is not writable (e.g., iCloud Reminders disabled). Check: **Settings → [Your Name] → iCloud → Reminders**.

### Permission dialog never appears

- The skill hasn't executed a real API call yet (dialogs appear on first use)
- If previously denied: **Settings → Privacy & Security → [Contacts/Calendars/Reminders] → PhoneClaw → Allow**

### Tool name mismatch error in console

```
[ToolRegistry] WARNING: Skill 'my-skill' declares tool 'my-tool-name'
but no handler is registered for this key.
```

Fix: ensure `handler.toolName` in your `ToolHandler` implementation exactly matches the string in `allowed-tools` in your `SKILL.md`.

### Pod install issues

```bash
pod deintegrate
pod install
```

Always reopen `.xcworkspace` after reinstalling pods.

---

## Key APIs and Patterns

### ToolHandler Protocol

```swift
protocol ToolHandler {
    var toolName: String { get }
    var parameterSchema: [String: ToolParameter] { get }
    func execute(parameters: [String: Any]) async throws -> String
}
```

### ToolParameter

```swift
struct ToolParameter {
    enum ParameterType { case string, number, boolean, array }
    let type: ParameterType
    let description: String
    let required: Bool
}
```

### ToolError

```swift
enum ToolError: Error {
    case missingRequiredParameter(String)
    case permissionDenied(String)
    case executionFailed(String)
    case invalidParameter(String)
}
```

---

## Roadmap Highlights

- **Planned**: File/photo/Notes API access
- **Planned**: Shortcuts / App Intents integration
- **Planned**: OCR + speech recognition local models
- **Planned**: Local knowledge base retrieval
- **Planned**: Multi-model coordination (embedding, reranker, smaller tool-call models)
- **Planned**: Cross-app automation via URL schemes, Share Sheet, clipboard relay

---

## References

- [Gemma 4 E2B on Hugging Face](https://huggingface.co/mlx-community/gemma-4-e2b-it-4bit)
- [Gemma 4 E4B on Hugging Face](https://huggingface.co/mlx-community/gemma-4-e4b-it-4bit)
- [Gemma 4 E2B on ModelScope](https://modelscope.cn/models/mlx-community/gemma-4-e2b-it-4bit)
- [Gemma 4 E4B on ModelScope](https://modelscope.cn/models/mlx-community/gemma-4-e4b-it-4bit)
- [Hugging Face CLI Docs](https://huggingface.co/docs/huggingface_hub/guides/cli)
- [PhoneClaw GitHub](https://github.com/kellyvv/phoneclaw)
```
