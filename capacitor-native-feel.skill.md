---
name: capacitor-native-feel
version: 1.0
description: >
  Wrap any web or mobile source code as a production-grade native Android APK
  using Capacitor 7. Covers Next.js / React / Vue / Svelte static-export
  conversion, native-feel UX (haptics, back-button bridge, immersive mode,
  external storage, file associations), the KaTeX/font bundling fix, the
  build pipeline (JDK + Android SDK + Gradle), and delivery. Distilled from
  CLIL Engine v2.0 production builds. Use this skill whenever the user wants
  to turn a web/JS source tree into a native Android app with native UX.
scope: broad
# Any web/mobile framework → native Android APK. Primary depth is
# Capacitor 7 + Next.js 16; secondary coverage for React/Vue/Svelte +
# Capacitor. Tertiary awareness of React Native/Expo/Flutter (refer user
# to specialized tooling — do not attempt those builds with this skill).
invocation: auto
# Auto-invoke when trigger keywords are detected in the user's request.
# No need for the user to explicitly say "use the skill".
keywords:
  # Exact-match — high confidence
  exact:
    - capacitor
    - capacitor 7
    - nextjs apk
    - next.js apk
    - android apk from nextjs
    - web to native
    - web to apk
    - native feel
    - native-feel
    - native ux
  # Contextual — fire when combined with web/JS context
  contextual:
    - apk
    - android app
    - mobile app
    - side-loadable
    - play store
    - haptics
    - back button
    - immersive mode
    - offline-first app
    - webview wrapper
# Recommended co-loaded references (read these when the skill fires):
references:
  - path: /home/z/my-project/download/Capacitor-Native-Feel-Playbook.md
    reason: Full 58-feature playbook with all code snippets and 20-error fix catalog
  - path: /home/z/my-project/skills/capacitor-native-feel/SKILL.md
    reason: This skill file (self-reference for nested agents)
---

# Capacitor Native-Feel Skill

## 1. What This Skill Does

Wraps a web/JS source tree as a production-grade native Android APK using **Capacitor 7**. Produces a side-loadable release APK with native UX: back-button bridge, haptics, immersive mode, external storage, file associations, KaTeX/font bundling fix, signed release build.

**Primary depth:** Capacitor 7 + Next.js 16 static export → Android APK.
**Secondary:** React/Vue/Svelte + Vite/Webpack static build → Capacitor → Android APK.
**Tertiary (refer out):** React Native, Expo, Flutter — do NOT attempt with this skill. Tell the user these need different toolchains.

## 2. When to Invoke (Decision Tree)

```
User request mentions "apk" / "android app" / "native" / "mobile" / "capacitor"
│
├─ Is the source a Next.js app? ────────────────── YES → Follow primary path (Chapter A below)
│   └─ NO → Is it React/Vue/Svelte + Vite? ────── YES → Follow secondary path (skip /api + Prisma strip)
│       └─ NO → Is it React Native / Expo / Flutter? ── YES → STOP. Tell user this skill doesn't cover that.
│           └─ NO → Ask user what framework; if unknown, attempt primary path with caveats.
│
├─ Does the user want native UX (haptics, back button, immersive)? ── YES → Apply feature catalog (Chapter C)
│   └─ NO (just wants a basic APK wrapper) → Skip feature catalog, just do Chapters A + B + D
│
├─ Does the source use KaTeX / Font Awesome / Material Icons? ── YES → Apply font bundling fix (Chapter B.4)
│   └─ NO → Skip B.4
│
└─ Does the user want persistence (data survives app close)? ── YES → Apply storage stack (Chapter C.6)
    └─ NO → Skip C.6, use localStorage only
```

## 3. Chapter A — Primary Path: Next.js → APK (Critical Steps)

### A.1 Audit the source

Before touching anything, check for server-side deps that will break static export:
- `src/app/api/` — must be deleted + logic inlined client-side
- `prisma/` + `src/lib/db.ts` + `.env` with `DATABASE_URL` — must be deleted
- `output: "standalone"` in next.config — must become `"export"`
- `public/sw.js` + `manifest.json` + ServiceWorkerRegister — must be deleted
- `next-auth`, `next-intl`, `sharp`, `@prisma/client` in package.json — must be removed

### A.2 Convert next.config.ts

```ts
import type { NextConfig } from "next";
const nextConfig: NextConfig = {
  output: "export",
  assetPrefix: "./",
  images: { unoptimized: true },
  trailingSlash: true,
  typescript: { ignoreBuildErrors: true },
  eslint: { ignoreDuringBuilds: true },
  reactStrictMode: false,
};
export default nextConfig;
```

### A.3 Inline /api routes client-side

For each `/api/foo` route: port the logic into a store action or lib function. Update `fetch("/api/foo")` calls to call the inlined function. The GitHub API and most public APIs support CORS, so direct fetches from the WebView work.

### A.4 Install Capacitor + plugins

```bash
cd /home/z/my-project/source
npm install --legacy-peer-deps \
  @capacitor/core @capacitor/cli @capacitor/android \
  @capacitor/app @capacitor/browser @capacitor/filesystem \
  @capacitor/haptics @capacitor/keyboard @capacitor/network \
  @capacitor/preferences @capacitor/screen-orientation \
  @capacitor/share @capacitor/splash-screen @capacitor/status-bar \
  @capacitor/local-notifications @capacitor/assets \
  @capacitor-community/sqlite @capacitor-community/keep-awake \
  @capawesome/capacitor-file-picker jszip
```

### A.5 Write capacitor.config.ts

```ts
import type { CapacitorConfig } from "@capacitor/cli";
const config: CapacitorConfig = {
  appId: "com.yourcompany.yourapp",
  appName: "Your App",
  webDir: "out",
  android: { allowMixedContent: true, webContentsDebuggingEnabled: true },
  server: { androidScheme: "https" },
  plugins: {
    SplashScreen: {
      launchShowDuration: 1500, launchAutoHide: true,
      backgroundColor: "#0b0b0d", androidScaleType: "CENTER_CROP",
      splashFullScreen: true, splashImmersive: true,
    },
    LocalNotifications: { smallIcon: "ic_stat_icon", iconColor: "#0b0b0d", sound: "bell.wav" },
    StatusBar: { backgroundColor: "#0b0b0d", style: "DARK", overlaysWebView: false },
    ScreenOrientation: { orientation: "portrait" },
  },
};
export default config;
```

## 4. Chapter B — Build Pipeline

### B.1 Toolchain (one-time per workspace)

**JDK 21 (portable Temurin — system JRE lacks javac):**
```bash
curl -fSL -o /tmp/jdk.tar.gz \
  "https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.5%2B11/OpenJDK21U-jdk_x64_linux_hotspot_21.0.5_11.tar.gz"
mkdir -p /home/z/my-project/jdk && tar -xzf /tmp/jdk.tar.gz -C /home/z/my-project/jdk/
```

**Android SDK cmdline-tools v9 (NOT v10 — v10 has broken sdkmanager layout):**
```bash
export ANDROID_HOME=/home/z/my-project/android-sdk
mkdir -p $ANDROID_HOME/cmdline-tools
cd /tmp
curl -fsSL -o cmdline-tools.zip \
  https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip
unzip -q cmdline-tools.zip -d $ANDROID_HOME/
mv $ANDROID_HOME/cmdline-tools $ANDROID_HOME/cmdline-tools-tmp
mkdir -p $ANDROID_HOME/cmdline-tools/latest
mv $ANDROID_HOME/cmdline-tools-tmp/* $ANDROID_HOME/cmdline-tools/latest/
rm -rf $ANDROID_HOME/cmdline-tools-tmp cmdline-tools.zip

export JAVA_HOME=/home/z/my-project/jdk/jdk-21.0.5+11
export PATH=$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH

yes | sdkmanager --licenses > /tmp/licenses.log 2>&1
sdkmanager --install "platform-tools" "platforms;android-34" "build-tools;34.0.0"
```

**Env vars (set once per session):**
```bash
export JAVA_HOME=/home/z/my-project/jdk/jdk-21.0.5+11
export ANDROID_HOME=/home/z/my-project/android-sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
export PATH=$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH
```

### B.2 Workspace hygiene

Sandbox `/tmp` is wiped between sessions. Install JDK + Android SDK to `/home/z/my-project/` which persists. Long downloads (JDK 200MB, SDK 150MB, npm 500MB+) time out the bash tool — run via `setsid + nohup` and poll with short `sleep` checks.

### B.3 The build loop

```bash
npm run build                  # Next.js static export → out/
npx cap sync android           # Copy out/ → android/app/src/main/assets/public/
cd android
./gradlew assembleDebug        # → app/build/outputs/apk/debug/app-debug.apk
./gradlew assembleRelease      # → app/build/outputs/apk/release/app-release.apk
```

### B.4 KaTeX / Font Awesome / Material Icons font fix (THE most common visual bug)

If the source uses KaTeX, Font Awesome, Material Icons, Bootstrap Icons, or ANY vendor CSS with `url()` font references, the fonts won't load in the APK. Symptom: math renders as raw text, icons render as boxes.

**4-step fix:**
```bash
# 1. Copy fonts to public/vendor-fonts/
mkdir -p public/katex-fonts
cp node_modules/katex/dist/fonts/*.woff2 public/katex-fonts/
cp node_modules/katex/dist/fonts/*.woff public/katex-fonts/
cp node_modules/katex/dist/fonts/*.ttf public/katex-fonts/
```

```css
/* 2. Create src/styles/katex-fonts.css with ABSOLUTE URLs (relative breaks Next.js build) */
@font-face {
  font-family: 'KaTeX_AMS';
  src: url(/katex-fonts/KaTeX_AMS-Regular.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_AMS-Regular.woff) format('woff'),
       url(/katex-fonts/KaTeX_AMS-Regular.ttf) format('truetype');
  font-weight: normal; font-style: normal;
}
/* ... 19 more @font-face rules for all KaTeX families ... */
```

```ts
// 3. Import AFTER vendor CSS in layout.tsx
import "katex/dist/katex.min.css";
import "@/styles/katex-fonts.css";  // MUST come after katex.min.css
```

```bash
# 4. Verify in APK
unzip -l app-release.apk | grep katex-fonts | wc -l   # should be 60
```

### B.5 AndroidManifest customization

Add permissions: `INTERNET`, `VIBRATE`, `ACCESS_NETWORK_STATE`, `POST_NOTIFICATIONS`, `WAKE_LOCK`, `READ_MEDIA_IMAGES`/`VIDEO`/`AUDIO`, `READ_EXTERNAL_STORAGE` with `maxSdkVersion=32`. Add `android:usesCleartextTraffic="true"` and `android:hardwareAccelerated="true"`. Add intent filters for file type associations (see C.9).

### B.6 Persistent release keystore

```bash
keytool -genkeypair -v \
  -keystore /home/z/my-project/scripts/app-release.keystore \
  -storepass android -keypass android \
  -alias app-release \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -dname "CN=App Name, OU=Team, O=Company, L=City, ST=State, C=XX"
```

In `android/app/build.gradle`, set `signingConfigs.debug` to point at this keystore and `buildTypes.release.signingConfig = signingConfigs.debug`. Release APK is now directly side-loadable.

## 5. Chapter C — Native-Feel Feature Catalog

Apply these based on user request. Each is briefly described — see the playbook for full code.

### C.1 Back-button bridge (CRITICAL — always apply)
`@capacitor/app` `backButton` listener with priority chain: toasts → dialogs (dispatch Escape) → sheets (custom event) → pages (`closeScreen()`) → exit-confirm. Without this, back closes the app instantly. Also flush pending storage writes on `appStateChange` to background.

### C.2 Storage stack (apply if user wants persistence)
4-layer: (1) `@capacitor/filesystem` JSON to `Android/data/<pkg>/files/apps_data/` (atomic tmp→bak→primary, 50ms debounce), (2) SQLite mirror via `@capacitor-community/sqlite`, (3) `localStorage` sync cache on every `setItem`, (4) `@capacitor/preferences` for small key-value. Multi-trigger flush: `visibilitychange` + `pagehide` + `beforeunload` + 3s interval. **CRITICAL**: `skipHydration: true` + manual `rehydrate()` to prevent race where empty default state overwrites persisted state.

### C.3 Haptics (apply for native feel)
Lazy-loaded wrapper around `@capacitor/haptics`: `light()` on button press, `success()`/`error()` on answer submit, `medium()` on long-press confirm, `medium()` on pull-to-refresh trigger, `light()` on toast dismiss. All imports use dynamic `import()` guarded by `Capacitor.isNativePlatform()`.

### C.4 Long-press context menu
`useLongPress(ref, {onLongPress, delay:500, tolerance:12})` hook. Touch + mouse + right-click. Portal-rendered floating menu at long-press coords, auto-clamps to viewport, closes on outside tap / Escape / scroll. Convert list item `<button>` to `<div role="button" tabIndex={0}>` so long-press doesn't conflict with button click semantics.

### C.5 Pull-to-refresh
`usePullToRefresh(ref, {onRefresh, threshold:80, max:120})` hook. Rubber-band physics (delta * 0.5), spinning indicator, haptic on trigger. Works on web + native.

### C.6 Immersive content mode
`useContentScreenNativeMode(active)` hook bundles: orientation lock to portrait + keep-awake + status bar hide. Apply with `active = !!activeScreenId`.

### C.7 Status bar theme match
`@capacitor/status-bar` `setStyle` + `setBackgroundColor`, re-apply on theme toggle via `MutationObserver` on `<html>` class.

### C.8 Network indicator
`@capacitor/network` `getStatus()` + `addListener("networkStatusChange")`. Render "Offline" badge in header.

### C.9 File type associations
AndroidManifest intent filter: `VIEW` action, `content`/`file`/`http`/`https` schemes, `pathPattern .*\.ext` PLUS nested `.*\..*\.ext` and `.*\..*\..*\.ext` (Android pathPattern is simple match, not regex — needs nested patterns for subdirectories). Handle opens via `@capacitor/app` `addListener("appUrlOpen")`.

### C.10 Native file picker
`@capawesome/capacitor-file-picker` `pickFiles({readData:true, types:[...]})`. Decode base64 via `atob` + `TextDecoder`. Falls back to `<input type="file">` on web.

### C.11 Share via native sheet
`@capacitor/share` `share({title, text, url})`. Falls back to `navigator.share` then `clipboard.writeText`.

### C.12 Local notifications
`@capacitor/local-notifications` `schedule({notifications:[{id, title, body, schedule:{at, repeats:true, every:"day"}}]})`. Requires `POST_NOTIFICATIONS` on Android 13+. Always `requestPermissions()` first, cancel existing before re-scheduling.

### C.13 Text selection policy
Global `* { user-select: none; -webkit-touch-callout: none; -webkit-tap-highlight-color: transparent; }`. Re-enable in content areas via `.selectable` class. Settings toggle for global selection via `.selectable-everywhere` class (but keep buttons/links/toasts non-selectable).

### C.14 Toast dismiss (tap + swipe + haptic)
Sonner config: `closeButton`, `swipeDirections={["left","right","up","down"]}`, `offset="80px"`. Capture-phase click listener finds `[data-sonner-toast]` and calls `toast.dismiss(id)` + `haptic.light()`.

### C.15 Safe area insets
`viewportFit: "cover"` in layout + CSS `env(safe-area-inset-*)` + JS probe hook for programmatic access.

### C.16 Splash + icons
`@capacitor/assets generate --android` from `resources/icon.png` (1024×1024) + `resources/splash.png` (1024×1024 dark).

## 6. Failure Modes (Top 5 — Recognize + Recover Fast)

### F1. "npm ERESOLVE peer dep conflict"
**Cause:** Capacitor 7 plugins have peer-dep conflicts with each other.
**Fix:** `npm install --legacy-peer-deps`. If a specific plugin still fails, check the correct v7 name (e.g. `@capacitor-community/keep-awake` NOT `keep-screen-awake`; `@capawesome/capacitor-file-picker` NOT `@capawesome-team/...`).

### F2. "Could not find or load main class SdkManagerCli"
**Cause:** You downloaded cmdline-tools v10 which has a broken layout.
**Fix:** Delete everything, re-download v9 (`commandlinetools-linux-9477386_latest.zip`). v9 extracts to `{bin, lib}`; v10 extracts to a nested directory missing the classpath jar.

### F3. KaTeX / icons render as raw text in the APK
**Cause:** Vendor CSS `url(fonts/X.woff2)` doesn't resolve under `capacitor://localhost`. Three compounding issues: Next.js doesn't rewrite `url()` in node_modules CSS, fonts aren't copied to `out/`, Capacitor can't fetch above `public/`.
**Fix:** See Chapter B.4 — copy fonts to `public/katex-fonts/`, write override `@font-face` with absolute `/katex-fonts/` URLs (relative breaks Next.js build), import after vendor CSS, verify with `unzip -l app-release.apk | grep katex-fonts | wc -l`.

### F4. Data loss on app close
**Cause:** Three issues: (a) debounced write doesn't flush before kill, (b) zustand auto-hydrate races with user interaction, (c) `localStorage` wiped with cache.
**Fix:** See Chapter C.2 — multi-trigger flush (`visibilitychange` + `pagehide` + `beforeunload` + 3s interval), `skipHydration: true` + manual `rehydrate()`, use `@capacitor/filesystem` `Directory.External` as primary store.

### F5. Bash tool timeouts during long downloads
**Cause:** JDK (200MB), Android SDK (150MB), npm install (500MB+) take longer than the bash tool's 2-minute default.
**Fix:** Run via `nohup setsid bash -c '...' & disown` so they're detached. Poll with `sleep 30 && tail /tmp/log && ls -la /tmp/file`. Write a master `bg-setup.sh` script that does JDK + SDK + npm sequentially in the background.

## 7. User Communication Guide

### When to give progress updates
- **After each major step** (toolchain ready, source migrated, build succeeded, APK signed). One line each.
- **Before long operations** ("Starting npm install — this takes 5-10 min"). Sets expectation.
- **On error** — immediately, with the error message + your fix plan. Don't silently retry more than twice.

### When to ask clarifying questions
- **Before starting work** — batch all questions in ONE `AskUserQuestion` call. Never drip questions across multiple turns.
- **Feature selection** — always offer a toggle list (e.g. "Select native-feel features to include: [haptics] [immersive] [notifications] ..."). Users love toggles.
- **App identity** — package name, app name, version. Don't assume.
- **Build variant** — debug, release, or both.

### When NOT to ask
- Don't ask "should I apply the KaTeX fix?" if the source uses KaTeX. Just apply it.
- Don't ask "should I bridge the back button?" — always bridge it.
- Don't ask "should I use persistent storage?" — always use the 4-layer stack.
- Only ask when the user's intent is genuinely ambiguous.

### How to report errors to the user
- Be specific: "npm ERESOLVE peer dep conflict on @capacitor-community/sqlite@6 — fixed by upgrading to v7"
- Show the fix: "I added `--legacy-peer-deps` and re-ran npm install"
- Show the outcome: "Install completed in 44s, 1013 packages added"
- Never blame the user or the system. Own it and fix it.

### How to deliver
- Always copy APKs to `/home/z/my-project/download/` with descriptive names (`App-Name-vX.Y-debug.apk`, `App-Name-vX.Y-release.apk`)
- Upload to CloudVault if available (`curl -X POST https://cloudvaultapi.space-z.ai/api/upload -F "file=@path"`)
- Write a README.md with install instructions + feature list
- Mention the download links explicitly in your final message (the IM gateway sometimes needs the explicit mention to surface the download UI)

## 8. Build & Delivery Checklist

```
□ Source audited (no /api, no prisma, no server-only deps)
□ next.config.ts converted to output:"export" + assetPrefix + trailingSlash
□ package.json updated with 15+ Capacitor plugins + jszip
□ capacitor.config.ts written
□ KaTeX/vendor fonts copied to public/ + override CSS written + imported after vendor CSS
□ Native feature hooks written (back-button, storage, haptics, etc.)
□ AndroidManifest.xml customized (permissions + intent filters)
□ variables.gradle: minSdkVersion = 26
□ app/build.gradle: versionCode + versionName + signingConfig
□ Keystore generated (10000-day validity)
□ Icons + splash generated via @capacitor/assets
□ npm run build succeeds
□ Verify: ls out/vendor-fonts/ | wc -l shows ~60 files
□ npx cap sync android succeeds
□ ./gradlew assembleDebug succeeds
□ ./gradlew assembleRelease succeeds
□ aapt2 dump badging shows correct package + version
□ unzip -l app-release.apk | grep vendor-fonts | wc -l shows fonts bundled
□ apksigner verify succeeds
□ Copy both APKs to /home/z/my-project/download/
□ Upload to CloudVault (if available)
□ Write README.md
□ Mention download links in final message to user
```

## 9. Deep-Dive Reference

For the full 58-feature catalog, all 47 code snippets, the 20-error fix table, and copy-pasteable config files, read:

**`/home/z/my-project/download/Capacitor-Native-Feel-Playbook.md`** (80 KB, 902 lines)

That file is the canonical reference. This skill file is the fast-path routing layer — it gets you started and tells you when to consult the playbook for depth.

## 10. Iteration Protocol

After delivering v1:
1. Ask the user to install the release APK and test key flows
2. Document any issues the user reports (screenshot, description, expected vs actual)
3. For each issue: classify (visual bug, data loss, UX, missing feature), find the root cause, apply the fix, rebuild
4. Common v1 issues: KaTeX not rendering (apply B.4), data loss on close (apply C.2), text selection on UI (apply C.13), back button closes app (apply C.1)
5. Bump versionCode (200 → 201 → 202...) for each rebuild so the user can tell which build they're on
6. Keep the same package name + keystore so the user can install over the previous APK without losing data

---

*This skill was distilled from CLIL Engine v2.0 production builds. Every recipe has been tested. Every error has been hit and fixed. Follow the recipes exactly and you will produce a working APK on the first try.*
