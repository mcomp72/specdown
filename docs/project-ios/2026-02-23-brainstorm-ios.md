# Specdown for iOS & iPad — Brainstorm

> Planning document. No code yet — just thinking things through.

---

## Where We Are Today

Specdown (v0.0.52) now has two surfaces:

1. **Web app** — hosted at `cbremer.github.io/specdown`, works in any browser
2. **Desktop app** — Electron-based macOS `.dmg` with native file open, tabs, file watching

Both share the same `markdown-viewer/` codebase (~2,000 lines across `app.js`, `styles.css`, and `index.html`) plus four vendored libraries (Marked.js, Mermaid, Panzoom, Highlight.js). The desktop app wraps this web code in an Electron shell with an IPC bridge for native features.

### What the web codebase provides
- GitHub Flavored Markdown rendering via Marked.js
- Interactive Mermaid diagrams with zoom, pan, reset, and fullscreen (via Panzoom)
- Syntax-highlighted code blocks via Highlight.js
- Light/dark theme toggle
- Raw/preview mode toggle
- Drag-and-drop file loading

### What the desktop app adds
- Multi-file tabbed interface
- Native file open (`Cmd+O`) and macOS menus
- File watching with auto-reload
- Persistent state across sessions (in progress)
- `.md` file association (double-click to open)

---

## Why iOS & iPad?

### The case for mobile

- **Reading on the go** — Markdown specs, docs, and notes are useful to review away from a desk
- **iPad as a work surface** — Many people use iPads for reading and reviewing documents; markdown files shouldn't require a laptop
- **Share Sheet integration** — Open `.md` files from email attachments, Messages, Slack, iCloud Drive without needing a separate viewer
- **Files app integration** — Browse and open markdown files from any cloud provider (iCloud, Dropbox, Google Drive) through the standard iOS file picker
- **Mermaid diagrams on touch** — No good mobile app exists for viewing Mermaid diagrams with real interactivity (pinch-to-zoom, pan)

### What makes this different from just using Safari

The web version technically works in mobile Safari, but:
- No optimized touch targets or gestures
- No Files app / Share Sheet integration
- Mermaid diagram interaction (Panzoom) works but feels clunky without touch optimization
- No offline support
- No way to "Open In" from other apps
- Browser chrome eats screen real estate, especially on iPhone

### Non-goals (at least initially)

- **Editing** — This is a viewer. Edit in your tool of choice, view in Specdown.
- **Sync service** — No custom cloud backend. Rely on iCloud Drive, Files app, and existing cloud providers.
- **Android** — Start with Apple platforms. Revisit later if demand exists.
- **Watch / TV** — No. Markdown on a 44mm screen is not useful.

---

## Framework Options

Four realistic paths for bringing Specdown to iOS/iPad. The key question is how much of the existing web codebase (`markdown-viewer/`) we can reuse.

### Option A: Swift + WKWebView (Hybrid)

| | |
|---|---|
| **What it is** | A native Swift app that embeds WKWebView to render the existing web frontend |
| **App size** | ~5–15 MB (web assets + thin native shell) |
| **Language** | Swift (native shell) + existing JS/HTML/CSS (viewer) |
| **Code reuse** | Very high — same `markdown-viewer/` files loaded into WKWebView |
| **iOS APIs** | Full access — Files app, Share Sheet, multitasking, system appearance |

**Pros:**
- Maximum code reuse — the exact same rendering code runs on web, desktop, and iOS
- Mermaid, Marked.js, Highlight.js, and Panzoom all work in WKWebView
- Native Swift shell handles all iOS-specific features (file picker, share extension, multitasking)
- Small app size — no bundled browser engine (WKWebView is system-provided)
- Consistent rendering across all three platforms

**Cons:**
- WKWebView has subtle iOS-specific quirks (viewport management, safe area insets, keyboard handling)
- JS ↔ Swift bridge (`WKScriptMessageHandler`) adds IPC complexity — but we already solved this pattern with Electron's `preload.js`
- Panzoom touch gestures may conflict with iOS system gestures (swipe back, scroll) — needs careful tuning
- Debugging web content in WKWebView is less convenient than a normal browser

**Best for:** Getting to a working app quickly while reusing all existing rendering code.

---

### Option B: SwiftUI Native (Full Rewrite)

| | |
|---|---|
| **What it is** | A fully native iOS app using SwiftUI for UI and native Swift libraries for markdown/Mermaid rendering |
| **App size** | ~3–10 MB |
| **Language** | Swift / SwiftUI only |
| **Code reuse** | None from web — full rewrite of rendering logic |
| **iOS APIs** | Full native access, best possible integration |

**Pros:**
- Most "iOS-native" result — system animations, gestures, accessibility all built-in
- No WebView overhead or quirks
- Best performance for large documents (native text rendering)
- Full access to SwiftUI features (NavigationStack, sheets, Document-based app template)
- Could use Apple's `AttributedString` for markdown (basic GFM support in iOS 15+)

**Cons:**
- **Mermaid is the hard part** — no mature native Swift Mermaid renderer exists. Would need to either:
  - Embed a WKWebView just for diagrams (defeating the purpose of going fully native)
  - Use a Swift SVG renderer and pre-render Mermaid to SVG at build/load time
  - Skip Mermaid entirely on v1 and add it later
- Every rendering feature needs to be reimplemented: GFM tables, task lists, syntax highlighting, etc.
- Two codebases to maintain — web rendering logic and native rendering logic would diverge
- Significantly more development time

**Best for:** The long-term "ideal" if we want a first-class native experience — but the Mermaid problem makes this impractical for v1.

---

### Option C: React Native + WebView

| | |
|---|---|
| **What it is** | A React Native app using `react-native-webview` to host the existing web frontend |
| **App size** | ~30–50 MB (React Native runtime + web assets) |
| **Language** | JavaScript (React Native shell) + existing JS/HTML/CSS (viewer) |
| **Code reuse** | High for viewer, but native shell is React Native (not shared with desktop) |
| **iOS APIs** | Good but through React Native bridges (file picker, sharing, etc.) |

**Pros:**
- All JavaScript — no Swift required
- WebView hosts the existing viewer code unchanged
- React Native provides access to native iOS APIs through JS bridges
- Could theoretically target Android later with the same React Native shell
- Large ecosystem of community packages

**Cons:**
- React Native adds significant bundle size and runtime overhead for what is essentially a WebView wrapper
- Adds a new framework dependency (React Native) to a project that currently has zero framework dependencies
- React Native ↔ native bridge is another layer of abstraction on top of the WebView bridge
- More complex build toolchain (Metro bundler, CocoaPods, etc.)
- The project philosophy is "no frameworks" — introducing React Native goes against that grain

**Best for:** If cross-platform mobile (iOS + Android) is a near-term priority and the team prefers all-JavaScript.

---

### Option D: Progressive Web App (PWA)

| | |
|---|---|
| **What it is** | Enhance the existing web app with a Service Worker, Web App Manifest, and mobile-optimized CSS |
| **App size** | 0 MB (no install needed) or ~1 MB if added to home screen |
| **Language** | Existing JS/HTML/CSS + Service Worker |
| **Code reuse** | 100% — it's the same app |
| **iOS APIs** | Very limited — no Share Sheet target, no Files app integration, limited offline |

**Pros:**
- Zero additional codebase — just enhance what exists
- Works immediately on all platforms (iOS, Android, desktop browsers)
- Add to Home Screen gives an "app-like" experience
- Service Worker enables offline viewing of previously loaded files
- No App Store review process

**Cons:**
- **iOS severely limits PWAs** — no background processing, limited storage, Apple regularly clears PWA data
- No Share Sheet integration (can't receive files from other apps)
- No Files app integration (no document picker beyond `<input type="file">`)
- Can't register as a file handler for `.md` files
- Safari's PWA support is minimal compared to Chrome on Android
- Feels like a webpage, not an app — no native navigation, no system integration
- Apple's stance on PWAs has been actively hostile (they almost removed them from EU iOS)

**Best for:** A quick mobile-friendly improvement to the web version — but not a replacement for a real iOS app.

---

### Framework Comparison

| Criteria | Swift + WKWebView | SwiftUI Native | React Native | PWA |
|---|---|---|---|---|
| Code reuse from web | Very high | None | High (viewer) | 100% |
| Mermaid support | Works (WKWebView) | Very hard | Works (WebView) | Works |
| App size | ~5–15 MB | ~3–10 MB | ~30–50 MB | ~0 MB |
| Files app integration | Yes | Yes | Yes (via bridge) | No |
| Share Sheet (receive) | Yes | Yes | Yes (via bridge) | No |
| Offline support | Yes | Yes | Yes | Limited |
| iPad multitasking | Yes | Yes (best) | Yes | Basic |
| Development time | Moderate | High | Moderate | Low |
| Maintenance burden | Low (shared code) | High (separate code) | Medium | Lowest |
| Native feel | Good | Best | Good | Poor |

---

## Touch Interaction Considerations

This is one of the most important design questions. The existing Panzoom-based Mermaid interaction works with mouse events. On iOS, we need to handle touch.

### Mermaid Diagrams

**Current behavior (desktop/web):**
- Mouse wheel to zoom
- Click-and-drag to pan
- Reset button to return to original view
- Fullscreen button for expanded view

**Needed for touch:**
- **Pinch-to-zoom** — Panzoom already supports touch events, but needs testing on iOS WKWebView
- **Two-finger pan** — Drag to pan, but need to avoid conflicts with page scroll
- **Double-tap to zoom** — Common iOS pattern, could zoom to fit or zoom to a specific level
- **Gesture conflict resolution** — Single-finger scroll should scroll the page, two-finger gestures should control the diagram. Need a clear modal boundary (e.g., tap diagram to "enter" interactive mode)

**Possible approach:**
```
┌──────────────────────────────────┐
│  [Diagram Title]                 │
│  ┌────────────────────────────┐  │
│  │                            │  │
│  │   [Mermaid Diagram]        │  │ ← Tap to enter interactive mode
│  │                            │  │ ← Then pinch/pan works
│  │                            │  │ ← Tap outside to exit
│  └────────────────────────────┘  │
│  [Fullscreen] [Reset]            │
└──────────────────────────────────┘
```

### General Touch Targets

- All buttons need to meet iOS minimum 44×44pt touch target size
- Theme toggle, raw/preview toggle, file actions all need larger tap areas than the current web UI
- Consider swipe gestures for tab switching (swipe left/right between open files)

---

## Code Sharing Strategy

### Recommended: Same Repo, Three Deployment Paths

Following the pattern established by the desktop app, the iOS app would live in the same repo and share the `markdown-viewer/` codebase.

```
specdown/
├── markdown-viewer/         ← Shared (web + desktop + iOS)
│   ├── index.html
│   ├── styles.css
│   └── app.js
├── desktop/                 ← Electron shell (macOS desktop)
│   ├── main.js
│   └── preload.js
├── ios/                     ← NEW: iOS/iPad app (Xcode project)
│   ├── SpecDown.xcodeproj/
│   ├── SpecDown/
│   │   ├── App.swift        ← App entry point
│   │   ├── ContentView.swift← Main view with WKWebView
│   │   ├── FileManager.swift← iOS file picking / sharing
│   │   ├── WebBridge.swift  ← JS ↔ Swift IPC (like preload.js)
│   │   └── Assets.xcassets/ ← App icon, colors
│   └── SpecDown.entitlements
├── docs/
│   ├── project-desktop/
│   └── project-ios/         ← NEW
└── .github/workflows/
    ├── static.yml           ← Web deploy (GitHub Pages)
    ├── desktop.yml          ← Desktop build (.dmg)
    └── ios.yml              ← NEW: iOS build (TestFlight / App Store)
```

### What's Shared vs. Separate

| Shared (all platforms) | iOS-only |
|---|---|
| `markdown-viewer/` (HTML, JS, CSS) | Swift native shell (`ios/`) |
| Vendored libraries (Marked, Mermaid, Panzoom, Highlight.js) | Xcode project configuration |
| Core rendering and diagram logic | iOS-specific touch handling |
| CSS theming (light/dark via CSS variables) | Files app / Share Sheet integration |
| | iOS CI workflow |
| | App Store assets (screenshots, metadata) |

### IPC Bridge Pattern

The desktop app established a bridge pattern with Electron's `preload.js`:
- **Native → Web**: "Here's file content", "File changed", "User opened from Finder"
- **Web → Native**: "User wants to open a file", "Save state"

The iOS version would use the same concept via `WKScriptMessageHandler`:
- **Swift → JS**: `webView.evaluateJavaScript("loadFile(...)")`
- **JS → Swift**: `window.webkit.messageHandlers.specdown.postMessage({...})`

This is architecturally identical to the Electron IPC. We could potentially define a shared interface that both `preload.js` and `WebBridge.swift` implement.

---

## iOS & iPad-Specific Features

### iPhone
- Share Sheet target — receive `.md` files from other apps
- Files app integration — browse and open markdown from any provider
- Compact UI — single file view, swipe between files
- System appearance — follow iOS light/dark mode automatically
- Dynamic Type — respect user's text size preferences

### iPad
- **Split View / Slide Over** — use Specdown alongside your markdown editor
- **Drag and drop** — drag `.md` files from Files app onto Specdown
- **Stage Manager** — resize freely on iPadOS 16+
- **Sidebar** — file list in a collapsible sidebar (iPad has room for this)
- **Keyboard shortcuts** — `Cmd+O`, `Cmd+W`, etc. for iPad with keyboard
- **Apple Pencil** — probably not useful for a viewer, but worth noting

### Both
- Spotlight indexing — search file contents from Spotlight
- Quick Look preview — provide a Quick Look extension so `.md` files preview in Files app
- Handoff — start viewing on Mac, continue on iPad (or vice versa)
- Widget — show recently opened files in a home screen widget

---

## Distribution

| Method | Pros | Cons |
|---|---|---|
| **App Store** | Discoverability, automatic updates, trusted source | Review process, 30% cut (if paid), sandboxing requirements |
| **TestFlight** | Easy beta testing, no review for internal builds | 90-day expiry, limited to 10,000 testers |
| **Ad Hoc / Enterprise** | Full control | Requires device registration, not scalable |

**Recommendation:** App Store for release, TestFlight for beta testing. The app is free and has no server component, so the App Store process should be straightforward.

---

## Open Questions

- **Which framework?** Swift + WKWebView seems like the strongest option given the existing codebase, but needs prototyping to validate Panzoom touch behavior in iOS WKWebView.
- **Minimum iOS version?** iOS 16 would cover ~95% of devices and gives us NavigationStack, Stage Manager support, and modern WKWebView features. iOS 15 is more conservative but adds compatibility burden.
- **Universal app or separate iPhone/iPad builds?** Universal is standard practice and recommended — one binary that adapts to the device.
- **How to handle Panzoom on touch?** Need to prototype the gesture conflict between page scroll and diagram interaction. The "tap to activate" pattern used by Google Maps in embedded contexts might work.
- **Offline support scope?** Should the app cache previously viewed files? Or only show files actively available on the file system?
- **State persistence on iOS** — `UserDefaults` for preferences, but what about open file references? Security-scoped bookmarks work differently on iOS vs macOS.
- **Should the app name be "Specdown" everywhere, or "Specdown Viewer" on iOS?** App Store has naming guidelines worth checking.
- **CI for iOS** — GitHub Actions has macOS runners with Xcode. Can we build and deploy to TestFlight from CI? (Yes — `xcodebuild` + `altool` or Fastlane.)

---

## Possible MVP Scope

The smallest useful iOS app:

1. **Swift + WKWebView shell** — loads `markdown-viewer/` in a WKWebView
2. **iOS file picker** — open `.md` files from Files app
3. **Share Sheet target** — receive `.md` files from other apps
4. **Touch-optimized Mermaid** — validate Panzoom works with pinch/pan, add tap-to-activate if needed
5. **System appearance** — follow iOS light/dark mode
6. **iPad multitasking** — Split View and Slide Over support
7. **Single file view** — start with one file at a time (tabs can come later)

That gets us from "web page in Safari" to "real iOS app that opens markdown files with touch-friendly diagram interaction." Multi-file tabs, sidebar, Spotlight indexing, and widgets can come in later iterations.

---

*Last updated: 2026-02-23*
