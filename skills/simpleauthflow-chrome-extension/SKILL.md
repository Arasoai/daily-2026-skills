```markdown
---
name: simpleauthflow-chrome-extension
description: Chrome extension that automates the ChatGPT OAuth registration and authorization flow with zero configuration required
triggers:
  - automate chatgpt oauth flow
  - chrome extension for chatgpt registration
  - simpleauthflow setup
  - automate openai account creation
  - chatgpt oauth chrome extension
  - simplify chatgpt authorization
  - automated chatgpt signup extension
  - chatgpt burner mailbox automation
---

# SimpleAuthFlow Chrome Extension

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

SimpleAuthFlow is a Chrome extension that automates the ChatGPT OAuth registration and authorization flow. It is designed for zero-configuration personal use — install it, point it at a local CPA instance, and it handles the entire signup/auth cycle automatically including email verification, CAPTCHA prompts, and protocol confirmation pages.

---

## What It Does

- Automates the full ChatGPT OAuth flow (registration + authorization)
- Uses Burner Mailbox for disposable email addresses — no personal email needed
- Handles email verification codes automatically
- Detects and auto-confirms OpenAI agreement/protocol pages
- Supports semi-automatic mode: manually handle one step, auto-continue the rest
- Provides a **Workflow** panel with step-by-step controls and a **Continue** button for resuming interrupted flows
- Works with a local [CPA](https://github.com/) instance at `http://127.0.0.1:5173/#/oauth` by default

---

## Prerequisites

1. **CPA** — the OAuth callback/proxy service, running locally (default: `http://127.0.0.1:5173/#/oauth`)
2. **This extension** — loaded as an unpacked Chrome extension

No personal email address or phone number is required.

---

## Installation

### Load the Extension (Developer Mode)

```bash
# 1. Clone the repository
git clone https://github.com/NyxTides/SimpleAuthFlow.git
cd SimpleAuthFlow

# 2. Open Chrome and navigate to:
#    chrome://extensions/

# 3. Enable "Developer mode" (top-right toggle)

# 4. Click "Load unpacked"

# 5. Select the SimpleAuthFlow project folder
```

### Start CPA Locally

```bash
# Example: start CPA on default port 5173
cd /path/to/cpa
npm install
npm run dev
# CPA now listens at http://127.0.0.1:5173/#/oauth
```

---

## Using the Extension

### Basic Flow

1. Click the extension icon in the Chrome toolbar to open the side panel.
2. The VPS/CPA URL field is pre-filled with `http://127.0.0.1:5173/#/oauth`. Leave it as-is for local use.
3. Click **Auto** to start the automated flow.
4. The extension will:
   - Obtain a disposable email from Burner Mailbox
   - Register a new ChatGPT account
   - Fetch verification codes from the inbox
   - Confirm any OpenAI protocol/agreement pages
   - Complete the OAuth callback to CPA

### Workflow Panel Controls

| Button | Behavior |
|---|---|
| **Auto** | Start the full automated flow from the beginning |
| **Continue** | Resume from the last successful step (skips already-completed steps) |
| **Stop** | Immediately halt the current flow at any point |
| Individual step buttons | Click any step directly to run it in isolation (supports manual handoff) |

### Semi-Automatic Mode

If the automated email chain fails (e.g., Burner Mailbox triggers a CAPTCHA):

1. Click **Stop** to pause.
2. Manually complete the problematic step (e.g., solve CAPTCHA in the email tab, retrieve the code yourself).
3. Click the **next step button** in the Workflow panel to continue automation from that point.
4. Or click **Continue** — it resumes from the last confirmed successful step.

---

## Configuration

### VPS/CPA URL

The input field in the side panel accepts the CPA OAuth endpoint. Default value:

```
http://127.0.0.1:5173/#/oauth
```

Toggle visibility of the URL with the eye icon (👁) next to the input field.

### Rate Limits (Default Behavior)

- ~1 minute per complete flow
- ~5 flows per day on a single IP/request environment (same node = same fingerprint)
- To exceed 5/day: switch your proxy/VPN node and continue

No config file changes are needed to adjust this — it is an upstream site limitation based on IP + request headers.

---

## Key Internal Behaviors

### Email Retry Logic

```
- Wait 4 seconds for new email to arrive
- If no email detected → auto-trigger resend
- Retry up to 3 rounds before marking as failed
- Most flows succeed on the first attempt
```

### Security Verification (CAPTCHA) Handling

When Burner Mailbox shows a human verification page:

1. Extension detects the security check page.
2. Side panel displays a prompt: **"Please complete the verification in the email tab."**
3. User solves the CAPTCHA manually.
4. User clicks **Continue** in the side panel.
5. Extension waits for the page to return to normal, then resumes.

### OpenAI Agreement Page

If OpenAI injects a "please agree to terms" or "confirm to continue" interstitial:

- Extension automatically detects it.
- Clicks through without interrupting the flow.
- No user action required.

---

## Code Examples

### Injecting a Step Handler (content script pattern)

```javascript
// content/steps/confirmAgreement.js

/**
 * Detects and auto-confirms OpenAI protocol/agreement pages.
 * Returns true if handled, false if page not detected.
 */
async function confirmAgreementIfPresent() {
  const continueBtn = document.querySelector(
    'button[data-testid="accept-terms-button"], button.continue-btn'
  );
  if (!continueBtn) return false;

  console.log('[SimpleAuthFlow] Agreement page detected, confirming...');
  continueBtn.click();

  // Wait for navigation away from the agreement page
  await waitForNavigation(3000);
  return true;
}

function waitForNavigation(timeout = 3000) {
  return new Promise((resolve) => {
    const start = location.href;
    const interval = setInterval(() => {
      if (location.href !== start) {
        clearInterval(interval);
        resolve();
      }
    }, 200);
    setTimeout(() => {
      clearInterval(interval);
      resolve();
    }, timeout);
  });
}
```

### Sending a Stop Signal from the Side Panel

```javascript
// sidepanel/index.js

document.getElementById('stop-btn').addEventListener('click', () => {
  chrome.runtime.sendMessage({ type: 'STOP_FLOW' }, (response) => {
    console.log('[SimpleAuthFlow] Stop acknowledged:', response);
  });
});
```

### Handling Stop in the Background Service Worker

```javascript
// background/worker.js

let flowActive = false;

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'STOP_FLOW') {
    flowActive = false;
    sendResponse({ stopped: true });
  }

  if (message.type === 'START_FLOW') {
    flowActive = true;
    runFlow().catch(console.error);
    sendResponse({ started: true });
  }
});

async function runFlow() {
  const steps = getSteps(); // returns ordered step functions
  for (const step of steps) {
    if (!flowActive) {
      console.log('[SimpleAuthFlow] Flow stopped by user.');
      break;
    }
    await step();
  }
}
```

### Continue from Last Successful Step

```javascript
// sidepanel/workflow.js

/**
 * Finds the index of the last successful step and resumes from the next one.
 */
function continueFromLastSuccess(steps, stepStatuses) {
  let lastSuccess = -1;

  for (let i = 0; i < steps.length; i++) {
    if (stepStatuses[i] === 'success') {
      lastSuccess = i;
    }
  }

  const resumeFrom = lastSuccess + 1;

  if (resumeFrom >= steps.length) {
    console.log('[SimpleAuthFlow] All steps already completed.');
    return;
  }

  console.log(`[SimpleAuthFlow] Resuming from step ${resumeFrom}: ${steps[resumeFrom].name}`);
  runStepsFrom(steps, resumeFrom);
}
```

### Email Retry with Backoff

```javascript
// content/steps/fetchVerificationCode.js

const MAX_RETRIES = 3;
const WAIT_MS = 4000;

async function fetchVerificationCode(triggerResendFn) {
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    console.log(`[SimpleAuthFlow] Waiting for verification email (attempt ${attempt})...`);
    await sleep(WAIT_MS);

    const code = extractCodeFromInbox();
    if (code) {
      console.log(`[SimpleAuthFlow] Code received: ${code}`);
      return code;
    }

    console.warn(`[SimpleAuthFlow] No email detected, triggering resend (attempt ${attempt})...`);
    await triggerResendFn();
  }

  throw new Error('[SimpleAuthFlow] Failed to receive verification code after max retries.');
}

function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function extractCodeFromInbox() {
  // DOM scraping logic specific to Burner Mailbox inbox
  const codeEl = document.querySelector('.email-body .verification-code');
  return codeEl ? codeEl.textContent.trim() : null;
}
```

---

## Troubleshooting

### Flow Stops Mid-Way

- Click **Continue** — it resumes from the last successful step, not from the beginning.
- If a specific step is stuck, click that step's button in the Workflow panel to retry it individually.

### Verification Code Never Arrives

- The extension auto-retries up to 3 times with resend triggers.
- If all retries fail, Burner Mailbox may be flagging your IP. Switch your VPN/proxy node and click Continue.

### Burner Mailbox Shows CAPTCHA

- Extension will display a prompt in the side panel.
- Solve the CAPTCHA manually in the email tab.
- Return to the side panel and click **Continue**.

### Birthday Field Cannot Be Filled

- The current version uses a stable plain-input fill strategy (dropdown compatibility was reverted for stability).
- If the birthday step fails, click the birthday step button individually to retry, or fill it manually and click Continue.

### Rate Limit Reached (>5/day)

- Switch your VPN/proxy to a different node (different IP).
- The site fingerprints by IP + request headers; a new node resets the counter.

### Stop Button Unresponsive

- The Stop button is always clickable regardless of flow state — it sends an immediate termination message to the background worker.
- If the tab is frozen, close and reopen the side panel.

---

## Project Structure

```
SimpleAuthFlow/
├── manifest.json          # Chrome extension manifest (MV3)
├── background/
│   └── worker.js          # Service worker: flow orchestration, message bus
├── content/
│   ├── main.js            # Content script injected into ChatGPT/OpenAI pages
│   └── steps/             # Individual automation step modules
│       ├── register.js
│       ├── fetchEmail.js
│       ├── fetchVerificationCode.js
│       ├── confirmAgreement.js
│       └── oauthCallback.js
├── sidepanel/
│   ├── index.html         # Side panel UI
│   ├── index.js           # Panel logic: Auto/Stop/Continue buttons, step display
│   └── workflow.js        # Step state tracking and resume logic
└── icons/
    └── icon.png
```

---

## Notes

- Built on top of [StepFlow-Duck v1.0](https://github.com/whwh1233/StepFlow-Duck) with two rounds of community modifications.
- Intended for **personal use only** — not designed for bulk/large-scale operation.
- Default throughput: ~1 flow/minute, ~5 flows/day per IP.
- License: MIT
```
