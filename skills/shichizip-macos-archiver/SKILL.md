```markdown
---
name: shichizip-macos-archiver
description: ShichiZip is a 7-Zip derivative for macOS built in Swift, providing archive creation, extraction, and management with native macOS integration.
triggers:
  - "how do I build ShichiZip"
  - "compress files with ShichiZip"
  - "extract archives on macOS with 7-zip"
  - "ShichiZip zstandard variant"
  - "build 7-zip for macOS Swift"
  - "ShichiZip xcodebuild commands"
  - "integrate ShichiZip into my app"
  - "ShichiZip mainline vs zstandard fork"
---

# ShichiZip macOS Archiver

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ShichiZip is a native macOS archive manager derived from 7-Zip, written in Swift. It provides a macOS-native UI and supports standard 7-Zip archive formats, with an optional Zstandard-enhanced fork variant for improved compression.

---

## What ShichiZip Does

- Opens, creates, and extracts archives in formats supported by 7-Zip (7z, zip, tar, gz, bz2, xz, rar, etc.)
- Provides a native macOS SwiftUI/AppKit interface
- Offers two build variants:
  - **Mainline**: Standard 7-Zip backend
  - **ZS (Zstandard)**: Fork with Zstandard compression support

---

## Requirements

- macOS (Apple Silicon recommended; `arm64` used in official build commands)
- Xcode (with command-line tools)
- `make`
- Swift toolchain

---

## Building ShichiZip

### Step 1: Clone the Repository

```bash
git clone https://github.com/idawnlight/ShichiZip.git
cd ShichiZip
```

### Step 2: Build the Native Library

**Mainline variant:**
```bash
make lib-mainline
```

**Zstandard fork variant:**
```bash
make lib-zs
```

### Step 3: Build the App with Xcode

**Mainline app:**
```bash
xcodebuild \
  -project ShichiZip.xcodeproj \
  -scheme ShichiZip \
  -configuration Debug \
  -arch arm64 \
  build
```

**Zstandard variant app:**
```bash
xcodebuild \
  -project ShichiZip.xcodeproj \
  -scheme ShichiZipZS \
  -configuration Debug \
  -arch arm64 \
  build
```

### Build for Release

```bash
xcodebuild \
  -project ShichiZip.xcodeproj \
  -scheme ShichiZip \
  -configuration Release \
  -arch arm64 \
  build
```

---

## Project Structure

```
ShichiZip/
├── ShichiZip.xcodeproj/    # Xcode project
├── ShichiZip/              # Swift app source
│   ├── AppDelegate.swift   # App entry point
│   ├── ContentView.swift   # Main SwiftUI view
│   └── ...
├── Makefile                # Library build targets
└── README.md
```

---

## Schemes and Variants

| Scheme        | Library Target  | Description                        |
|---------------|-----------------|------------------------------------|
| `ShichiZip`   | `lib-mainline`  | Standard 7-Zip backend             |
| `ShichiZipZS` | `lib-zs`        | Zstandard-enhanced fork            |

Always match the `make` target to the Xcode scheme:
- `make lib-mainline` → `-scheme ShichiZip`
- `make lib-zs` → `-scheme ShichiZipZS`

---

## Swift Integration Patterns

### Opening an Archive (SwiftUI View Example)

```swift
import SwiftUI

struct ArchiveOpenView: View {
    @State private var selectedURL: URL? = nil
    @State private var showFilePicker = false

    var body: some View {
        Button("Open Archive") {
            showFilePicker = true
        }
        .fileImporter(
            isPresented: $showFilePicker,
            allowedContentTypes: [.archive, .zip],
            allowsMultipleSelection: false
        ) { result in
            switch result {
            case .success(let urls):
                selectedURL = urls.first
                if let url = selectedURL {
                    openArchive(at: url)
                }
            case .failure(let error):
                print("File picker error: \(error.localizedDescription)")
            }
        }
    }

    func openArchive(at url: URL) {
        // Pass URL to ShichiZip's archive handling layer
        print("Opening archive at: \(url.path)")
    }
}
```

### Triggering Archive Extraction via Process (Shell Bridge)

If integrating with the 7-Zip CLI binary bundled in the app:

```swift
import Foundation

func extract(archivePath: String, destinationPath: String) {
    guard let binaryURL = Bundle.main.url(
        forResource: "7zz",
        withExtension: nil,
        subdirectory: "Binaries"
    ) else {
        print("7zz binary not found in bundle")
        return
    }

    let process = Process()
    process.executableURL = binaryURL
    process.arguments = ["x", archivePath, "-o\(destinationPath)", "-y"]

    let outputPipe = Pipe()
    let errorPipe = Pipe()
    process.standardOutput = outputPipe
    process.standardError = errorPipe

    do {
        try process.run()
        process.waitUntilExit()

        let outputData = outputPipe.fileHandleForReading.readDataToEndOfFile()
        let output = String(data: outputData, encoding: .utf8) ?? ""
        print("Output: \(output)")

        if process.terminationStatus != 0 {
            let errorData = errorPipe.fileHandleForReading.readDataToEndOfFile()
            let error = String(data: errorData, encoding: .utf8) ?? ""
            print("Error: \(error)")
        }
    } catch {
        print("Failed to run extraction: \(error)")
    }
}
```

### Creating an Archive

```swift
func createArchive(inputPath: String, outputPath: String) {
    guard let binaryURL = Bundle.main.url(
        forResource: "7zz",
        withExtension: nil,
        subdirectory: "Binaries"
    ) else { return }

    let process = Process()
    process.executableURL = binaryURL
    // "a" = add to archive
    process.arguments = ["a", outputPath, inputPath]

    do {
        try process.run()
        process.waitUntilExit()
        print("Archive created at \(outputPath), exit: \(process.terminationStatus)")
    } catch {
        print("Failed to create archive: \(error)")
    }
}
```

### Listing Archive Contents

```swift
func listArchiveContents(archivePath: String) -> [String] {
    guard let binaryURL = Bundle.main.url(
        forResource: "7zz",
        withExtension: nil,
        subdirectory: "Binaries"
    ) else { return [] }

    let process = Process()
    process.executableURL = binaryURL
    process.arguments = ["l", archivePath]

    let pipe = Pipe()
    process.standardOutput = pipe

    do {
        try process.run()
        process.waitUntilExit()
        let data = pipe.fileHandleForReading.readDataToEndOfFile()
        let output = String(data: data, encoding: .utf8) ?? ""
        return output.components(separatedBy: "\n").filter { !$0.isEmpty }
    } catch {
        print("Failed to list archive: \(error)")
        return []
    }
}
```

---

## Common Makefile Targets

```bash
# Build the mainline 7-Zip library
make lib-mainline

# Build the Zstandard fork library
make lib-zs

# Clean build artifacts (if defined)
make clean
```

---

## Xcode Build Flags Reference

| Flag              | Description                          |
|-------------------|--------------------------------------|
| `-project`        | Path to `.xcodeproj`                 |
| `-scheme`         | `ShichiZip` or `ShichiZipZS`        |
| `-configuration`  | `Debug` or `Release`                 |
| `-arch`           | `arm64` (Apple Silicon) or `x86_64`  |
| `build`           | Build action                         |

---

## Troubleshooting

### `make lib-mainline` fails with missing dependencies
- Ensure Xcode command-line tools are installed: `xcode-select --install`
- Verify you have internet access for any fetched dependencies during `make`

### Xcode build fails: "No such scheme"
- Confirm you ran the correct `make` target before building
- Open `ShichiZip.xcodeproj` in Xcode and verify the scheme list matches

### Binary not found at runtime
- Ensure the 7-Zip binary is included in the app bundle's Copy Files build phase
- Check `Bundle.main.url(forResource:)` path matches the actual bundle structure

### Architecture mismatch
- If running on Intel Mac, replace `-arch arm64` with `-arch x86_64`
- For universal binary: `-arch arm64 -arch x86_64` (requires both library builds)

### Permission denied on extraction
- Use `NSOpenPanel` or `fileImporter` to obtain security-scoped bookmarks before accessing user files
- Call `url.startAccessingSecurityScopedResource()` / `url.stopAccessingSecurityScopedResource()`

```swift
if url.startAccessingSecurityScopedResource() {
    defer { url.stopAccessingSecurityScopedResource() }
    extract(archivePath: url.path, destinationPath: destinationURL.path)
}
```

---

## License

ShichiZip is licensed under **LGPL-2.1**, inherited from 7-Zip's licensing terms.
```
