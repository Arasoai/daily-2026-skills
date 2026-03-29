```markdown
---
name: qclaw-openclaw-desktop
description: GUI desktop client for OpenClaw — lets non-technical users run and configure OpenClaw AI gateway without the command line
triggers:
  - set up Qclaw desktop app
  - configure OpenClaw with GUI
  - add IM channel to Qclaw
  - develop Qclaw electron app
  - build Qclaw from source
  - connect feishu dingtalk to OpenClaw
  - manage OpenClaw models in Qclaw
  - Qclaw skill plugin management
---

# Qclaw OpenClaw Desktop

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Qclaw is an Electron-based desktop GUI for [OpenClaw](https://github.com/openclaw/openclaw), a self-hosted AI gateway. It removes the need for CLI configuration by providing visual wizards for model setup, IM channel integration (Feishu, DingTalk, QQ, WeCom), skill management, and gateway monitoring.

## What Qclaw Does

- **Environment auto-check** — detects and installs Node.js and OpenClaw CLI automatically
- **Model management** — configure any OpenClaw-supported model with OAuth or API key
- **IM integration** — scan QR codes to connect Feishu, DingTalk, QQ, WeCom, WeChat
- **Gateway dashboard** — real-time status, one-click restart/repair
- **Skills management** — install and manage OpenClaw skill plugins
- **Backup/restore** — auto and manual config backups
- **Auto-update** — keeps OpenClaw CLI up to date

## Installation (End Users)

Download from:
- **Official site**: https://qclawai.com/
- **GitHub Releases**: https://github.com/qiuzhi2046/Qclaw/releases

Supported platforms:
- macOS 11 (Big Sur)+
- Windows 10+ x64 (in development)
- Linux (planned)

## Development Setup

### Prerequisites

- Node.js 22+ (Node.js 24 recommended)
- macOS (primary dev platform)
- Git

### Clone and Run

```bash
git clone https://github.com/qiuzhi2046/Qclaw.git
cd Qclaw

npm install

# Start dev server (Electron + Vite hot reload)
npm run dev

# Type check
npm run typecheck

# Run tests
npm test

# Build production app
npm run build
```

### Key npm Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start Electron dev server with HMR |
| `npm run build` | Build and package app with electron-builder |
| `npm test` | Run test suite |
| `npm run typecheck` | TypeScript type checking |

## Project Structure

```
electron/
  main/         # Main process: window, CLI calls, IPC handlers
  preload/      # Preload scripts (secure IPC bridge)
src/
  pages/        # React pages: wizard steps, Dashboard, Chat
  components/   # Shared UI components
  lib/          # Business logic: channel/provider registration
  shared/       # Config workflows, gateway diagnostics
  assets/       # Icons and static assets
docs/           # Architecture docs, changelogs
scripts/        # Build, signing, notarization, versioning, COS release
build/          # App icons and packaging resources
```

## Tech Stack

| Layer | Technology |
|---|---|
| Desktop | Electron |
| Frontend | React + TypeScript |
| Build | Vite + vite-plugin-electron |
| UI | Mantine + Tailwind CSS |
| Packaging | electron-builder |

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│                    Qclaw                    │
│                                             │
│  Main Process (Node.js)  ◄─IPC─►  Renderer │
│  ┌──────────────────┐            (React)    │
│  │ cli.ts           │                       │
│  │ OpenClaw CLI     │    Wizard pages        │
│  │ invocation       │    Dashboard           │
│  ├──────────────────┤    Chat UI             │
│  │ File I/O         │                       │
│  │ Process mgmt     │                       │
│  │ System integration│                      │
│  └────────┬─────────┘                       │
│           │                                 │
│           ▼                                 │
│  OpenClaw CLI (~/.openclaw/)                │
└─────────────────────────────────────────────┘
```

## IPC Communication Pattern

Qclaw uses Electron's contextBridge for secure main ↔ renderer communication.

### Preload Bridge (electron/preload/index.ts)

```typescript
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  // Invoke main process handlers
  invoke: (channel: string, ...args: unknown[]) =>
    ipcRenderer.invoke(channel, ...args),
  // Subscribe to events from main
  on: (channel: string, callback: (...args: unknown[]) => void) => {
    ipcRenderer.on(channel, (_event, ...args) => callback(...args))
  },
  // Remove listeners
  removeAllListeners: (channel: string) => {
    ipcRenderer.removeAllListeners(channel)
  },
})
```

### Main Process IPC Handler (electron/main/index.ts)

```typescript
import { ipcMain } from 'electron'
import { runOpenClawCli } from './cli'

// Handle CLI invocation from renderer
ipcMain.handle('openclaw:start', async (_event, config) => {
  try {
    const result = await runOpenClawCli(['start', '--config', config])
    return { success: true, output: result }
  } catch (error) {
    return { success: false, error: (error as Error).message }
  }
})

ipcMain.handle('openclaw:status', async () => {
  return await runOpenClawCli(['status'])
})

ipcMain.handle('openclaw:restart', async () => {
  return await runOpenClawCli(['restart'])
})
```

### Renderer Usage

```typescript
// src/pages/Dashboard.tsx
const handleRestart = async () => {
  const result = await window.electronAPI.invoke('openclaw:restart')
  if (result.success) {
    notifications.show({ message: 'Gateway restarted', color: 'green' })
  }
}
```

## Adding a New IM Channel

Channels are registered in `src/lib/` following a provider pattern.

```typescript
// src/lib/channels/myChannel.ts
import type { IMChannel } from '../types'

export const myChannel: IMChannel = {
  id: 'my-channel',
  name: 'My Platform',
  icon: '/assets/my-channel-icon.png',
  pluginPackage: '@openclaw/plugin-my-channel',

  // Fields shown in the setup wizard
  configFields: [
    {
      key: 'appId',
      label: 'App ID',
      type: 'text',
      required: true,
      placeholder: 'Enter your App ID',
    },
    {
      key: 'appSecret',
      label: 'App Secret',
      type: 'password',
      required: true,
      placeholder: 'Enter your App Secret',
    },
  ],

  // Called when user saves the channel config
  async install(config: Record<string, string>) {
    // Install plugin via CLI
    await window.electronAPI.invoke('openclaw:install-plugin', this.pluginPackage)
    // Write config
    await window.electronAPI.invoke('openclaw:write-channel-config', {
      channel: this.id,
      config,
    })
  },
}
```

Register it in `src/lib/channels/index.ts`:

```typescript
import { myChannel } from './myChannel'

export const channels = [
  feishuChannel,
  dingtalkChannel,
  qqChannel,
  wecomChannel,
  myChannel, // add here
]
```

## Adding a New AI Provider/Model

```typescript
// src/lib/providers/myProvider.ts
import type { AIProvider } from '../types'

export const myProvider: AIProvider = {
  id: 'my-provider',
  name: 'My AI Provider',
  authType: 'api-key', // or 'oauth'

  models: [
    { id: 'my-model-v1', name: 'My Model V1', contextLength: 128000 },
    { id: 'my-model-v2', name: 'My Model V2', contextLength: 200000 },
  ],

  configFields: [
    {
      key: 'apiKey',
      label: 'API Key',
      type: 'password',
      required: true,
      envVar: 'MY_PROVIDER_API_KEY', // shown as hint to user
    },
    {
      key: 'baseUrl',
      label: 'Base URL',
      type: 'text',
      required: false,
      defaultValue: 'https://api.myprovider.com',
    },
  ],
}
```

## Gateway Diagnostics (src/shared/)

```typescript
// src/shared/gatewayDiagnostics.ts
import { ipcRenderer } from 'electron'

export async function diagnoseGateway(): Promise<DiagnosticResult> {
  const checks = await Promise.allSettled([
    checkNodeVersion(),
    checkOpenClawInstalled(),
    checkConfigFile(),
    checkGatewayProcess(),
  ])

  return {
    nodeOk: checks[0].status === 'fulfilled',
    cliOk: checks[1].status === 'fulfilled',
    configOk: checks[2].status === 'fulfilled',
    processOk: checks[3].status === 'fulfilled',
    errors: checks
      .filter(c => c.status === 'rejected')
      .map(c => (c as PromiseRejectedResult).reason),
  }
}

async function checkNodeVersion(): Promise<void> {
  const version = await window.electronAPI.invoke('system:node-version')
  const major = parseInt(version.replace('v', '').split('.')[0])
  if (major < 22) throw new Error(`Node.js ${version} is below minimum v22`)
}
```

## Config File Location

OpenClaw stores its config at `~/.openclaw/`. Qclaw reads/writes this directory via the main process.

```typescript
// electron/main/configManager.ts
import path from 'path'
import os from 'os'
import fs from 'fs/promises'

const OPENCLAW_DIR = path.join(os.homedir(), '.openclaw')
const CONFIG_FILE = path.join(OPENCLAW_DIR, 'config.yaml')

export async function readConfig(): Promise<string> {
  return fs.readFile(CONFIG_FILE, 'utf-8')
}

export async function writeConfig(content: string): Promise<void> {
  await fs.mkdir(OPENCLAW_DIR, { recursive: true })
  await fs.writeFile(CONFIG_FILE, content, 'utf-8')
}

export async function backupConfig(): Promise<string> {
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-')
  const backupPath = path.join(OPENCLAW_DIR, `backups/config-${timestamp}.yaml`)
  await fs.mkdir(path.dirname(backupPath), { recursive: true })
  await fs.copyFile(CONFIG_FILE, backupPath)
  return backupPath
}
```

## Wizard Step Component Pattern

```typescript
// src/pages/wizard/StepModelConfig.tsx
import { useState } from 'react'
import { Button, Select, PasswordInput, Stack, Text } from '@mantine/core'
import { notifications } from '@mantine/notifications'

interface Props {
  onComplete: () => void
  onBack: () => void
}

export function StepModelConfig({ onComplete, onBack }: Props) {
  const [provider, setProvider] = useState<string | null>(null)
  const [apiKey, setApiKey] = useState('')
  const [loading, setLoading] = useState(false)

  const handleSave = async () => {
    if (!provider || !apiKey) return
    setLoading(true)
    try {
      await window.electronAPI.invoke('openclaw:configure-provider', {
        provider,
        apiKey,
      })
      notifications.show({
        title: 'Success',
        message: 'Model configured successfully',
        color: 'green',
      })
      onComplete()
    } catch (err) {
      notifications.show({
        title: 'Error',
        message: (err as Error).message,
        color: 'red',
      })
    } finally {
      setLoading(false)
    }
  }

  return (
    <Stack gap="md">
      <Text size="lg" fw={600}>Configure AI Model</Text>
      <Select
        label="Provider"
        placeholder="Select a provider"
        data={['openai', 'anthropic', 'gemini', 'custom']}
        value={provider}
        onChange={setProvider}
      />
      <PasswordInput
        label="API Key"
        placeholder="sk-..."
        value={apiKey}
        onChange={e => setApiKey(e.target.value)}
      />
      <Button onClick={handleSave} loading={loading} disabled={!provider || !apiKey}>
        Save & Continue
      </Button>
      <Button variant="subtle" onClick={onBack}>Back</Button>
    </Stack>
  )
}
```

## TypeScript Type Declarations for electronAPI

Add to `src/types/global.d.ts`:

```typescript
export {}

declare global {
  interface Window {
    electronAPI: {
      invoke: (channel: string, ...args: unknown[]) => Promise<unknown>
      on: (channel: string, callback: (...args: unknown[]) => void) => void
      removeAllListeners: (channel: string) => void
    }
  }
}
```

## Packaging and Release

```bash
# Build for current platform
npm run build

# The scripts/ directory contains:
# - sign-notarize.sh    macOS code signing + notarization
# - version.ts          version bump utility
# - release-cos.ts      publish to Tencent COS
```

electron-builder config is typically in `package.json` or `electron-builder.yml`:

```yaml
# electron-builder.yml (typical config)
appId: com.qiuzhi2046.qclaw
productName: Qclaw
directories:
  output: dist-electron
files:
  - dist
  - electron/main
  - electron/preload
mac:
  target: dmg
  icon: build/icon.icns
win:
  target: nsis
  icon: build/icon.ico
```

## Common Patterns

### Listen for Gateway Status Updates

```typescript
// In a React component
useEffect(() => {
  window.electronAPI.on('gateway:status-change', (status: unknown) => {
    setGatewayStatus(status as GatewayStatus)
  })
  return () => {
    window.electronAPI.removeAllListeners('gateway:status-change')
  }
}, [])
```

### Check if OpenClaw CLI Is Installed

```typescript
const isInstalled = await window.electronAPI.invoke('system:check-openclaw')
if (!isInstalled) {
  // Show install wizard step
  setCurrentStep('install-cli')
}
```

### Import Existing OpenClaw Config

```typescript
const existingConfig = await window.electronAPI.invoke('config:detect-existing')
if (existingConfig) {
  const confirmed = await modals.openConfirmModal({
    title: 'Existing configuration found',
    children: 'Import your existing OpenClaw configuration?',
  })
  if (confirmed) {
    await window.electronAPI.invoke('config:import', existingConfig)
  }
}
```

## Troubleshooting

| Problem | Solution |
|---|---|
| `npm run dev` fails | Ensure Node.js ≥ 22; run `npm install` again |
| Electron window blank | Check `vite.config.ts` for correct `base` path |
| IPC handler not found | Verify handler is registered before `app.whenReady()` |
| OpenClaw CLI not detected | Check `~/.openclaw/` exists; run env check in app |
| Build fails on macOS | Check signing identity with `security find-identity -v -p codesigning` |
| IM channel QR not showing | Ensure plugin installed: `~/.openclaw/plugins/` directory |

## Community and Support

- **GitHub Issues**: https://github.com/qiuzhi2046/Qclaw/issues
- **Official site**: https://qclawai.com/
- **IM setup guides**:
  - Feishu: https://my.feishu.cn/wiki/WAfWw1bqriZP02kqdNycHlvnnHb
  - DingTalk: https://my.feishu.cn/wiki/NUJew2DzaipVsukUvPmcZ2yvnYb
  - QQ: https://my.feishu.cn/wiki/AvuSwchqviAO6dkwiZycmZeInPf
  - WeCom: https://my.feishu.cn/wiki/TsLTwplveiqbW8kH5XOclgvYn1d
```
