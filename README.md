# scoop-bucket

Scoop bucket with portable Windows applications.

## Installation

```powershell
scoop bucket add ruslan-rv-ua https://github.com/ruslan-rv-ua/scoop-bucket
```

## Available Apps

### [BrowserSelector](https://github.com/ruslan-rv-ua/BrowserSelector)

A lightweight portable Windows app that intercepts link clicks and asks you which browser to open them with. Set it as your default browser and take full control over how URLs are handled.

**Key features:**
- Presents a menu every time a link is clicked — choose any browser or shell command
- Configurable auto-open timer (1–10 sec) that falls back to your preferred browser automatically
- Any shell command can be added as an option (e.g. copy to clipboard, open in incognito, pipe to a script)
- Full keyboard navigation (arrows, numbers 1–9, Enter, Escape)
- Screen reader support (NVDA, Windows Narrator)
- Interface available in 9 languages, auto-detected from system locale
- Portable single executable + JSON config file

```powershell
scoop install browserselector
```

---

### [QuickSnippets](https://github.com/ruslan-rv-ua/quick-snippets)

A portable text snippet launcher — press a global hotkey from any application, search by title, and copy the snippet to the clipboard in seconds. No cloud, no telemetry, no installer.

**Key features:**
- Global hotkey **Ctrl+Alt+Space** opens the window from anywhere
- Fuzzy search filters snippets as you type
- Sensitive snippets can be encrypted locally with AES-256-GCM (keys never leave your device)
- Fully keyboard-driven; mouse is optional
- Screen reader support (NVDA, JAWS, Windows Narrator)
- Light and dark themes, auto-switched based on system preference
- Portable — the entire app lives in one folder; database and settings sit next to the executable
- System tray icon for quick access; auto-hides when you switch away

```powershell
scoop install quick-snippets
```

---

### [Axygen Shot](https://github.com/ruslan-rv-ua/axygen-shot)

A portable Windows CLI screenshot tool designed for blind developers. Captures
a single window to PNG, copies to clipboard, with audio feedback.

**Key features:**
- Single-window capture via PrintWindow/BitBlt
- Watch mode with global hotkey and system tray icon
- Dual binaries: `shot.exe` (CLI) + `shot-watch.exe` (tray daemon)
- Audio feedback via Windows system sounds
- Full screen reader support

```powershell
scoop install axygen-shot
```

---

## Update all apps

```powershell
scoop update *
```
