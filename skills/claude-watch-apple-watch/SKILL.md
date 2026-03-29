```markdown
---
name: claude-watch-apple-watch
description: Control Claude Code AI coding sessions from your Apple Watch — stream terminal output, approve permissions, and send voice commands from your wrist
triggers:
  - "set up claude watch on my apple watch"
  - "control claude code from my watch"
  - "stream claude terminal output to watch"
  - "approve claude permissions from apple watch"
  - "connect claude code to apple watch"
  - "set up agent watch bridge server"
  - "install claude code hooks for watch"
  - "voice commands to claude from wrist"
---

# Claude Watch (Agent Watch)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Control Claude Code from your Apple Watch. Stream live terminal output, approve/deny permission prompts, answer dynamic questions, and send voice commands — all from your wrist.

## Architecture

```
Apple Watch  <==WCSession==>  iPhone  <==HTTP/SSE==>  Bridge Server (Mac)
 (SwiftUI)                   (Relay)                   (Node.js)
                                                            |
                                                  HTTP Hooks | PTY stdin
                                                            v
                                                   Claude Code Session
```

Three components:
- **Bridge Server** — Node.js on Mac, receives Claude Code hooks, streams via SSE, blocks on permission requests
- **iPhone App** — SwiftUI, discovers bridge via Bonjour, relays events to watch via WCSession
- **watchOS App** — SwiftUI, connects directly to bridge over Wi-Fi, renders terminal + permissions

## Installation

### Prerequisites
- macOS 13+, Node.js 18+, Xcode 16+
- Apple Watch (watchOS 10+) on **same Wi-Fi** as Mac
- Claude Code CLI 2.1+
- Apple Watch: **Settings > Wi-Fi > your network > Private Wi-Fi Address → Off**

### Step 1: Install Bridge

```bash
git clone https://github.com/shobhit99/claude-watch
cd claude-watch/skill/bridge
npm install
```

### Step 2: Install Claude Code Hooks

```bash
./skill/setup-hooks.sh
```

This writes to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "type": "http", "url": "http://127.0.0.1:7860/hook/post-tool-use" }
    ],
    "PreToolUse": [
      { "type": "http", "url": "http://127.0.0.1:7860/hook/pre-tool-use" }
    ],
    "PermissionRequest": [
      { "type": "http", "url": "http://127.0.0.1:7860/hook/permission-request", "timeout": 600 }
    ],
    "Stop": [
      { "type": "http", "url": "http://127.0.0.1:7860/hook/stop" }
    ]
  }
}
```

To remove hooks later:
```bash
./skill/setup-hooks.sh --remove
```

### Step 3: Start Bridge Server

```bash
cd skill/bridge
node server.js
```

Output:
```
╔═══════════════════════════════════════╗
║        AGENT WATCH BRIDGE             ║
╠═══════════════════════════════════════╣
║  Pairing Code:  648505                ║
║  IP Address:    192.168.1.4           ║
║  Port:          7860                  ║
╚═══════════════════════════════════════╝
```

### Step 4: Build iOS + watchOS Apps

```bash
cd ios/ClaudeWatch
xcodegen generate
open ClaudeWatch.xcodeproj
```

In Xcode:
1. Set **Development Team** on both targets: `ClaudeWatch` and `ClaudeWatchWatch`
2. Run `ClaudeWatch` scheme → iPhone
3. Run `ClaudeWatchWatch` scheme → Apple Watch

### Step 5: Pair

- **iPhone**: Enter 6-digit code from bridge banner
- **Watch**: Auto-discovers via Bonjour; fallback: enter IP manually

## Key Commands

| Command | Description |
|---------|-------------|
| `node server.js` | Start bridge (port 7860, auto-increments to 7869) |
| `./skill/setup-hooks.sh` | Install Claude Code hooks globally |
| `./skill/setup-hooks.sh --remove` | Remove hooks |
| `xcodegen generate` | Regenerate Xcode project from `project.yml` |
| `curl http://127.0.0.1:7860/status` | Check bridge health |

## Bridge Server API

### HTTP Endpoints

```
GET  /status                  — Health check + connection counts
GET  /events                  — SSE stream (requires session token)
POST /hook/post-tool-use      — Claude tool use events (async)
POST /hook/pre-tool-use       — Claude pre-tool events (async)
POST /hook/permission-request — Permission prompts (BLOCKING, 10min timeout)
POST /hook/stop               — Claude session ended (async)
POST /approve                 — Submit permission decision from watch/phone
POST /pair                    — Pair with 6-digit code, returns session token
```

### SSE Event Types

```javascript
// Terminal output event
{
  "type": "tool-use",
  "tool": "Edit",           // Read | Edit | Bash | Grep | Write
  "path": "src/index.ts",
  "summary": "Editing src/index.ts"
}

// Permission request (watch shows approval sheet)
{
  "type": "permission-request",
  "id": "req_abc123",
  "question": "Do you want to edit this file?",
  "options": ["Yes", "Yes to all", "No"]
}

// AskUserQuestion (dynamic options)
{
  "type": "permission-request",
  "id": "req_xyz456",
  "question": "Which approach should I take?",
  "options": [
    { "label": "Refactor", "description": "Clean up existing code" },
    { "label": "Rewrite", "description": "Start fresh" }
  ]
}

// Session ended
{ "type": "stop", "exitCode": 0 }
```

### Pairing Flow

```swift
// iPhone BridgeClient.swift pattern
func pair(code: String) async throws -> String {
    let response = try await post("/pair", body: ["code": code])
    return response["sessionToken"] as! String
}

// All subsequent requests include session token
func streamEvents(token: String) -> AsyncStream<SSEEvent> {
    let url = URL(string: "\(bridgeURL)/events?token=\(token)")!
    return SSEClient(url: url).stream()
}
```

## Swift Code Examples

### Watch: Connecting to Bridge (Direct Wi-Fi)

```swift
// WatchBridgeClient.swift
import Foundation

class WatchBridgeClient: ObservableObject {
    private var sseTask: URLSessionDataTask?
    private let session = URLSession.shared
    
    func connect(to bridgeURL: URL, token: String) {
        let eventsURL = bridgeURL
            .appendingPathComponent("events")
            .appending(queryItems: [URLQueryItem(name: "token", value: token)])
        
        var request = URLRequest(url: eventsURL)
        request.setValue("text/event-stream", forHTTPHeaderField: "Accept")
        
        sseTask = session.dataTask(with: request) { [weak self] data, _, _ in
            guard let data, let text = String(data: data, encoding: .utf8) else { return }
            self?.parseSSEEvents(text)
        }
        sseTask?.resume()
    }
    
    func submitApproval(bridgeURL: URL, token: String, requestID: String, decision: String) async throws {
        var request = URLRequest(url: bridgeURL.appendingPathComponent("approve"))
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONSerialization.data(withJSONObject: [
            "id": requestID,
            "decision": decision,
            "token": token
        ])
        let (_, _) = try await URLSession.shared.data(for: request)
    }
}
```

### Watch: Permission Approval View

```swift
// ApprovalView.swift
import SwiftUI
import WatchKit

struct ApprovalView: View {
    let request: ApprovalRequest
    let onDecision: (String) -> Void
    
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 8) {
                Text(request.question)
                    .font(.headline)
                    .padding(.bottom, 4)
                
                ForEach(request.options, id: \.self) { option in
                    Button(action: {
                        WKInterfaceDevice.current().play(.click)
                        onDecision(option)
                    }) {
                        Text(option)
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 6)
                    }
                    .buttonStyle(.bordered)
                    .tint(option.lowercased().contains("no") ? .red : .green)
                }
            }
            .padding()
        }
        .navigationTitle("Permission")
        .onAppear {
            WKInterfaceDevice.current().play(.notification)
        }
    }
}
```

### Watch: Voice Command Input

```swift
// VoiceInputView.swift
import SwiftUI
import WatchKit

struct VoiceInputView: View {
    @StateObject private var speech = SpeechService()
    @Environment(\.dismiss) var dismiss
    let onCommand: (String) -> Void
    
    var body: some View {
        VStack(spacing: 12) {
            if speech.isRecording {
                Image(systemName: "waveform")
                    .font(.largeTitle)
                    .foregroundColor(.blue)
                    .symbolEffect(.variableColor)
                Text("Listening...")
                    .foregroundColor(.secondary)
            } else if let result = speech.transcription {
                Text(result)
                    .multilineTextAlignment(.center)
                Button("Send") {
                    onCommand(result)
                    dismiss()
                }
                .buttonStyle(.borderedProminent)
            }
        }
        .onAppear { speech.startDictation() }
    }
}
```

### iPhone: Bonjour Discovery

```swift
// BonjourDiscovery.swift
import Foundation

class BonjourDiscovery: NSObject, ObservableObject, NetServiceBrowserDelegate, NetServiceDelegate {
    @Published var discoveredBridgeURL: URL?
    private let browser = NetServiceBrowser()
    private var resolving: NetService?
    
    func startDiscovery() {
        browser.delegate = self
        browser.searchForServices(ofType: "_agentwatch._tcp.", inDomain: "local.")
    }
    
    func netServiceBrowser(_ browser: NetServiceBrowser, didFind service: NetService, moreComing: Bool) {
        resolving = service
        service.delegate = self
        service.resolve(withTimeout: 5)
    }
    
    func netServiceDidResolveAddress(_ sender: NetService) {
        guard let host = sender.hostName else { return }
        let url = URL(string: "http://\(host):\(sender.port)")!
        DispatchQueue.main.async { self.discoveredBridgeURL = url }
    }
}
```

### iPhone: Relay to Watch via WCSession

```swift
// RelayService.swift
import WatchConnectivity

class RelayService: NSObject, WCSessionDelegate {
    static let shared = RelayService()
    private let session = WCSession.default
    
    func setup() {
        guard WCSession.isSupported() else { return }
        session.delegate = self
        session.activate()
    }
    
    func relayEvent(_ event: [String: Any]) {
        guard session.isReachable else {
            // Fallback: transferUserInfo for non-urgent events
            session.transferUserInfo(event)
            return
        }
        session.sendMessage(event, replyHandler: nil)
    }
    
    func relayPermissionRequest(_ request: ApprovalRequest) {
        let payload: [String: Any] = [
            "type": "permission-request",
            "id": request.id,
            "question": request.question,
            "options": request.options
        ]
        session.sendMessage(payload, replyHandler: nil)
    }
}
```

## Shared Models

```swift
// ApprovalRequest.swift
struct ApprovalRequest: Identifiable, Codable {
    let id: String
    let question: String
    let options: [String]
    var timestamp: Date = .now
}

// TerminalLine.swift
struct TerminalLine: Identifiable {
    let id = UUID()
    let tool: String      // "Read" | "Edit" | "Bash" | "Grep" | "Write"
    let path: String?
    let summary: String
    let timestamp: Date
    
    var icon: String {
        switch tool {
        case "Read": return "doc.text"
        case "Edit": return "pencil"
        case "Bash": return "terminal"
        case "Grep": return "magnifyingglass"
        case "Write": return "square.and.pencil"
        default: return "bolt"
        }
    }
}

// WatchMessage.swift — shared over WCSession
enum WatchMessageType: String, Codable {
    case terminalLine = "terminal-line"
    case permissionRequest = "permission-request"
    case sessionStopped = "session-stopped"
    case connectionStatus = "connection-status"
}
```

## Configuration

### Bridge Server Environment

```bash
PORT=7860 node server.js   # Override default port (tries 7860-7869)
```

### project.yml (XcodeGen) Key Settings

```yaml
# ios/ClaudeWatch/project.yml
targets:
  ClaudeWatch:
    type: application
    platform: iOS
    deploymentTarget: "17.0"
    dependencies:
      - target: ClaudeWatchWatch
    
  ClaudeWatchWatch:
    type: application
    platform: watchOS
    deploymentTarget: "10.0"
```

## Hook Event Payload Examples

### PostToolUse (Edit file)
```json
{
  "event": "PostToolUse",
  "tool_name": "Edit",
  "tool_input": { "file_path": "src/index.ts", "old_string": "...", "new_string": "..." },
  "tool_result": { "success": true }
}
```

### PermissionRequest (blocking)
```json
{
  "event": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf dist/" },
  "question": "Allow running: rm -rf dist/?",
  "options": ["Yes", "Yes to all", "No"]
}
```

Bridge **blocks** responding to this hook until user approves via watch/phone (up to 10 minutes).

## Troubleshooting

### "Bridge not found" on Watch
```bash
# Verify bridge is running
curl http://127.0.0.1:7860/status

# Check watch Wi-Fi — must match Mac's network
# Disable Private Wi-Fi Address on watch:
# Settings > Wi-Fi > [Network] > Private Wi-Fi Address → Off

# Use manual IP entry in watch app if Bonjour fails
```

### Permission prompts not appearing
```bash
# Verify hooks are installed
cat ~/.claude/settings.json | grep -A 5 "PermissionRequest"

# Watch bridge logs for hook receipt
# Should see: "Hook: PermissionRequest received"

# Check watch shows green status dot
```

### "unsupported architecture" build error
```
Xcode: Product > Clean Build Folder (Cmd+Shift+Option+K)
Select scheme: ClaudeWatchWatch (not ClaudeWatch)
Deploy via paired iPhone destination if direct watch deployment fails
```

### iPhone "Connection failed"
```bash
# Bridge must be on same LAN as iPhone
curl http://192.168.1.4:7860/status   # Use bridge's actual IP

# Check firewall allows port 7860
sudo lsof -i :7860
```

### Hooks fire but watch doesn't update
```bash
# Test SSE stream directly
curl -N "http://127.0.0.1:7860/events?token=YOUR_SESSION_TOKEN"

# Check WCSession is active on iPhone
# iPhone app must be open or backgrounded (not force-quit)
```

### Bridge exits immediately
The bridge waits for Claude Code hooks — it does **not** spawn Claude itself.
Start Claude Code in a separate terminal; hooks forward events automatically:
```bash
# Terminal 1: Bridge
node server.js

# Terminal 2: Claude Code (hooks auto-fire)
claude
```

## Uninstall / Reset

```bash
# Remove Claude Code hooks
./skill/setup-hooks.sh --remove

# Unpair iPhone: Settings > Forget Mac (in app)
# Unpair Watch: Restart app (credentials clear when bridge restarts)
```

## Requirements Summary

| Component | Minimum |
|-----------|---------|
| macOS | 13.0+ |
| Node.js | 18+ |
| Xcode | 16+ |
| iOS | 17.0 |
| watchOS | 10.0 |
| Claude Code | 2.1+ |
```
