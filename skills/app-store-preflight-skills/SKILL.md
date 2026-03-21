```markdown
---
name: app-store-preflight-skills
description: AI agent skill for scanning iOS/macOS projects to catch App Store rejection patterns before submission
triggers:
  - check my app for App Store rejection
  - run preflight checks before App Store submission
  - scan my iOS project for review guideline violations
  - preflight my Xcode project for App Store
  - catch App Store rejection issues
  - validate my app metadata for submission
  - check for Apple review guideline violations
  - pre-submission scan for iOS app
---

# App Store Preflight Skills

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

An AI agent skill that runs pre-submission checks on iOS/macOS projects to catch common App Store rejection patterns before you submit to Apple Review.

## What This Skill Does

App Store Preflight scans your:
- **Xcode project files** — entitlements, capabilities, configuration
- **Source code** — Sign in with Apple usage, privacy manifest, data collection
- **App metadata** — descriptions, keywords, screenshots, preview videos
- **Subscription/IAP setup** — ToS links, pricing display, EULA presence
- **Privacy configuration** — `PrivacyInfo.xcprivacy`, unnecessary data requests

It maps findings to specific Apple Review Guidelines, assigns severity (`REJECTION` or `WARNING`), and provides resolution steps.

## Install

```bash
npx skills add truongduy2611/app-store-preflight-skills
```

This installs the skill so AI coding agents (Claude Code, Cursor, Codex, etc.) can use it automatically when you ask about App Store submission readiness.

## Prerequisites

Install the `asc` CLI for metadata pulling:

```bash
brew install asc
```

Set up App Store Connect API credentials:

```bash
export ASC_KEY_ID="your_key_id"
export ASC_ISSUER_ID="your_issuer_id"
export ASC_PRIVATE_KEY_PATH="/path/to/AuthKey_XXXXXXXX.p8"
```

## Core Workflow

### 1. Identify Your App Type

Choose the matching checklist from `references/guidelines/by-app-type/`:

| App Type | Checklist File |
|----------|---------------|
| All apps (universal) | `all_apps.md` |
| Subscriptions / IAP | `subscription_iap.md` |
| Social / UGC | `social_ugc.md` |
| Kids Category | `kids.md` |
| Health & Fitness | `health_fitness.md` |
| Games | `games.md` |
| macOS / Mac App Store | `macos.md` |
| AI / Generative AI | `ai_apps.md` |
| Crypto / Finance | `crypto_finance.md` |
| VPN / Networking | `vpn.md` |

### 2. Pull App Metadata

```bash
# Pull metadata for a specific app and version
asc metadata pull --app "1234567890" --version "2.1.0" --dir ./metadata

# Or use the asc-metadata-sync skill
npx skills add rudrankriyam/app-store-connect-cli-skills
```

After pulling, your metadata directory structure will look like:

```
./metadata/
  en-US/
    description.txt
    keywords.txt
    release_notes.txt
    promotional_text.txt
  metadata.json
  screenshots/
  app_previews/
```

### 3. Run Preflight Scan

Ask your AI agent to run preflight checks:

```
"Check my iOS project at ./MyApp.xcodeproj for App Store rejection patterns"
"Run preflight scan on my subscription app before I submit version 2.1.0"
"Scan my metadata folder and Xcode project for App Store guideline violations"
```

## Rejection Rules Reference

All rules live in `references/rules/` organized by category.

### Metadata Rules (`references/rules/metadata/`)

**competitor_terms** — Guideline 2.3.1
```
REJECTS: Description or keywords containing "Android", "Google Play", 
         "Samsung", "Windows" or other competitor brand references
CHECK:   metadata/en-US/description.txt
         metadata/en-US/keywords.txt
         metadata/en-US/promotional_text.txt
```

**apple_trademark** — Guideline 5.2.5
```
REJECTS: App icon containing device frames (iPhone bezels, MacBook outline)
         Misuse of "Apple", "iPhone", "iPad" in app name or description
CHECK:   AppIcon.appiconset/
         metadata/en-US/description.txt
```

**china_storefront** — Guideline 5
```
REJECTS: References to OpenAI, ChatGPT, Gemini in China storefront metadata
CHECK:   metadata/zh-Hans/description.txt (China-specific locale)
```

**accurate_metadata** — Guideline 2.3.4
```
REJECTS: App preview videos containing device frames/bezels
         Screenshots that misrepresent actual app UI
CHECK:   metadata/app_previews/
```

**subscription_metadata** — Guideline 3.1.2
```
REJECTS: Subscription app missing ToS/EULA URL in metadata
         Missing Privacy Policy URL
CHECK:   metadata.json → "licenseAgreementUrl"
         metadata.json → "privacyPolicyUrl"
```

### Subscription Rules (`references/rules/subscription/`)

**missing_tos_pp** — Guideline 3.1.2

Check `metadata.json` for required URLs:
```json
{
  "privacyPolicyUrl": "https://yourapp.com/privacy",
  "licenseAgreementUrl": "https://yourapp.com/terms"
}
```

Also verify in-app presentation:
```swift
// ✅ Required: Show links before purchase
Button("Subscribe - $9.99/month") { }
Link("Terms of Service", destination: URL(string: "https://yourapp.com/terms")!)
Link("Privacy Policy", destination: URL(string: "https://yourapp.com/privacy")!)
```

**misleading_pricing** — Guideline 3.1.2
```swift
// ❌ REJECTION: Monthly price more prominent than actual billing
Text("Only $2.99/month")        // Large, prominent
Text("Billed $35.99 annually")  // Small, hidden

// ✅ CORRECT: Annual price equally or more prominent
Text("$35.99/year")
Text("($2.99/month)")
```

### Privacy Rules (`references/rules/privacy/`)

**privacy_manifest** — Guideline 5.1.1

Check for `PrivacyInfo.xcprivacy` in your project:
```bash
find . -name "PrivacyInfo.xcprivacy" -not -path "*/node_modules/*"
```

Required file structure:
```xml
<!-- MyApp/PrivacyInfo.xcprivacy -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyTracking</key>
  <false/>
  <key>NSPrivacyTrackingDomains</key>
  <array/>
  <key>NSPrivacyCollectedDataTypes</key>
  <array/>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <!-- Required if using UserDefaults -->
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>CA92.1</string>
      </array>
    </dict>
  </array>
</dict>
</plist>
```

**unnecessary_data** — Guideline 5.1.1
```swift
// ❌ REJECTION: Requiring irrelevant personal data at signup
struct SignUpView: View {
    @State var birthdate: Date      // Not needed for your app type
    @State var phoneNumber: String  // Not needed for your app type
    @State var address: String      // Not needed for your app type
}

// ✅ CORRECT: Only collect what your app genuinely needs
struct SignUpView: View {
    @State var email: String
    @State var password: String
    // Request additional data only when feature requires it
}
```

### Design Rules (`references/rules/design/`)

**sign_in_with_apple** — Guideline 4.0
```swift
// ❌ REJECTION: Asking for name/email AFTER Sign in with Apple completes
func authorizationController(
    controller: ASAuthorizationController,
    didCompleteWithAuthorization authorization: ASAuthorization
) {
    // DON'T show a form asking for name or email here
    // Apple already provided it — use credential.fullName and credential.email
    if let credential = authorization.credential as? ASAuthorizationAppleIDCredential {
        let name = credential.fullName      // ✅ Use this
        let email = credential.email        // ✅ Use this
        // ❌ Don't: showAdditionalInfoForm()
    }
}
```

**minimum_functionality** — Guideline 4.2
```
REJECTS: App is a pure WebView wrapper with no native functionality
         App has fewer than 3 distinct screens/features
         App provides no unique value over mobile browser
         App is a template/demo with placeholder content

SIGNALS TO CHECK:
- WKWebView loading a single URL as the entire app
- Only 1-2 view controllers in the entire project
- No use of native iOS APIs beyond basic UI
```

Check for WebView-only pattern:
```swift
// ❌ REJECTION: Entire app is one WebView
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            WebView(url: URL(string: "https://mywebsite.com")!)
            // Nothing else in the app
        }
    }
}
```

### Entitlements Rules (`references/rules/entitlements/`)

**unused_entitlements** — Guideline 2.4.5(i)

Scan your `.entitlements` file against actual code usage:
```bash
# Find your entitlements file
find . -name "*.entitlements" -not -path "*/Build/*"

# Check for commonly flagged unused entitlements
# These must be USED in code, not just declared:
# - com.apple.security.personal-information.contacts
# - com.apple.security.personal-information.calendars  
# - com.apple.security.personal-information.location
```

```xml
<!-- MyApp.entitlements — only include what your app actively uses -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- ✅ Only if your app actually reads contacts -->
  <key>com.apple.security.personal-information.contacts</key>
  <true/>
</dict>
</plist>
```

## Running a Full Preflight Check

When asked to run a preflight scan, the AI agent follows this sequence:

```
1. READ   references/guidelines/by-app-type/<matching_type>.md
2. SCAN   Xcode project → entitlements, Info.plist, capabilities
3. SCAN   Source code → SIWA usage, WebView usage, data collection
4. SCAN   PrivacyInfo.xcprivacy → existence and completeness  
5. SCAN   metadata/ → all locale folders, metadata.json
6. MAP    Each finding → rule file in references/rules/<category>/
7. REPORT Severity, affected file, guideline number, resolution
8. FIX    Apply fixes for REJECTION-severity issues
9. VERIFY Re-run affected checks after fixes
```

## Sample Preflight Report Output

```
🛫 App Store Preflight Report — MyApp v2.1.0
============================================

❌ REJECTION (3 issues)
-----------------------
[1] Missing PrivacyInfo.xcprivacy
    Rule: references/rules/privacy/privacy_manifest.md
    Guideline: 5.1.1
    File: MyApp/ (file not found)
    Fix: Create MyApp/PrivacyInfo.xcprivacy and add to Xcode target

[2] Competitor term in description
    Rule: references/rules/metadata/competitor_terms.md  
    Guideline: 2.3.1
    File: metadata/en-US/description.txt:14
    Found: "better than Google's apps"
    Fix: Remove all competitor brand references

[3] Missing Privacy Policy URL in metadata
    Rule: references/rules/subscription/missing_tos_pp.md
    Guideline: 3.1.2
    File: metadata.json → privacyPolicyUrl
    Fix: Add "privacyPolicyUrl": "https://yourapp.com/privacy"

⚠️  WARNING (1 issue)
----------------------
[4] Annual subscription price less prominent than monthly equivalent
    Rule: references/rules/subscription/misleading_pricing.md
    Guideline: 3.1.2
    File: SubscriptionView.swift:42
    Fix: Display annual price at equal or larger font size

✅ PASSED (11 checks)
```

## Adding Custom Rules

Create a `.md` file in the appropriate `references/rules/` subdirectory:

```markdown
# Rule: custom-rule-name
- **Guideline**: 2.3.1
- **Severity**: REJECTION
- **Category**: metadata

## What to Check
Describe what pattern triggers this rule.

## How to Detect
```bash
# Command or code pattern to detect the issue
grep -r "problematic_pattern" ./metadata/
```

## Resolution
Step-by-step fix instructions.

## Example Rejection
Paste or describe a real rejection message from Apple.
```

## Integration with ASC CLI Skills

Use alongside [`rudrankriyam/app-store-connect-cli-skills`](https://github.com/rudrankriyam/app-store-connect-cli-skills) for a full submission workflow:

```bash
# Install both skills
npx skills add truongduy2611/app-store-preflight-skills
npx skills add rudrankriyam/app-store-connect-cli-skills

# Typical pre-submission workflow:
# 1. Pull current metadata
asc metadata pull --app "$APP_ID" --version "$VERSION" --dir ./metadata

# 2. Ask agent to run preflight
# "Run full preflight scan on my app before I submit"

# 3. Fix all REJECTION issues

# 4. Push corrected metadata
asc metadata push --app "$APP_ID" --version "$VERSION" --dir ./metadata

# 5. Submit for review
asc submit --app "$APP_ID" --version "$VERSION"
```

## Troubleshooting

**`asc` command not found**
```bash
brew install asc
# Verify:
asc --version
```

**Metadata pull fails with auth error**
```bash
# Verify your environment variables are set:
echo $ASC_KEY_ID        # Should print your key ID
echo $ASC_ISSUER_ID     # Should print your issuer ID
ls $ASC_PRIVATE_KEY_PATH # Should show the .p8 file exists
```

**Preflight not finding my Xcode project**
```bash
# Run from your project root, or specify the path explicitly:
# "Run preflight on the Xcode project at ./ios/MyApp.xcodeproj"
find . -name "*.xcodeproj" -maxdepth 3
```

**Using fastlane metadata instead of `asc` layout**
```
The rules reference canonical asc metadata layout (metadata/en-US/description.txt).
Fastlane uses a similar layout but may differ in folder structure.
Either: run `asc metadata pull` to get canonical layout,
or: tell the agent your metadata is in fastlane format so it adjusts paths.
```

**Rule not triggering on a known issue**
```
Check references/rules/<category>/ — if no rule exists for your issue, 
add a custom rule file following the template in "Adding Custom Rules" above.
```

## Key Files Reference

```
references/
  guidelines/
    README.md                          # Full guideline index
    by-app-type/
      all_apps.md                      # Universal checklist
      subscription_iap.md
      social_ugc.md
      kids.md
      health_fitness.md
      games.md
      macos.md
      ai_apps.md
      crypto_finance.md
      vpn.md
  rules/
    metadata/
      competitor_terms.md
      apple_trademark.md
      china_storefront.md
      accurate_metadata.md
      subscription_metadata.md
    subscription/
      missing_tos_pp.md
      misleading_pricing.md
    privacy/
      unnecessary_data.md
      privacy_manifest.md
    design/
      sign_in_with_apple.md
      minimum_functionality.md
    entitlements/
      unused_entitlements.md
SKILL.md                               # Full AI agent instructions
```
```
