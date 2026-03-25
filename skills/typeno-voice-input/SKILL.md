```markdown
---
name: typeno-voice-input
description: TypeNo is a free, open source, privacy-first macOS menu bar app that captures voice via a Control key shortcut, transcribes locally using the coli speech engine, and pastes text into the active app.
triggers:
  - set up TypeNo voice input on macOS
  - add voice dictation to my Mac app
  - integrate TypeNo speech to text
  - build TypeNo from source
  - configure local voice transcription macOS
  - use TypeNo to paste voice input
  - fix TypeNo microphone or accessibility permissions
  - drag audio file to transcribe with TypeNo
---

# TypeNo Voice Input macOS Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

TypeNo is a minimal, privacy-first macOS menu bar app that records your voice on a Control key short-press, transcribes it locally via the [coli](https://github.com/marswaveai/coli) Node.js engine, and pastes the resulting text directly into whatever app is focused — no cloud, no accounts, no UI windows.

---

## What TypeNo Does

- **Global hotkey:** Short-press `Control` (< 300 ms, no modifier) → start/stop recording
- **Local transcription:** All speech processing runs on-device via the `coli` CLI
- **Auto-paste:** Transcribed text is typed into the active app and copied to the clipboard
- **Drag-to-transcribe:** Drop `.m4a`, `.mp3`, `.wav`, or `.aac` onto the menu bar icon
- **No preferences UI:** Zero configuration by design

---

## Installation

### 1. Install the coli speech engine

TypeNo shells out to `coli` for transcription. Install it globally:

```bash
npm install -g @marswave/coli
```

Verify:

```bash
coli --version
```

If `coli` is missing at launch, TypeNo shows an in-app prompt with the install command.

### 2a. Download the pre-built app

```bash
# Download latest release zip from GitHub
# https://github.com/marswaveai/TypeNo/releases/latest
# Download TypeNo.app.zip, unzip, move to /Applications
open /Applications/TypeNo.app
```

If macOS quarantines the app (not yet notarized):

```bash
xattr -dr com.apple.quarantine "/Applications/TypeNo.app"
open /Applications/TypeNo.app
```

Or right-click → **Open** in Finder, then **Open Anyway** in System Settings → Privacy & Security.

### 2b. Build from source

```bash
git clone https://github.com/marswaveai/TypeNo.git
cd TypeNo
scripts/generate_icon.sh   # generates app icon assets
scripts/build_app.sh       # compiles and bundles to dist/TypeNo.app
cp -R dist/TypeNo.app /Applications/
open /Applications/TypeNo.app
```

### 3. Grant one-time permissions

On first launch TypeNo requests:
- **Microphone** — for audio capture
- **Accessibility** — to simulate paste (⌘V) into the active app

Grant both in **System Settings → Privacy & Security**.

---

## Usage

| Action | How |
|---|---|
| Start recording | Short-press `Control` (< 300 ms, no other keys held) |
| Stop recording & transcribe | Short-press `Control` again |
| Start/stop via menu | Menu bar icon → **Record** |
| Transcribe an audio file | Drag `.m4a` / `.mp3` / `.wav` / `.aac` onto the menu bar icon |
| Check for updates | Menu bar icon → **Check for Updates...** |
| Quit | Menu bar icon → **Quit** (`⌘Q`) |

After stopping, TypeNo:
1. Sends audio to `coli` for local transcription
2. Pastes result into the previously focused app
3. Copies result to the clipboard

---

## Project Structure (Swift source)

```
TypeNo/
├── AppDelegate.swift          # NSApplicationDelegate, menu bar setup
├── AudioRecorder.swift        # AVFoundation capture logic
├── TranscriptionService.swift # Shells out to `coli` CLI
├── PasteService.swift         # Accessibility / CGEvent paste
├── HotkeyMonitor.swift        # Global CGEventTap for Control key
├── StatusBarController.swift  # NSStatusItem, menu construction
├── DragDropHandler.swift      # Drag audio files onto status icon
└── assets/
    └── hero.webp
scripts/
├── generate_icon.sh
└── build_app.sh
```

---

## Key Swift Patterns

### Global Control-key hotkey (CGEventTap)

```swift
// HotkeyMonitor.swift
import Cocoa

class HotkeyMonitor {
    private var eventTap: CFMachPort?
    private var pressStart: Date?
    private let maxPressDuration: TimeInterval = 0.3   // 300 ms

    func start(onToggle: @escaping () -> Void) {
        let mask: CGEventMask = (1 << CGEventType.flagsChanged.rawValue)
        eventTap = CGEvent.tapCreate(
            tap: .cgSessionEventTap,
            place: .headInsertEventTap,
            options: .defaultTap,
            eventsOfInterest: mask,
            callback: { proxy, type, event, refcon in
                let monitor = Unmanaged<HotkeyMonitor>
                    .fromOpaque(refcon!).takeUnretainedValue()
                monitor.handle(event: event)
                return Unmanaged.passUnretained(event)
            },
            userInfo: Unmanaged.passUnretained(self).toOpaque()
        )
        guard let tap = eventTap else { return }
        let loop = CFMachPortCreateRunLoopSource(nil, tap, 0)
        CFRunLoopAddSource(CFRunLoopGetCurrent(), loop, .commonModes)
        CGEvent.tapEnable(tap: tap, enable: true)
    }

    private func handle(event: CGEvent) {
        let flags = event.flags
        let onlyControl = flags.contains(.maskControl) &&
                          !flags.contains(.maskCommand) &&
                          !flags.contains(.maskAlternate) &&
                          !flags.contains(.maskShift)

        if onlyControl {
            pressStart = Date()
        } else if let start = pressStart {
            let duration = Date().timeIntervalSince(start)
            pressStart = nil
            if duration < maxPressDuration {
                DispatchQueue.main.async { self.onToggle?() }
            }
        }
    }

    var onToggle: (() -> Void)?
}
```

### Audio recording with AVFoundation

```swift
// AudioRecorder.swift
import AVFoundation

class AudioRecorder: NSObject {
    private var engine = AVAudioEngine()
    private var file: AVAudioFile?
    private(set) var outputURL: URL?

    func startRecording() throws {
        let dir = FileManager.default.temporaryDirectory
        let url = dir.appendingPathComponent(UUID().uuidString + ".wav")
        outputURL = url

        let input = engine.inputNode
        let format = input.outputFormat(forBus: 0)

        file = try AVAudioFile(forWriting: url,
                               settings: format.settings)

        input.installTap(onBus: 0, bufferSize: 4096, format: format) { [weak self] buffer, _ in
            try? self?.file?.write(from: buffer)
        }
        try engine.start()
    }

    func stopRecording() -> URL? {
        engine.inputNode.removeTap(onBus: 0)
        engine.stop()
        file = nil
        return outputURL
    }
}
```

### Transcription via coli CLI

```swift
// TranscriptionService.swift
import Foundation

class TranscriptionService {
    /// Transcribe a local audio file using the coli CLI.
    /// - Parameter url: Path to .wav / .m4a / .mp3 / .aac file
    /// - Returns: Transcribed string, or nil on failure
    func transcribe(url: URL) async throws -> String {
        return try await withCheckedThrowingContinuation { continuation in
            let process = Process()
            process.executableURL = coliExecutableURL()
            process.arguments = ["transcribe", url.path]

            let pipe = Pipe()
            process.standardOutput = pipe
            process.standardError = Pipe()

            process.terminationHandler = { _ in
                let data = pipe.fileHandleForReading.readDataToEndOfFile()
                let result = String(data: data, encoding: .utf8)?
                    .trimmingCharacters(in: .whitespacesAndNewlines) ?? ""
                if result.isEmpty {
                    continuation.resume(throwing: TranscriptionError.emptyResult)
                } else {
                    continuation.resume(returning: result)
                }
            }
            do {
                try process.run()
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }

    private func coliExecutableURL() -> URL {
        // Check common npm global bin locations
        let candidates = [
            "/usr/local/bin/coli",
            "/opt/homebrew/bin/coli",
            (ProcessInfo.processInfo.environment["HOME"] ?? "") + "/.npm-global/bin/coli"
        ]
        for path in candidates {
            if FileManager.default.isExecutableFile(atPath: path) {
                return URL(fileURLWithPath: path)
            }
        }
        return URL(fileURLWithPath: "/usr/local/bin/coli") // fallback
    }
}

enum TranscriptionError: Error {
    case emptyResult
    case coliNotFound
}
```

### Pasting text via Accessibility

```swift
// PasteService.swift
import AppKit

class PasteService {
    func paste(text: String) {
        // 1. Write to clipboard
        NSPasteboard.general.clearContents()
        NSPasteboard.general.setString(text, forType: .string)

        // 2. Simulate ⌘V in the previously active app
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.05) {
            let source = CGEventSource(stateID: .combinedSessionState)
            let vKeyCode: CGKeyCode = 0x09  // kVK_ANSI_V

            let keyDown = CGEvent(keyboardEventSource: source,
                                  virtualKey: vKeyCode, keyDown: true)
            let keyUp   = CGEvent(keyboardEventSource: source,
                                  virtualKey: vKeyCode, keyDown: false)

            keyDown?.flags = .maskCommand
            keyUp?.flags   = .maskCommand

            keyDown?.post(tap: .cgAnnotatedSessionEventTap)
            keyUp?.post(tap: .cgAnnotatedSessionEventTap)
        }
    }
}
```

### Menu bar icon setup

```swift
// StatusBarController.swift
import AppKit

class StatusBarController {
    private let statusItem = NSStatusBar.system.statusItem(
        withLength: NSStatusItem.squareLength
    )

    init(recorder: AudioRecorder,
         transcriber: TranscriptionService,
         paster: PasteService) {
        setupIcon(recording: false)
        setupMenu(recorder: recorder, transcriber: transcriber, paster: paster)
    }

    func setupIcon(recording: Bool) {
        let name = recording ? "mic.fill" : "mic"
        statusItem.button?.image = NSImage(
            systemSymbolName: name,
            accessibilityDescription: recording ? "Recording" : "TypeNo"
        )
    }

    private func setupMenu(recorder: AudioRecorder,
                           transcriber: TranscriptionService,
                           paster: PasteService) {
        let menu = NSMenu()
        menu.addItem(NSMenuItem(title: "Record",
                                action: #selector(toggleRecord),
                                keyEquivalent: ""))
        menu.addItem(.separator())
        menu.addItem(NSMenuItem(title: "Check for Updates...",
                                action: #selector(checkForUpdates),
                                keyEquivalent: ""))
        menu.addItem(.separator())
        menu.addItem(NSMenuItem(title: "Quit TypeNo",
                                action: #selector(NSApplication.terminate(_:)),
                                keyEquivalent: "q"))
        statusItem.menu = menu
    }

    @objc private func toggleRecord() { /* delegate to AppDelegate */ }
    @objc private func checkForUpdates() { /* open releases URL */ }
}
```

### Drag-and-drop audio file onto status icon

```swift
// DragDropHandler.swift — attach to the NSStatusItem button's window
import AppKit

class DragDropHandler: NSView {
    var onFileDrop: ((URL) -> Void)?
    private let accepted: [String] = ["m4a", "mp3", "wav", "aac"]

    override func awakeFromNib() {
        super.awakeFromNib()
        registerForDraggedTypes([.fileURL])
    }

    override func draggingEntered(_ sender: NSDraggingInfo) -> NSDragOperation {
        guard let url = url(from: sender), accepted.contains(url.pathExtension.lowercased()) else {
            return []
        }
        return .copy
    }

    override func performDragOperation(_ sender: NSDraggingInfo) -> Bool {
        guard let url = url(from: sender) else { return false }
        onFileDrop?(url)
        return true
    }

    private func url(from info: NSDraggingInfo) -> URL? {
        info.draggingPasteboard
            .readObjects(forClasses: [NSURL.self], options: nil)?
            .first as? URL
    }
}
```

---

## Building & Releasing

```bash
# Generate app icon from source PNG
scripts/generate_icon.sh

# Build release app bundle to dist/TypeNo.app
scripts/build_app.sh

# Move to Applications for stable Accessibility permissions
cp -R dist/TypeNo.app /Applications/

# Remove quarantine (for unsigned/non-notarized builds)
xattr -dr com.apple.quarantine /Applications/TypeNo.app
```

---

## Troubleshooting

### "coli: command not found" at runtime

TypeNo resolves `coli` from known npm bin paths. If your Node setup uses a custom prefix:

```bash
npm config get prefix          # e.g. /Users/you/.npm-global
export PATH="$PATH:$(npm config get prefix)/bin"
# Add the above to ~/.zshrc or ~/.bash_profile
```

Then relaunch TypeNo.

### Microphone permission denied

```
System Settings → Privacy & Security → Microphone → enable TypeNo
```

If the toggle is missing, delete and reinstall from `/Applications`.

### Accessibility permission denied (paste doesn't work)

```
System Settings → Privacy & Security → Accessibility → enable TypeNo
```

After toggling, quit and relaunch TypeNo. Accessibility permissions are cached per bundle path — always run from `/Applications/TypeNo.app`, not `dist/`.

### macOS says the app is damaged

```bash
xattr -dr com.apple.quarantine "/Applications/TypeNo.app"
```

### Control key not triggering recording

- Ensure no other app has a global `Control` tap (e.g. Karabiner-Elements remapping)
- Check TypeNo has Accessibility permission — the CGEventTap requires it
- Short-press only: hold duration must be < 300 ms with no other modifier keys

### Audio file drag-drop not working

Accepted extensions: `.m4a`, `.mp3`, `.wav`, `.aac`. Other formats are silently rejected. Convert with:

```bash
ffmpeg -i input.ogg output.wav
```

---

## Key Design Constraints

- **No cloud:** All transcription is local via `coli`; no network calls are made
- **No preferences window:** The app is intentionally zero-config
- **No sandbox:** TypeNo uses `CGEvent.tapCreate` and Accessibility APIs that require a non-sandboxed entitlement — it cannot be distributed via the Mac App Store
- **GPL-3.0:** Any derivative work must also be GPL-3.0

---

## Links

- Homepage: [https://typeno.com](https://typeno.com)
- Releases: [https://github.com/marswaveai/TypeNo/releases](https://github.com/marswaveai/TypeNo/releases)
- coli engine: [https://github.com/marswaveai/coli](https://github.com/marswaveai/coli)
- License: GNU GPL v3.0
```
