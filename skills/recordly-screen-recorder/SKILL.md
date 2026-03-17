```markdown
---
name: recordly-screen-recorder
description: Expertise in using and extending Recordly, an open-source Electron screen recorder and editor with auto-zoom, cursor animations, and cross-platform capture.
triggers:
  - add zoom animation to screen recording
  - recordly cursor animation setup
  - screen studio alternative open source
  - recordly electron recording project
  - auto zoom screen recorder typescript
  - recordly export mp4 gif
  - recordly plugin contribution
  - build recordly from source
---

# Recordly Screen Recorder Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Recordly is a free, open-source, cross-platform screen recorder and editor built on Electron and TypeScript. It mimics Screen Studio's auto-zoom and cursor animation features, uses PixiJS for scene rendering, and provides platform-native capture (ScreenCaptureKit on macOS, WGC on Windows, Electron capture on Linux).

---

## Installation & Setup

### Prerequisites
- Node.js 18+
- npm 9+
- Git

### Clone and Run

```bash
git clone https://github.com/webadderall/Recordly.git recordly
cd recordly
npm install
npm run dev
```

### Build for Distribution

```bash
npm run build          # Compile TypeScript
npm run dist           # Package with electron-builder
```

### macOS Quarantine Fix (unsigned local builds)

```bash
xattr -rd com.apple.quarantine /Applications/Recordly.app
```

### Prebuilt Releases

Download from: `https://github.com/webadderall/Recordly/releases`

---

## Project Architecture

```
recordly/
├── src/
│   ├── main/           # Electron main process (capture, IPC, native helpers)
│   ├── renderer/       # Electron renderer process (editor UI, PixiJS pipeline)
│   ├── shared/         # Types, constants shared between main and renderer
│   └── native/         # Platform-specific native helpers (macOS SCK, Windows WGC)
├── public/             # Static assets
├── CONTRIBUTING.md
└── package.json
```

- **Main process**: Orchestrates recording start/stop, invokes native capture helpers, manages `.recordly` project files.
- **Renderer process**: Hosts the timeline editor, PixiJS scene compositor, cursor overlay pipeline.
- **PixiJS pipeline**: All zoom, cursor, pan, and speed-change effects are composed and rendered here — both in the live preview and during export.

---

## Key Concepts

### `.recordly` Project File

A `.recordly` file is a JSON bundle containing:

```typescript
interface RecordlyProject {
  version: string;
  sourceVideoPath: string;      // Absolute path to raw captured video
  cursorData: CursorEvent[];    // Array of {x, y, timestamp, type} events
  zoomRegions: ZoomRegion[];
  speedRegions: SpeedRegion[];
  trimIn: number;               // Seconds
  trimOut: number;              // Seconds
  frameStyle: FrameStyle;
  audioTrack?: string;          // Path to audio file
  exportSettings: ExportSettings;
}
```

### Cursor Events

```typescript
interface CursorEvent {
  x: number;          // Normalised 0–1 across display width
  y: number;          // Normalised 0–1 across display height
  timestamp: number;  // Milliseconds from recording start
  type: 'move' | 'click' | 'scroll';
  button?: 'left' | 'right' | 'middle';
}
```

### Zoom Regions

```typescript
interface ZoomRegion {
  id: string;
  startTime: number;    // Seconds in video timeline
  endTime: number;
  targetX: number;      // Normalised 0–1 centre of zoom target
  targetY: number;
  scale: number;        // e.g. 2.0 = 2× zoom
  easing: 'ease-in-out' | 'spring' | 'linear';
}
```

### Speed Regions

```typescript
interface SpeedRegion {
  id: string;
  startTime: number;
  endTime: number;
  speed: number;   // 0.25 = slow-mo, 2.0 = double speed
}
```

---

## IPC API (Main ↔ Renderer)

Recordly uses Electron's `ipcMain` / `ipcRenderer` for all cross-process communication.

### Start Recording

```typescript
// renderer → main
ipcRenderer.invoke('recording:start', {
  sourceId: 'screen:0',          // Electron desktopCapturer source ID
  audioMode: 'microphone',       // 'microphone' | 'system' | 'both' | 'none'
  hideSystemCursor: true,
});
```

### Stop Recording

```typescript
// renderer → main
const result = await ipcRenderer.invoke('recording:stop');
// result: { videoPath: string, cursorDataPath: string }
```

### Open Project

```typescript
// renderer → main
const project: RecordlyProject = await ipcRenderer.invoke('project:open', filePath);
```

### Save Project

```typescript
// renderer → main
await ipcRenderer.invoke('project:save', {
  filePath: '/path/to/output.recordly',
  project: projectState,
});
```

### Export

```typescript
// renderer → main
await ipcRenderer.invoke('export:start', {
  project: projectState,
  outputPath: '/path/to/output.mp4',
  format: 'mp4',           // 'mp4' | 'gif'
  quality: 'high',         // 'low' | 'medium' | 'high'
  aspectRatio: '16:9',
});
```

---

## PixiJS Rendering Pipeline

The editor and exporter share the same PixiJS scene graph. To add a new visual effect:

```typescript
import * as PIXI from 'pixi.js';

// Access the shared app instance (renderer process)
import { getPixiApp } from '@/renderer/pixi/appInstance';

const app = getPixiApp();

// Add a custom overlay layer
const overlayContainer = new PIXI.Container();
overlayContainer.name = 'myCustomEffect';
app.stage.addChild(overlayContainer);

// Hook into the render tick
app.ticker.add((delta) => {
  const currentTime = getPlayheadTime(); // from timeline store
  // Update your effect based on currentTime
  overlayContainer.alpha = computeEffectAlpha(currentTime);
});
```

### Cursor Overlay Pattern

```typescript
import * as PIXI from 'pixi.js';
import { useCursorStore } from '@/renderer/stores/cursorStore';

const cursorSprite = PIXI.Sprite.from('/assets/cursors/macos-default.png');
cursorSprite.anchor.set(0, 0);

app.ticker.add(() => {
  const { smoothedX, smoothedY, isClicking } = useCursorStore.getState();
  
  cursorSprite.x = smoothedX * app.screen.width;
  cursorSprite.y = smoothedY * app.screen.height;
  cursorSprite.scale.set(isClicking ? 0.85 : 1.0); // click bounce
});
```

---

## Adding Auto-Zoom Suggestions

Zoom suggestions are generated from cursor activity density:

```typescript
import type { CursorEvent, ZoomRegion } from '@/shared/types';

export function generateZoomSuggestions(
  cursorEvents: CursorEvent[],
  videoDuration: number,
  options = { minDwell: 0.8, zoomScale: 2.2 }
): ZoomRegion[] {
  const suggestions: ZoomRegion[] = [];
  const windowSize = 0.5; // seconds

  for (let t = 0; t < videoDuration; t += windowSize) {
    const window = cursorEvents.filter(
      (e) => e.timestamp / 1000 >= t && e.timestamp / 1000 < t + windowSize
    );

    if (window.length < 3) continue;

    const clicks = window.filter((e) => e.type === 'click');
    if (clicks.length === 0) continue;

    const avgX = clicks.reduce((s, e) => s + e.x, 0) / clicks.length;
    const avgY = clicks.reduce((s, e) => s + e.y, 0) / clicks.length;

    suggestions.push({
      id: crypto.randomUUID(),
      startTime: t,
      endTime: t + options.minDwell,
      targetX: avgX,
      targetY: avgY,
      scale: options.zoomScale,
      easing: 'ease-in-out',
    });
  }

  return suggestions;
}
```

---

## Frame Styling

```typescript
interface FrameStyle {
  background:
    | { type: 'wallpaper'; src: string }
    | { type: 'gradient'; stops: GradientStop[] }
    | { type: 'solid'; color: string };
  padding: number;          // px, applied uniformly
  borderRadius: number;     // px on video frame corners
  shadow: {
    enabled: boolean;
    blur: number;
    offsetY: number;
    color: string;
  };
  blur: number;             // background blur radius
}

// Example usage in project state
const frameStyle: FrameStyle = {
  background: {
    type: 'gradient',
    stops: [
      { offset: 0, color: '#1e1b4b' },
      { offset: 1, color: '#2563eb' },
    ],
  },
  padding: 48,
  borderRadius: 12,
  shadow: { enabled: true, blur: 40, offsetY: 20, color: 'rgba(0,0,0,0.5)' },
  blur: 0,
};
```

---

## Export Configuration

```typescript
interface ExportSettings {
  format: 'mp4' | 'gif';
  quality: 'low' | 'medium' | 'high';
  aspectRatio: '16:9' | '4:3' | '1:1' | '9:16' | 'original';
  outputWidth?: number;    // Override resolution width
  fps?: number;            // Default: 30 for MP4, 15 for GIF
  loop?: boolean;          // GIF only — use with cursor loop feature
}
```

---

## Platform-Specific Capture Notes

### macOS (ScreenCaptureKit)

- Requires macOS 12.3+
- Cursor excluded at capture level — always clean
- Native helper invoked via Electron `utilityProcess` or `child_process`

### Windows (WGC)

- Requires Windows 10 Build 19041+
- Uses `IsCursorCaptureEnabled(false)` to hide real cursor
- Falls back to Electron browser capture on older builds (cursor visible)
- Native WASAPI for system audio — works out of the box

### Linux

- Electron desktop capture only
- Cursor **cannot** be hidden — animated overlay will produce two cursors
- System audio requires PipeWire (Ubuntu 22.04+, Fedora 34+)

---

## Common Patterns

### Load and Render a Project

```typescript
import { loadProject } from '@/renderer/project/loader';
import { useEditorStore } from '@/renderer/stores/editorStore';

async function openExistingProject(filePath: string) {
  const project = await loadProject(filePath);
  useEditorStore.setState({
    project,
    playheadTime: 0,
    isPlaying: false,
  });
}
```

### Add a Zoom Region Programmatically

```typescript
import { useEditorStore } from '@/renderer/stores/editorStore';
import { v4 as uuid } from 'uuid';

function addZoom(startTime: number, endTime: number, x: number, y: number) {
  const { project } = useEditorStore.getState();
  const newRegion: ZoomRegion = {
    id: uuid(),
    startTime,
    endTime,
    targetX: x,
    targetY: y,
    scale: 2.0,
    easing: 'ease-in-out',
  };
  useEditorStore.setState({
    project: {
      ...project,
      zoomRegions: [...project.zoomRegions, newRegion],
    },
  });
}
```

### Cursor Smoothing Utility

```typescript
export function smoothCursorPath(
  events: CursorEvent[],
  smoothingFactor = 0.15
): CursorEvent[] {
  if (events.length < 2) return events;

  const smoothed = [...events];
  for (let i = 1; i < smoothed.length; i++) {
    smoothed[i] = {
      ...smoothed[i],
      x: smoothed[i - 1].x + (events[i].x - smoothed[i - 1].x) * smoothingFactor,
      y: smoothed[i - 1].y + (events[i].y - smoothed[i - 1].y) * smoothingFactor,
    };
  }
  return smoothed;
}
```

---

## Cursor Loop Feature

When cursor loop is enabled, the cursor position is interpolated back to its starting coordinates at the end of the video — creating a seamless GIF loop:

```typescript
function applyCursorLoop(events: CursorEvent[], videoDuration: number): CursorEvent[] {
  if (events.length === 0) return events;

  const startX = events[0].x;
  const startY = events[0].y;
  const loopWindowSec = 0.3; // blend back over last 300ms

  return events.map((e) => {
    const timeFromEnd = videoDuration - e.timestamp / 1000;
    if (timeFromEnd > loopWindowSec) return e;

    const t = 1 - timeFromEnd / loopWindowSec; // 0→1 as we approach end
    return {
      ...e,
      x: e.x + (startX - e.x) * t,
      y: e.y + (startY - e.y) * t,
    };
  });
}
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| "App cannot be opened" on macOS | Unsigned app quarantined | `xattr -rd com.apple.quarantine /Applications/Recordly.app` |
| Two cursors visible in recording on Linux | Cursor hiding unsupported; overlay is additive | Disable cursor overlay or contribute a Linux cursor-hide solution |
| Two cursors on Windows | Build older than 19041, fallback capture | Upgrade Windows or accept visible cursor |
| No system audio on Linux | PulseAudio instead of PipeWire | Upgrade to Ubuntu 22.04+ / Fedora 34+ for PipeWire |
| Recording black screen on macOS | SCKit permissions not granted | System Preferences → Privacy & Security → Screen Recording → enable Recordly |
| `npm run dev` fails with native module error | Native addons need rebuild | `npm run rebuild` or `npx electron-rebuild` |
| Zoom animation looks jerky | `easing` set to `linear` | Change `ZoomRegion.easing` to `'ease-in-out'` or `'spring'` |
| Export produces huge GIF | High fps or resolution | Set `fps: 15` and reduce `outputWidth` in `ExportSettings` |

---

## Contributing

Priority contribution areas per the maintainer:

1. **Linux cursor pipeline** — hide OS cursor during Electron capture
2. **Webcam bubble overlay** — picture-in-picture webcam during recording
3. **Localisation** — especially Chinese (`README.zh-CN.md` exists as a template)
4. **Export speed** — parallelise PixiJS frame rendering
5. **UI/UX improvements**

```bash
# Run tests
npm test

# Lint
npm run lint

# Type check
npm run typecheck
```

Keep PRs focused and modular. Test playback, editing, and export flows before submitting.

---

## Resources

- Homepage: https://recordly.dev
- Releases: https://github.com/webadderall/Recordly/releases
- Issues: https://github.com/webadderall/Recordly/issues
- Donate: https://ko-fi.com/webadderall/goal?g=0
- Twitter/X: [@webadderall](https://x.com/webadderall)
- Based on: [OpenScreen](https://github.com/siddharthvaddem/openscreen)
```
