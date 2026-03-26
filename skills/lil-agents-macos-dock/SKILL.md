```markdown
---
name: lil-agents-macos-dock
description: Tiny AI companions (Bruce and Jazz) that live on your macOS dock and provide Claude Code, OpenAI Codex, and GitHub Copilot CLI access via animated characters
triggers:
  - "add lil agents to my dock"
  - "set up dock AI companions"
  - "configure lil agents"
  - "add a new character or theme to lil agents"
  - "integrate claude codex copilot with lil agents"
  - "build lil agents from source"
  - "customize lil agents appearance"
  - "troubleshoot lil agents not showing"
---

# lil-agents macOS Dock AI Companions

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

lil-agents places animated AI companion characters (Bruce and Jazz) above your macOS dock. Click a character to open a themed terminal popover that shells out to your chosen AI CLI (Claude Code, OpenAI Codex, or GitHub Copilot). Characters walk, display thinking bubbles, and play sound effects — all rendered from transparent HEVC video bundled in the app.

---

## What It Does

| Feature | Detail |
|---|---|
| Characters | Bruce & Jazz walk back and forth above the dock |
| AI backends | Claude Code, OpenAI Codex CLI, GitHub Copilot CLI |
| Themes | Peach, Midnight, Cloud, Moss |
| Terminal | Themed popover with live streaming output |
| Thinking bubbles | Playful phrases while the agent runs |
| Updates | Sparkle framework for auto-updates |
| Privacy | Fully local — no telemetry, no accounts |

---

## Requirements

- macOS Sonoma 14.0+
- Xcode 15+ (to build from source)
- At least one AI CLI installed

### Install AI CLIs

```bash
# Claude Code
# Download from https://claude.ai/download and install

# OpenAI Codex
npm install -g @openai/codex

# GitHub Copilot CLI
brew install copilot-cli
```

---

## Building from Source

```bash
git clone https://github.com/ryanstephen/lil-agents.git
cd lil-agents
open lil-agents.xcodeproj
# Press ⌘R in Xcode to build and run
```

No additional package manager steps are required — dependencies (Sparkle) are resolved automatically by Xcode's Swift Package Manager integration.

---

## Project Structure

```
lil-agents/
├── lil-agents.xcodeproj/
├── lil agents/
│   ├── App/
│   │   ├── AppDelegate.swift          # NSApplication entry, StatusItem, Sparkle
│   │   └── OnboardingWindowController.swift
│   ├── Characters/
│   │   ├── CharacterWindowController.swift   # Transparent overlay window above dock
│   │   ├── CharacterView.swift               # AVPlayer HEVC rendering
│   │   └── ThinkingBubbleView.swift
│   ├── Terminal/
│   │   ├── TerminalPopoverController.swift   # Popover terminal UI
│   │   ├── AgentProcess.swift                # Shells out to CLI
│   │   └── TerminalTheme.swift               # Peach/Midnight/Cloud/Moss
│   ├── Settings/
│   │   └── SettingsManager.swift             # UserDefaults-backed config
│   ├── Resources/
│   │   ├── bruce/                    # HEVC .mov files (walk, think, idle)
│   │   ├── jazz/                     # HEVC .mov files
│   │   └── Sounds/
│   └── Info.plist
└── README.md
```

---

## Core Architecture

### Transparent Window Above the Dock

Characters live in a `NSWindow` with `level = .statusBar` positioned just above the dock frame.

```swift
// CharacterWindowController.swift pattern
import AppKit
import AVKit

class CharacterWindowController: NSWindowController {

    private var playerView: AVPlayerView!
    private var player: AVPlayer!

    override func windowDidLoad() {
        super.windowDidLoad()

        guard let window = window else { return }

        // Make window transparent and click-through by default
        window.isOpaque = false
        window.backgroundColor = .clear
        window.hasShadow = false
        window.ignoresMouseEvents = false   // false so clicks register
        window.level = .statusBar           // float above normal windows

        // Position above the dock
        positionAboveDock()
        setupHEVCPlayer()
    }

    private func positionAboveDock() {
        guard let screen = NSScreen.main else { return }
        let dockHeight = getDockHeight(for: screen)
        let charSize = CGSize(width: 80, height: 80)

        // Start at left edge, above dock
        let origin = CGPoint(
            x: 100,
            y: dockHeight + 4
        )
        window?.setFrame(CGRect(origin: origin, size: charSize), display: true)
    }

    private func getDockHeight(for screen: NSScreen) -> CGFloat {
        // visibleFrame excludes dock and menu bar
        let visible = screen.visibleFrame
        let full    = screen.frame
        return visible.minY - full.minY   // bottom inset = dock height
    }

    private func setupHEVCPlayer() {
        guard let url = Bundle.main.url(forResource: "bruce-walk",
                                        withExtension: "mov") else { return }
        player = AVPlayer(url: url)
        player.actionAtItemEnd = .none  // we loop manually

        NotificationCenter.default.addObserver(
            self,
            selector: #selector(playerDidReachEnd),
            name: .AVPlayerItemDidPlayToEndTime,
            object: player.currentItem
        )

        playerView = AVPlayerView(frame: window!.contentView!.bounds)
        playerView.player = player
        playerView.videoGravity = .resizeAspect
        playerView.controlsStyle = .none
        window?.contentView?.addSubview(playerView)
        player.play()
    }

    @objc private func playerDidReachEnd(_ notification: Notification) {
        player.seek(to: .zero)
        player.play()
    }
}
```

### Shelling Out to AI CLIs

```swift
// AgentProcess.swift pattern
import Foundation

enum AIBackend: String, CaseIterable {
    case claude  = "claude"
    case codex   = "codex"
    case copilot = "gh"     // `gh copilot suggest`

    var executablePath: String {
        // Resolve from common install locations
        let candidates: [String]
        switch self {
        case .claude:
            candidates = ["/usr/local/bin/claude", "/opt/homebrew/bin/claude"]
        case .codex:
            candidates = ["/usr/local/bin/codex", "/opt/homebrew/bin/codex",
                          "\(NSHomeDirectory())/.npm-global/bin/codex"]
        case .copilot:
            candidates = ["/usr/local/bin/gh", "/opt/homebrew/bin/gh"]
        }
        return candidates.first { FileManager.default.fileExists(atPath: $0) }
               ?? "/usr/local/bin/\(rawValue)"
    }

    func buildArguments(for prompt: String) -> [String] {
        switch self {
        case .claude:
            return ["-p", prompt]
        case .codex:
            return ["-p", prompt]
        case .copilot:
            return ["copilot", "suggest", "-t", "shell", prompt]
        }
    }
}

class AgentProcess {

    var onOutput: ((String) -> Void)?
    var onComplete: (() -> Void)?
    var onError: ((String) -> Void)?

    private var process: Process?

    func run(prompt: String, backend: AIBackend) {
        let proc = Process()
        proc.executableURL = URL(fileURLWithPath: backend.executablePath)
        proc.arguments = backend.buildArguments(for: prompt)

        // Inherit a PATH that includes Homebrew and npm globals
        var env = ProcessInfo.processInfo.environment
        env["PATH"] = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:"
                    + "\(NSHomeDirectory())/.npm-global/bin:"
                    + (env["PATH"] ?? "")
        proc.environment = env

        let pipe = Pipe()
        proc.standardOutput = pipe
        proc.standardError  = pipe

        pipe.fileHandleForReading.readabilityHandler = { [weak self] handle in
            let data = handle.availableData
            guard !data.isEmpty,
                  let text = String(data: data, encoding: .utf8) else { return }
            DispatchQueue.main.async { self?.onOutput?(text) }
        }

        proc.terminationHandler = { [weak self] _ in
            DispatchQueue.main.async { self?.onComplete?() }
        }

        do {
            try proc.run()
            process = proc
        } catch {
            onError?("Failed to launch \(backend.rawValue): \(error.localizedDescription)")
        }
    }

    func stop() {
        process?.terminate()
        process = nil
    }
}
```

### Settings Manager

```swift
// SettingsManager.swift pattern
import Foundation

enum TerminalTheme: String, CaseIterable {
    case peach    = "Peach"
    case midnight = "Midnight"
    case cloud    = "Cloud"
    case moss     = "Moss"
}

class SettingsManager {

    static let shared = SettingsManager()
    private let defaults = UserDefaults.standard

    private enum Keys {
        static let selectedBackend = "selectedBackend"
        static let selectedTheme   = "selectedTheme"
        static let soundEnabled    = "soundEnabled"
        static let activeCharacter = "activeCharacter"
    }

    var selectedBackend: AIBackend {
        get {
            let raw = defaults.string(forKey: Keys.selectedBackend) ?? AIBackend.claude.rawValue
            return AIBackend(rawValue: raw) ?? .claude
        }
        set { defaults.set(newValue.rawValue, forKey: Keys.selectedBackend) }
    }

    var selectedTheme: TerminalTheme {
        get {
            let raw = defaults.string(forKey: Keys.selectedTheme) ?? TerminalTheme.peach.rawValue
            return TerminalTheme(rawValue: raw) ?? .peach
        }
        set { defaults.set(newValue.rawValue, forKey: Keys.selectedTheme) }
    }

    var soundEnabled: Bool {
        get { defaults.object(forKey: Keys.soundEnabled) as? Bool ?? true }
        set { defaults.set(newValue, forKey: Keys.soundEnabled) }
    }

    var activeCharacter: String {
        get { defaults.string(forKey: Keys.activeCharacter) ?? "bruce" }
        set { defaults.set(newValue, forKey: Keys.activeCharacter) }
    }
}
```

### Menubar Integration (AppDelegate)

```swift
// AppDelegate.swift pattern
import AppKit
import Sparkle

@main
class AppDelegate: NSObject, NSApplicationDelegate {

    private var statusItem: NSStatusItem!
    private var updaterController: SPUStandardUpdaterController!

    func applicationDidFinishLaunching(_ notification: Notification) {
        // Hide from Dock — lives only in menu bar + dock overlay
        NSApp.setActivationPolicy(.accessory)

        setupMenuBar()
        setupSparkle()
        launchCharacterWindows()
        showOnboardingIfNeeded()
    }

    private func setupMenuBar() {
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
        statusItem.button?.title = "🐾"

        let menu = NSMenu()

        // Backend submenu
        let backendMenu = NSMenu()
        for backend in AIBackend.allCases {
            let item = NSMenuItem(
                title: backend.rawValue.capitalized,
                action: #selector(selectBackend(_:)),
                keyEquivalent: ""
            )
            item.representedObject = backend
            item.state = backend == SettingsManager.shared.selectedBackend ? .on : .off
            backendMenu.addItem(item)
        }
        let backendItem = NSMenuItem(title: "AI Backend", action: nil, keyEquivalent: "")
        menu.addItem(backendItem)
        menu.setSubmenu(backendMenu, for: backendItem)

        // Theme submenu
        let themeMenu = NSMenu()
        for theme in TerminalTheme.allCases {
            let item = NSMenuItem(
                title: theme.rawValue,
                action: #selector(selectTheme(_:)),
                keyEquivalent: ""
            )
            item.representedObject = theme
            item.state = theme == SettingsManager.shared.selectedTheme ? .on : .off
            themeMenu.addItem(item)
        }
        let themeItem = NSMenuItem(title: "Theme", action: nil, keyEquivalent: "")
        menu.addItem(themeItem)
        menu.setSubmenu(themeMenu, for: themeItem)

        menu.addItem(.separator())
        menu.addItem(NSMenuItem(title: "Check for Updates…",
                                action: #selector(checkForUpdates),
                                keyEquivalent: ""))
        menu.addItem(.separator())
        menu.addItem(NSMenuItem(title: "Quit lil agents",
                                action: #selector(NSApplication.terminate(_:)),
                                keyEquivalent: "q"))
        statusItem.menu = menu
    }

    @objc private func selectBackend(_ sender: NSMenuItem) {
        guard let backend = sender.representedObject as? AIBackend else { return }
        SettingsManager.shared.selectedBackend = backend
        // Rebuild menu to update checkmarks
        setupMenuBar()
    }

    @objc private func selectTheme(_ sender: NSMenuItem) {
        guard let theme = sender.representedObject as? TerminalTheme else { return }
        SettingsManager.shared.selectedTheme = theme
        setupMenuBar()
    }

    @objc private func checkForUpdates() {
        updaterController.checkForUpdates(nil)
    }

    private func setupSparkle() {
        updaterController = SPUStandardUpdaterController(
            startingUpdater: true,
            updaterDelegate: nil,
            userDriverDelegate: nil
        )
    }

    private func launchCharacterWindows() {
        // Instantiate one window per character
        CharacterWindowController.show(character: "bruce")
        CharacterWindowController.show(character: "jazz")
    }

    private func showOnboardingIfNeeded() {
        let key = "hasSeenOnboarding"
        guard !UserDefaults.standard.bool(forKey: key) else { return }
        OnboardingWindowController.show()
        UserDefaults.standard.set(true, forKey: key)
    }
}
```

---

## Adding a New Theme

1. Define colors in `TerminalTheme.swift`:

```swift
extension TerminalTheme {
    var backgroundColor: NSColor {
        switch self {
        case .peach:    return NSColor(hex: "#FFF0E6")
        case .midnight: return NSColor(hex: "#0D1117")
        case .cloud:    return NSColor(hex: "#F0F4F8")
        case .moss:     return NSColor(hex: "#1A2E1A")
        // Add new theme here:
        // case .ocean: return NSColor(hex: "#0A1628")
        }
    }

    var foregroundColor: NSColor {
        switch self {
        case .peach:    return NSColor(hex: "#3D2000")
        case .midnight: return NSColor(hex: "#E6EDF3")
        case .cloud:    return NSColor(hex: "#24292E")
        case .moss:     return NSColor(hex: "#B5D6A7")
        }
    }

    var cursorColor: NSColor {
        switch self {
        case .peach:    return NSColor(hex: "#FF8C42")
        case .midnight: return NSColor(hex: "#58A6FF")
        case .cloud:    return NSColor(hex: "#0366D6")
        case .moss:     return NSColor(hex: "#57AB5A")
        }
    }
}
```

2. Add the case to the `TerminalTheme` enum and `allCases` if not auto-synthesised.
3. The menubar submenu rebuilds dynamically — no further changes needed.

---

## Adding a New Character

1. Export transparent HEVC `.mov` files for states: `walk`, `think`, `idle`.
   - Use HEVC with alpha (`.mov`, ProRes 4444 or HEVC with alpha channel).
   - Recommended size: 80×80pt @2x = 160×160px.

2. Add files to `Resources/<character-name>/` in Xcode (copy items, add to target).

3. Register the character:

```swift
// In CharacterRegistry.swift (create if needed)
enum Character: String, CaseIterable {
    case bruce = "bruce"
    case jazz  = "jazz"
    case nova  = "nova"   // new character

    func videoURL(for state: CharacterState) -> URL? {
        Bundle.main.url(forResource: "\(rawValue)-\(state.rawValue)",
                        withExtension: "mov")
    }
}

enum CharacterState: String {
    case walk  = "walk"
    case think = "think"
    case idle  = "idle"
}
```

4. Add the character to the menubar selection and persist via `SettingsManager`.

---

## Common Patterns

### Streaming Output to a Text View

```swift
// In TerminalPopoverController.swift
func appendOutput(_ text: String) {
    let attributed = NSAttributedString(
        string: text,
        attributes: [
            .font: NSFont.monospacedSystemFont(ofSize: 12, weight: .regular),
            .foregroundColor: SettingsManager.shared.selectedTheme.foregroundColor
        ]
    )
    textView.textStorage?.append(attributed)
    // Auto-scroll
    textView.scrollToEndOfDocument(nil)
}
```

### Playing a Completion Sound

```swift
import AppKit

func playCompletionSound() {
    guard SettingsManager.shared.soundEnabled else { return }
    guard let url = Bundle.main.url(forResource: "complete", withExtension: "aiff") else { return }
    NSSound(contentsOf: url, byReference: false)?.play()
}
```

### Thinking Bubble Phrases

```swift
let thinkingPhrases = [
    "hmm...",
    "on it 🤔",
    "cooking...",
    "big brain time",
    "processing vibes",
    "almost there...",
    "asking the AI gods",
]

func randomThinkingPhrase() -> String {
    thinkingPhrases.randomElement() ?? "thinking..."
}
```

---

## Troubleshooting

### Characters don't appear above the dock

- Ensure the app has **Screen Recording** or **Accessibility** permission if you added features that require it (base lil-agents does not, but custom extensions might).
- Check that `NSWindow.level = .statusBar` is set — lower levels may go behind the dock.
- If using multiple monitors, verify `positionAboveDock()` queries `NSScreen.screens` rather than only `.main`.

### CLI not found / agent doesn't respond

```bash
# Verify CLI is on PATH
which claude   # /opt/homebrew/bin/claude
which codex    # ~/.npm-global/bin/codex
which gh       # /opt/homebrew/bin/gh

# Test CLI directly
claude -p "say hello"
codex -p "say hello"
gh copilot suggest -t shell "list files"
```

- In `AgentProcess.swift`, extend the `PATH` in the subprocess environment to include `/opt/homebrew/bin` and `~/.npm-global/bin`.
- Make sure the CLI is authenticated (`claude auth`, `codex login`, `gh auth login`).

### HEVC video shows black box instead of transparency

- The `.mov` must be exported with an **alpha channel** (ProRes 4444 or HEVC with alpha).
- Set `AVPlayerView.videoGravity = .resizeAspect` and ensure the window's `contentView` has `wantsLayer = true` with `layer?.backgroundColor = .clear`.

```swift
window?.contentView?.wantsLayer = true
window?.contentView?.layer?.backgroundColor = CGColor.clear
```

### Sparkle update check fails in debug builds

- Sparkle requires a valid `SUFeedURL` in `Info.plist`. For local development, you can disable the updater:

```swift
// In AppDelegate, skip Sparkle during debug
#if !DEBUG
setupSparkle()
#endif
```

### App activates as a dock icon (unwanted)

Ensure `Info.plist` contains:

```xml
<key>LSUIElement</key>
<true/>
```

And in `applicationDidFinishLaunching`:

```swift
NSApp.setActivationPolicy(.accessory)
```

---

## Info.plist Key Reference

| Key | Value | Purpose |
|---|---|---|
| `LSUIElement` | `YES` | Hide from Dock, run as background/menu-bar app |
| `SUFeedURL` | `https://your-host/appcast.xml` | Sparkle update feed |
| `NSMicrophoneUsageDescription` | — | Not needed unless you add voice |
| `NSAppleEventsUsageDescription` | — | Not needed for base app |

---

## License

MIT — see [LICENSE](https://github.com/ryanstephen/lil-agents/blob/main/LICENSE).
```
