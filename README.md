# scoop-bucket

Scoop bucket with portable Windows applications.

## Installation

```powershell
scoop bucket add ruslan-rv-ua https://github.com/ruslan-rv-ua/scoop-bucket
```

## Available Apps

### [AudioCaptor](https://github.com/ruslan-rv-ua/AudioCaptor)

A portable system audio recorder for Windows. Captures microphone and system audio simultaneously into a single WAV file.

**Key features:**
- Records microphone and system audio (loopback) in real-time
- Real-time mixing with volume control per source
- Global hotkey with short/long press detection (start, pause, stop)
- Sound notifications for recording state changes
- Full keyboard navigation and screen reader support
- Portable single executable — all data stored next to the app

```powershell
scoop install audiocaptor
```

---

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

### [Marka](https://github.com/ruslan-rv-ua/marka)

A fast, accessible Markdown file viewer for Windows. Opens `.md` files with clean rendering, adjustable typography, and full screen reader support.

**Key features:**
- Opens Markdown files via command line, file dialog, or drag & drop
- Adjustable font size and padding (Ctrl+±, Ctrl+[/])
- Light/dark theme toggle (Ctrl+T), follows system preference
- Full NVDA/Narrator accessibility — all elements labeled, code blocks navigable
- Portable — single executable with settings.json next to it

```powershell
scoop install marka
```

---

## Update all apps

```powershell
scoop update *
```
