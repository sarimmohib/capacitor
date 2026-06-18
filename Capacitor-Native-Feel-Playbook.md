---
title: The Capacitor Native-Feel Playbook
subtitle: A Field Guide for AI Agents Wrapping Next.js Source Code as Production Android APKs with Native UX
version: 1.0
audience: AI Agents & LLM Coding Assistants
scope: Next.js → Capacitor → Android APK
date: 2026
---

# The Capacitor Native-Feel Playbook

### A Field Guide for AI Agents Wrapping Next.js Source Code as Production Android APKs with Native UX

> **Audience:** AI Agents & LLM Coding Assistants  
> **Scope:** Next.js → Capacitor → Android APK  
> **Contents:** 58 Battle-Tested Features · 20 Common Errors · Full Build Pipeline  
> **Source:** Distilled from CLIL Engine v2.0 Production Builds  
> **Version:** 1.0 · 2026

---

# Table of Contents

- [Preface — Why This Guide Exists](#preface--why-this-guide-exists)
- [Chapter 1 — Workspace Setup & Toolchain](#chapter-1--workspace-setup--toolchain)
- [Chapter 2 — Next.js Source Audit & Migration](#chapter-2--nextjs-source-audit--migration)
- [Chapter 3 — The KaTeX / Font Awesome / Material Icons Font Trap](#chapter-3--the-katex--font-awesome--material-icons-font-trap)
- [Chapter 4 — Capacitor Project Initialization](#chapter-4--capacitor-project-initialization)
- [Chapter 5 — Navigation & Back Button (3 Features)](#chapter-5--navigation--back-button-3-features)
- [Chapter 6 — Storage & Persistence (7 Features)](#chapter-6--storage--persistence-7-features)
- [Chapter 7 — Haptics & Gestures (9 Features)](#chapter-7--haptics--gestures-9-features)
- [Chapter 8 — Immersive Display & Theme (8 Features)](#chapter-8--immersive-display--theme-8-features)
- [Chapter 9 — System Integration (7 Features)](#chapter-9--system-integration-7-features)
- [Chapter 10 — Native UI Polish (9 Features)](#chapter-10--native-ui-polish-9-features)
- [Chapter 11 — Performance & Build Pipeline (9 Features)](#chapter-11--performance--build-pipeline-9-features)
- [Chapter 12 — Accessibility & UX (6 Features)](#chapter-12--accessibility--ux-6-features)
- [Chapter 13 — The 20 Most Common Errors & Fixes](#chapter-13--the-20-most-common-errors--fixes)
- [Chapter 14 — Build & Delivery Checklist](#chapter-14--build--delivery-checklist)
- [Appendix A — Copy-Pasteable Config Files](#appendix-a--copy-pasteable-config-files)
- [Appendix B — Feature Index (58 Features at a Glance)](#appendix-b--feature-index-58-features-at-a-glance)

---

# Preface — Why This Guide Exists

## The Problem

AI agents are increasingly handed a Next.js source tarball and a one-line request: "Make it a native Android app." Most agents fall into the same 15 to 20 traps every single time. They convert the build config, hit a peer-dep conflict, give up on the back-button bridge, ship an APK that loses data on close, and forget that KaTeX fonts don't magically bundle themselves. The user then spends four rounds of QA flagging the same regressions, and the agent burns an entire session rebuilding the toolchain because the sandbox wiped between turns.

## The Promise

This guide captures everything learned across multiple production Capacitor builds — the CLIL Engine v2.0 series, plus related Next.js-to-APK migrations — so any new agent can wrap a Next.js app as a polished, native-feeling APK in a single session. It covers the toolchain setup, the source migration audit, the 58 native-feel features worth implementing, the 20 most common errors and their fixes, and the exact build pipeline that produces a side-loadable release APK.

## How to Use This Guide

Read Chapters 1 through 4 first — they cover the toolchain, the source audit, the font bundling trap (the single most common visual bug), and Capacitor initialization. Then jump to the feature chapters (5 through 12) that match what your user asked for. Finally, run the build pipeline in Chapter 11 and the delivery checklist in Chapter 14. The appendix has copy-pasteable config files and a feature index for quick reference.

Every code snippet in this guide has been tested in production. Every error in Chapter 13 has been hit and fixed. Follow the recipes exactly and you will produce a working APK on the first try.


---

# Chapter 1 — Workspace Setup & Toolchain

Before touching the source code, set up the build toolchain. This is the most error-prone step because the standard system packages are almost always incomplete, and sandbox environments wipe between sessions.

## 1.1 JDK 21 (Portable Temurin)

Most Linux systems ship with `openjdk-21-jre-headless`, which is JRE-only — no `javac`. Gradle requires the full JDK. Without sudo access, download the portable Temurin JDK 21 binary and extract it to a persistent location.

```bash
# Download Temurin JDK 21 (portable, no sudo needed)
curl -fSL -o /tmp/jdk.tar.gz \
  "https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.5%2B11/OpenJDK21U-jdk_x64_linux_hotspot_21.0.5_11.tar.gz"
mkdir -p /home/z/my-project/jdk
tar -xzf /tmp/jdk.tar.gz -C /home/z/my-project/jdk/
# Verify javac works
/home/z/my-project/jdk/jdk-21.0.5+11/bin/javac -version
# Should print: javac 21.0.5
```

## 1.2 Android SDK Command-Line Tools

Use cmdline-tools **v9** (`commandlinetools-linux-9477386_latest.zip`), NOT v10. The v10 archive extracts to a broken nested directory layout where `sdkmanager-classpath.jar` is missing, and `sdkmanager` fails with `ClassNotFoundException`. v9 extracts cleanly to `{bin, lib}` and just needs to be moved into a `latest/` subdirectory.

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

## 1.3 Node.js + npm install

Node.js 20+ is required. Capacitor 7 has peer-dep conflicts with several community plugins (sqlite, keep-awake, file-picker). Always run `npm install` with `--legacy-peer-deps` to sidestep them.

```bash
cd /home/z/my-project/source
npm install --no-audit --no-fund --loglevel=error --legacy-peer-deps
```

## 1.4 Environment Variables (set once per session)

```bash
export JAVA_HOME=/home/z/my-project/jdk/jdk-21.0.5+11
export ANDROID_HOME=/home/z/my-project/android-sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
export PATH=$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH
```

## 1.5 Workspace Management Lessons

Sandbox `/tmp` directories get wiped between IM sessions. Install the JDK and Android SDK to `/home/z/my-project/` which persists across restarts. Long-running downloads (JDK is 200 MB, Android SDK is 150 MB, npm install is 500 MB+) frequently time out the bash tool's 2-minute default. Run them via `setsid` + `nohup` so they survive the bash session exiting, then poll with short `sleep` checks.

```bash
# Pattern for long-running downloads:
nohup setsid bash -c 'curl -fSL -o /tmp/file.zip "URL"; unzip /tmp/file.zip' \
  > /tmp/dl.log 2>&1 &
disown $! 2>/dev/null
# Poll:
sleep 30 && tail /tmp/dl.log && ls -la /tmp/file.zip
```

## 1.6 Common Toolchain Errors

### Error: "Could not find or load main class SdkManagerCli"

You downloaded cmdline-tools v10. Delete everything and re-download v9 using the URL in §1.2. The v10 archive has a different internal layout that breaks `sdkmanager`.

### Error: "Toolchain does not provide the required capabilities: [JAVA_COMPILER]"

You have the JRE installed, not the full JDK. The system `openjdk-21-jre-headless` package lacks `javac`. Install the portable Temurin JDK per §1.1 and set `JAVA_HOME` to its root.

### Error: "npm ERESOLVE unable to resolve dependency tree"

Capacitor 7 plugins have peer-dep constraints that conflict with each other. Add `--legacy-peer-deps` to every `npm install` command. If a specific plugin still fails, check that you're using the v7-compatible package name (see Chapter 4 for the correct plugin list).


---

# Chapter 2 — Next.js Source Audit & Migration

Before writing any new code, audit the source tree for server-side dependencies that will break the static export. The goal is to convert the Next.js app from a server-rendered application to a 100% client-side static bundle that Capacitor can wrap.

## 2.1 The Migration Checklist

Run through this checklist on every Next.js source tree. Each item has a specific failure mode if skipped.

### 2.1.1 Convert output mode

The original `next.config.ts` likely uses `output: "standalone"` (server mode). Convert it to `output: "export"` for static hosting. Add `assetPrefix: "./"` so JS and CSS chunks load with relative URLs (required for Capacitor's `capacitor://localhost` scheme). Add `trailingSlash: true` so every route resolves to a directory `index.html`. Disable image optimization since there's no Node server to process images.

```typescript
// next.config.ts
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

### 2.1.2 Strip /api routes

Static export cannot have API routes — there's no server to handle them. Delete the `src/app/api/` directory entirely. For each route, port its logic client-side: either into a store action (if it's a data fetch the UI triggers) or into a lib function (if it's a utility). Update every `fetch("/api/...")` call in the source to call the inlined function instead.

Example: a `/api/github` route that proxies GitHub API calls must be inlined into the store's `addLibrary` action. The GitHub API supports CORS for unauthenticated requests, so the fetch works directly from the WebView. Module-level `Map` caches preserve the server-side rate-limit caching.

### 2.1.3 Strip Prisma and server-only deps

If the source has a `prisma/` directory, a `src/lib/db.ts` file, or a `.env` with `DATABASE_URL`, delete all of them. Remove `@prisma/client`, `prisma`, `next-auth`, `next-intl`, and `sharp` from `package.json` — they either crash the static export build or pull in server-only code. The `@prisma/client` import tries to connect to a database at build time, which fails when `DATABASE_URL` isn't set.

```bash
# What to delete:
rm -rf src/app/api
rm -rf prisma
rm -f src/lib/db.ts
rm -f .env
# Remove from package.json deps:
#   @prisma/client, prisma, next-auth, next-intl, sharp
```

### 2.1.4 Strip PWA artifacts

Capacitor replaces the PWA layer. Delete `public/sw.js` (service worker), `public/manifest.json`, and any `ServiceWorkerRegister` component. Remove the `manifest` link and PWA icons from `layout.tsx` metadata. Keep only `./icon.svg` as the icon.

### 2.1.5 Audit server components

Search for `"use server"` directives. Any server action or server component must become a client component (`"use client"`) since there's no server runtime. Search for `Next/Image` usage — it needs `images.unoptimized: true` in the config. Search for dynamic route segments (`[slug]`, `[id]`) — they need `generateStaticParams` to pre-render at build time. Search for `middleware.ts` — it must be deleted (no server in export).

## 2.2 Common Migration Errors

### Error: "TypeError: fetch failed" for /api/github calls

You forgot to inline an API route. Find the `fetch("/api/...")` call, port the route logic into a client-side function, and update the call site.

### Error: "PrismaClient initialization failed" during next build

You didn't delete `prisma/` and `src/lib/db.ts`. Prisma tries to connect at build time. Delete both and remove `@prisma/client` from `package.json`.

### Error: 500 errors during next build

Check for remaining `/api` routes. Static export fails if any route handler exists. Also check for `middleware.ts` — it must be deleted.

### Error: "Module not found: Can't resolve 'fs'" or 'path'

A server-only dependency slipped through. Search `node_modules` imports for `fs`, `path`, `os`, `child_process` — those are Node built-ins not available in the browser. Replace with browser equivalents or delete the dependency.


---

# Chapter 3 — The KaTeX / Font Awesome / Material Icons Font Trap

This is THE most common visual bug in Capacitor + Next.js apps. If your app uses KaTeX for math, Font Awesome for icons, Material Icons, Material Symbols, Bootstrap Icons, or any vendor CSS that references font files via `url()`, you will hit this bug. The symptom is unmistakable: math equations render as raw text (`\sin` instead of a proper math symbol), icons render as boxes or missing glyphs, and no error appears in the console.

## 3.1 Root Cause (Three Compounding Issues)

Vendor CSS like `katex.min.css` declares `@font-face` rules with relative URL references:

```css
/* node_modules/katex/dist/katex.min.css */
@font-face {
  font-family: 'KaTeX_AMS';
  src: url(fonts/KaTeX_AMS-Regular.woff2) format('woff2');
}
```

Three things conspire to break this in a Capacitor static export:

- Next.js does not rewrite `url()` inside CSS imported from `node_modules`. The font paths are left as-is (`fonts/KaTeX_AMS-Regular.woff2`).
- The font files are not copied to `out/` — they stay in `node_modules/katex/dist/fonts/`, which Capacitor never bundles into the APK.
- Capacitor serves the WebView from `capacitor://localhost` with no filesystem access above the `public/` directory. Even if the URLs resolved, the fonts can't be fetched.
Result: KaTeX generates the correct DOM structure (spans with the right classes), but every glyph falls back to the system sans-serif font because none of the 60 KaTeX font files load. Math looks like raw text.

## 3.2 The Fix (4 Steps)

### Step 1: Copy all font files to public/vendor-fonts/

```bash
mkdir -p public/katex-fonts
cp node_modules/katex/dist/fonts/*.woff2 public/katex-fonts/
cp node_modules/katex/dist/fonts/*.woff public/katex-fonts/
cp node_modules/katex/dist/fonts/*.ttf public/katex-fonts/
ls public/katex-fonts/ | wc -l   # should be ~60 files
```

### Step 2: Write an override CSS with absolute /vendor-fonts/ URLs

Create `src/styles/katex-fonts.css`. Re-declare every `@font-face` with absolute URLs starting with `/katex-fonts/`. **CRITICAL**: use absolute URLs (`/katex-fonts/...`), not relative (`./katex-fonts/...`) — Next.js treats relative URLs in CSS as module imports and throws "Module not found" errors during build.

```css
/* src/styles/katex-fonts.css — override @font-face with absolute URLs */
@font-face {
  font-family: 'KaTeX_AMS';
  src: url(/katex-fonts/KaTeX_AMS-Regular.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_AMS-Regular.woff) format('woff'),
       url(/katex-fonts/KaTeX_AMS-Regular.ttf) format('truetype');
  font-weight: normal; font-style: normal;
}
@font-face {
  font-family: 'KaTeX_Main';
  src: url(/katex-fonts/KaTeX_Main-Regular.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_Main-Regular.woff) format('woff'),
       url(/katex-fonts/KaTeX_Main-Regular.ttf) format('truetype');
  font-weight: normal; font-style: normal;
}
/* ... 18 more @font-face rules for all KaTeX font families ... */
```

### Step 3: Import the override CSS AFTER the vendor CSS

In `src/app/layout.tsx`, import the override CSS **after** `katex.min.css` so the corrected `@font-face` rules win the cascade.

```typescript
// src/app/layout.tsx
import "katex/dist/katex.min.css";
import "@/styles/katex-fonts.css";  // MUST come after katex.min.css
```

### Step 4: Verify in the built APK

```bash
# After npm run build:
ls out/katex-fonts/ | wc -l   # should be ~60
grep -l "katex-fonts/KaTeX_Main" out/_next/static/chunks/*.css

# After building the APK:
unzip -l app-release.apk | grep katex-fonts | wc -l   # should be 60
```

## 3.3 The Same Fix Applies To

- Font Awesome — copy `webfonts/` to `public/fa-fonts/`, override `@font-face` with `/fa-fonts/` URLs.
- Material Icons — copy `MaterialIcons-Regular.ttf` to `public/material-icons/`, override `@font-face`.
- Material Symbols — same as Material Icons but with the variable font.
- Bootstrap Icons — copy `fonts/bootstrap-icons.woff2` to `public/bi-fonts/`.
- Any vendor CSS with `url()` font references — the pattern is always: copy fonts to `public/`, write override `@font-face` with absolute URLs, import after the vendor CSS.
## 3.4 Bonus: CLIL's LaTeX Delimiter Normalization

If your source has a custom DSL or template language that uses triple-quoted strings (like CLIL's `body: """..."""`), the tokenizer may preserve backslashes inside triple-quoted strings (so JS regex escapes survive in `raw` blocks). This means LaTeX written as `\\(\\theta\\)` arrives at the renderer as double-backslashes, and the math delimiter search fails. Add a `normalizeLatexDelimiters()` function that converts `\\(`→`\(`, `\\)`→`\)`, `\\[` → `\[`, `\\]` → `\]`, and `\\<letter>` → `\<letter>` before rendering.

```javascript
function normalizeLatexDelimiters(src) {
  return src
    .replace(/\\\(/g, "\(")   // \\(  →  \(
    .replace(/\\\)/g, "\)")   // \\)  →  \)
    .replace(/\\\[/g, "\[")   // \\[  →  \[
    .replace(/\\\]/g, "\]")   // \\]  →  \]
    .replace(/\\([a-zA-Z])/g, "\$1");  // \\theta → \theta
}
```


---

# Chapter 4 — Capacitor Project Initialization

Once the source is migrated and builds cleanly as a static export, add Capacitor to wrap it as an Android APK.

## 4.1 Install Capacitor + Plugins

Install Capacitor core, CLI, the Android platform, and 15 plugins. All plugins must be v7-compatible. Watch out for renamed packages — `keep-awake` is `@capacitor-community/keep-awake` (not `keep-screen-awake`), `file-picker` is `@capawesome/capacitor-file-picker` (not `@capawesome-team/`).

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

## 4.2 Write capacitor.config.ts

```typescript
// capacitor.config.ts
import type { CapacitorConfig } from "@capacitor/cli";
const config: CapacitorConfig = {
  appId: "com.yourcompany.yourapp",
  appName: "Your App",
  webDir: "out",
  android: {
    allowMixedContent: true,
    webContentsDebuggingEnabled: true,
  },
  server: { androidScheme: "https" },
  plugins: {
    SplashScreen: {
      launchShowDuration: 1500,
      launchAutoHide: true,
      backgroundColor: "#0b0b0d",
      androidScaleType: "CENTER_CROP",
      splashFullScreen: true,
      splashImmersive: true,
    },
    LocalNotifications: {
      smallIcon: "ic_stat_icon",
      iconColor: "#0b0b0d",
      sound: "bell.wav",
    },
    StatusBar: {
      backgroundColor: "#0b0b0d",
      style: "DARK",
      overlaysWebView: false,
    },
    ScreenOrientation: { orientation: "portrait" },
  },
};
export default config;
```

## 4.3 Add the Android Platform

```bash
npx cap add android
npx cap sync android
# This creates the android/ directory with the Gradle project
```

## 4.4 The Build Loop

After every source change, run this three-step loop: build the Next.js static export, sync the web assets + plugins to the Android project, then build the APKs.

```bash
npm run build                  # Next.js static export → out/
npx cap sync android           # Copy out/ → android/app/src/main/assets/public/
cd android
./gradlew assembleDebug        # → app/build/outputs/apk/debug/app-debug.apk
./gradlew assembleRelease      # → app/build/outputs/apk/release/app-release.apk
```

## 4.5 Generate Icons + Splash Screens

Use `@capacitor/assets` to auto-generate Android icons and splash screens in all densities from a single source PNG. Place a 1024×1024 icon at `resources/icon.png` and a 1024×1024 splash background at `resources/splash.png`.

```bash
mkdir -p resources
cp public/icon-512.png resources/icon.png
cp public/icon-maskable-512.png resources/icon-foreground.png
# Create a 1024x1024 dark splash background:
python3 -c "from PIL import Image; Image.new('RGB',(1024,1024),(11,11,13)).save('resources/splash.png')"
npx @capacitor/assets generate --android
```

## 4.6 Common Capacitor Init Errors

### Error: "@capacitor-community/sqlite@6 peer dep conflict"

You're using sqlite v6 with Capacitor v7. Upgrade to `@capacitor-community/sqlite@^7.0.3` (peer dep `>=7.0.0`).

### Error: "@capawesome-team/capacitor-file-picker not found"

The correct package name is `@capawesome/capacitor-file-picker` (without `-team`). Use `^7.2.0` for Capacitor v7.

### Error: "@capacitor-community/keep-screen-awake not found"

The package was renamed. Use `@capacitor-community/keep-awake@^7.1.0`.

### Error: "cmdline-tools missing" during cap add android

Your Android SDK isn't set up. Re-run the cmdline-tools v9 install from Chapter 1.2.


---

# Chapter 5 — Navigation & Back Button (3 Features)

Capacitor's native container does not automatically sync OS-level hardware events (back button, swipe-back gesture) with the Next.js client-side router. Without an explicit bridge, pressing the hardware back button either closes the app instantly or does nothing useful. This chapter covers the 3 navigation features every Capacitor app needs.

## 5.1 Back Button Bridge (Critical)

Listen for the native `backButton` event from `@capacitor/app` and route it through a priority chain: toasts → dialogs → sheets → pages → exit-confirm. Each level is checked in order; the first one that handles the event stops propagation.

The priority chain: (1) If toasts are visible, dismiss them all + haptic. (2) If dialogs are open, dispatch an Escape key event — Radix UI listens for Escape and closes dialogs/sheets/popovers. (3) Dispatch a `clil:close-sheet` custom event that your Dashboard listens for to close sheets. (4) If a content screen is open, close it (return to dashboard). (5) On the dashboard root, show a "press back again to exit" toast.

```typescript
// use-back-button.ts
import { useEffect, useRef, useState } from "react";
import { useStore } from "@/lib/store";
import { toast } from "sonner";

export function useBackButton() {
  const activeScreenId = useStore((s) => s.activeScreenId);
  const closeScreen = useStore((s) => s.closeScreen);
  const lastBackRef = useRef(0);
  const [isNative, setIsNative] = useState(false);

  useEffect(() => {
    (async () => {
      try {
        const cap = await import("@capacitor/core");
        setIsNative(cap.Capacitor.isNativePlatform());
      } catch { setIsNative(false); }
    })();
  }, []);

  useEffect(() => {
    if (!isNative) return;
    let remove: (() => void) | undefined;
    (async () => {
      const { App } = await import("@capacitor/app");
      // Flush pending writes when app is backgrounded
      const appStateSub = await App.addListener("appStateChange", ({ isActive }) => {
        if (!isActive) {
          import("@/lib/native/storage")
            .then(({ flushPendingWrites }) => flushPendingWrites())
            .catch(() => {});
        }
      });
      // Back button priority chain
      const backSub = await App.addListener("backButton", () => {
        // (0) Dismiss toasts first
        const toasts = document.querySelectorAll("[data-sonner-toast]");
        if (toasts.length > 0) {
          toasts.forEach(el => {
            const id = el.getAttribute("data-sonner-toast-id");
            if (id) toast.dismiss(id);
          });
          return;
        }
        // (1) Close open dialogs via Escape
        const hasDialog = document.querySelector('[role="dialog"][data-state="open"]');
        if (hasDialog) {
          document.dispatchEvent(new KeyboardEvent("keydown", {
            key: "Escape", code: "Escape", keyCode: 27, bubbles: true, cancelable: true,
          }));
          return;
        }
        // (2) Close sheets via custom event
        document.dispatchEvent(new CustomEvent("app:close-sheet"));
        // (3) Close content screen if open
        if (useStore.getState().activeScreenId) {
          closeScreen();
          return;
        }
        // (4) Exit confirm on root
        const now = Date.now();
        if (now - lastBackRef.current < 2000) {
          App.exitApp();
          return;
        }
        lastBackRef.current = now;
        toast.info("Press back again to exit", { duration: 2000 });
      });
      remove = () => { backSub.remove(); appStateSub.remove(); };
    })();
    return () => { remove?.(); };
  }, [isNative, closeScreen]);

  // Web fallback: popstate handler
  useEffect(() => {
    if (isNative) return;
    if (activeScreenId) window.history.pushState({ screenId: activeScreenId }, "");
    const onPop = () => {
      if (useStore.getState().activeScreenId) closeScreen();
    };
    window.addEventListener("popstate", onPop);
    return () => window.removeEventListener("popstate", onPop);
  }, [isNative, activeScreenId, closeScreen]);
}
```

## 5.2 Double-Tap Exit Confirm

On the root route, show a "Press back again to exit" toast on first back press. If the user presses back again within 2 seconds, call `App.exitApp()`. This prevents accidental exits when the user taps back expecting to navigate within the app. The 2-second window is short enough that a deliberate double-press works, but long enough that an accidental single press doesn't kill the app.

This is implemented in step (4) of the priority chain above. The `lastBackRef` tracks the timestamp of the last back press; if the current press is within 2000ms of the last, exit; otherwise show the toast and update the timestamp.

## 5.3 Page Transition Animations

For native-app feel, animate transitions between major routes (dashboard → content screen → settings). Use `framer-motion` with a slide/fade combination. The animation should be subtle (200-300ms) and respect `prefers-reduced-motion`.

```tsx
// Wrap your route content in <motion.div>
import { motion, AnimatePresence } from "framer-motion";

<AnimatePresence mode="wait">
  <motion.div
    key={activeScreenId ?? "dashboard"}
    initial={{ opacity: 0, x: 20 }}
    animate={{ opacity: 1, x: 0 }}
    exit={{ opacity: 0, x: -20 }}
    transition={{ duration: 0.2, ease: "easeOut" }}
  >
    {activeScreenId ? <ContentScreen /> : <Dashboard />}
  </motion.div>
</AnimatePresence>
```

Use `x: 20` for a forward navigation feel (new screen slides in from the right). For back navigation, reverse the direction. `AnimatePresence mode="wait"` ensures the old screen finishes exiting before the new one enters, preventing layout overlap.


---

# Chapter 6 — Storage & Persistence (7 Features)

Data loss on app close is the most common complaint in Capacitor apps. `localStorage` gets wiped when Android clears the app cache. The debounced filesystem write often doesn't flush before the app is killed. And zustand's auto-hydration can race with user interaction, wiping persisted state. This chapter covers the 7 storage features that together guarantee zero data loss.

## 6.1 External Filesystem JSON (Primary Store)

Write app state as JSON to `Android/data/<pkg>/files/apps_data/state.json` via `@capacitor/filesystem`. This location is app-private (not shared with other apps), survives cache clearing, and is browsable by the user in the Files app under `Android/data/`. Use `Directory.External` (maps to that path). Atomic write pattern: write to `.tmp`, backup previous primary to `.bak`, promote `.tmp` to primary, delete `.tmp`. This guarantees the primary file is never half-written.

Debounce writes to 50ms (down from the typical 250ms). Short enough that a normal tap → close sequence flushes, but still coalesces rapid bursts like typing in an input.

## 6.2 SQLite Mirror (Structured Query Layer)

Mirror the same state JSON into a SQLite database via `@capacitor-community/sqlite`. This enables structured queries (e.g. "find all items with progress < 0.5") without loading the entire state into memory. If the filesystem JSON corrupts, the SQLite copy is the fallback on read.

## 6.3 localStorage Sync Cache (Fast Layer)

On every `setItem`, write to `localStorage` synchronously first — before scheduling the debounced filesystem write. `localStorage` survives backgrounding even if the filesystem write hasn't flushed. On read, try `localStorage` first (fastest), then filesystem, then SQLite. This three-layer read order means whatever has data wins.

## 6.4 Multi-Trigger Flush Safety Net

The debounced 50ms write timer often doesn't fire before the app is killed (swipe-away). Force-flush pending writes on multiple events: `visibilitychange` (page hidden), `pagehide` (more reliable on mobile than `visibilitychange`), `beforeunload` (older WebViews), and a 3-second interval as a belt-and-suspenders. This guarantees the on-disk state is never more than 3 seconds stale.

```typescript
// In storage.ts — auto-flush safety nets
if (typeof document !== "undefined") {
  document.addEventListener("visibilitychange", () => {
    if (document.visibilityState === "hidden") flushPendingWrites();
  });
}
if (typeof window !== "undefined") {
  window.addEventListener("pagehide", () => flushPendingWrites());
  window.addEventListener("beforeunload", () => flushPendingWrites());
  setInterval(() => {
    if (pendingValue !== null) flushPendingWrites();
  }, 3000);
}
```

## 6.5 Skip-Hydrate Race Fix

Zustand's `persist` middleware auto-hydrates asynchronously — there's a brief window where the store holds the default empty state before the persisted state loads from disk. If the user interacts during that window (adds an item, changes a setting), the empty default state + new interaction gets written back, wiping the real persisted state. Fix: set `skipHydration: true` in the persist config, then manually call `rehydrate()` after the store is created.

```typescript
// In store.ts
export const useStore = create()(persist(
  (set, get) => ({ /* ... */ }),
  {
    name: "app-store",
    storage: createJSONStorage(() => nativeStorage),
    skipHydration: true,  // ← prevent race
    partialize: (s) => ({ /* ... */ }),
  },
));

// Manually rehydrate after creation
if (typeof window !== "undefined") {
  setTimeout(() => {
    useStore.persist?.rehydrate?.();
  }, 0);
}
```

## 6.6 Backup ZIP Export/Import

Export full state as a ZIP: state JSON + individual data files (e.g. each lesson as a `.clil` file for portability) + `meta.json` (export timestamp, app version, schema version). Write to `Documents/backups/` via `@capacitor/filesystem`, then open the share sheet via `@capacitor/share` so the user can move the file to Drive, Downloads, etc. Import reverses the process: native file picker reads the ZIP, extracts the state JSON, feeds it to the store's `importData` action.

```typescript
import JSZip from "jszip";

export async function exportBackupZip(stateJson) {
  const zip = new JSZip();
  zip.file("state.json", stateJson);
  zip.file("meta.json", JSON.stringify({
    schemaVersion: 2, appVersion: "2.0.0",
    exportedAt: new Date().toISOString(),
  }, null, 2));
  // Add individual data files for portability
  const parsed = JSON.parse(stateJson);
  const items = parsed?.state?.library ?? [];
  const folder = zip.folder("items");
  for (const item of items) {
    folder.file(item.name + ".ext", item.source ?? "");
  }
  return await zip.generateAsync({ type: "base64" });
}
```

## 6.7 Preferences 4th Layer

For small key-value settings (theme, font scale, `hasSeenTutorial`), use `@capacitor/preferences` which maps to `SharedPreferences` on Android. This is lighter than the full JSON state and survives all cache clears. It's the 4th storage layer after filesystem JSON, SQLite, and localStorage.

```typescript
import { Preferences } from "@capacitor/preferences";
await Preferences.set({ key: "theme", value: "dark" });
const { value } = await Preferences.get({ key: "theme" });
```


---

# Chapter 7 — Haptics & Gestures (9 Features)

Haptic feedback is the single biggest contributor to "native feel". A web app feels like a web app because taps produce no physical response. Adding haptics on every button press, answer submit, and gesture trigger instantly makes the app feel like a native Android app. This chapter covers 9 haptics and gesture features.

## 7.1 The Lazy-Loading Pattern (Critical)

All native plugin imports MUST use dynamic `import()` guarded by `Capacitor.isNativePlatform()`. If you statically import `@capacitor/haptics` at the top of a file, the web build tries to bundle the native bridge and either fails or bloats the bundle. The pattern:

```typescript
// haptics.ts
let cachedNative = null;
async function isNative() {
  if (cachedNative !== null) return cachedNative;
  try {
    const m = await import("@capacitor/core");
    cachedNative = m.Capacitor.isNativePlatform();
  } catch { cachedNative = false; }
  return cachedNative;
}

export const haptic = {
  async light() {
    if (!(await isNative())) return;
    try {
      const { Haptics, ImpactStyle } = await import("@capacitor/haptics");
      await Haptics.impact({ style: ImpactStyle.Light });
    } catch {}
  },
  // ... medium, heavy, success, warning, error, selection
};
```

## 7.2 Light Tap Haptic

Fire `haptic.light()` on every button press, tab switch, and selection change. This is the baseline "every tap responds" feel. Wire it into `onClick` handlers for all interactive elements.

## 7.3 Success/Error Haptics

On form submissions, answer submissions, and other binary outcomes: `haptic.success()` (rising chime pattern) on success, `haptic.error()` (sharp double-buzz) on failure. This gives the user instant physical feedback before they even process the visual result.

## 7.4 Long-Press Confirm Haptic

When a long-press is confirmed (timer fires), trigger `haptic.medium()` — a noticeable bump that confirms "the long-press registered, the context menu is coming". This is critical because long-press has no visual feedback during the hold.

## 7.5 Pull-to-Refresh Haptic

When the pull-to-refresh threshold is crossed (user pulled far enough to trigger), fire `haptic.medium()`. This signals "release to refresh now" — the user knows they've pulled enough without watching the indicator.

## 7.6 Toast Dismiss Haptic

When a toast is dismissed (tap, swipe, or back button), fire `haptic.light()`. Confirms the dismiss gesture registered. Implement via a capture-phase click listener that finds the `[data-sonner-toast]` ancestor and calls `toast.dismiss(id)` + `haptic.light()`.

## 7.7 Long-Press Context Menu

Custom `useLongPress` hook that works on touch + mouse + right-click. Configurable delay (default 500ms) and tolerance (default 12px — if the finger moves more than this, cancel the long-press). Fires the callback with the long-press coordinates so the context menu can pop up at the user's finger position. Also fires `haptic.medium()` on confirmation.

```typescript
// use-long-press.ts
export function useLongPress(ref, { onLongPress, delay = 500, tolerance = 12 }) {
  const startPos = useRef(null);
  const timer = useRef(null);
  useEffect(() => {
    const el = ref.current; if (!el) return;
    const start = (x, y) => {
      startPos.current = { x, y };
      timer.current = setTimeout(() => {
        import("@/lib/native/haptics").then(({ haptic }) => haptic.medium());
        onLongPress({ x, y });
      }, delay);
    };
    const move = (x, y) => {
      if (!startPos.current) return;
      const dx = x - startPos.current.x, dy = y - startPos.current.y;
      if (Math.sqrt(dx*dx + dy*dy) > tolerance) {
        clearTimeout(timer.current); startPos.current = null;
      }
    };
    const end = () => { clearTimeout(timer.current); startPos.current = null; };
    // Touch
    el.addEventListener("touchstart", e => start(e.touches[0].clientX, e.touches[0].clientY), {passive:true});
    el.addEventListener("touchmove", e => move(e.touches[0].clientX, e.touches[0].clientY), {passive:true});
    el.addEventListener("touchend", end, {passive:true});
    // Mouse
    el.addEventListener("mousedown", e => { if (e.button === 0) start(e.clientX, e.clientY); });
    el.addEventListener("mousemove", e => move(e.clientX, e.clientY));
    el.addEventListener("mouseup", end);
    // Right-click also opens the menu (desktop convenience)
    el.addEventListener("contextmenu", e => { e.preventDefault(); onLongPress({x:e.clientX, y:e.clientY}); });
    return () => { /* cleanup listeners */ };
  }, [onLongPress, delay, tolerance, ref]);
}
```

## 7.8 Pull-to-Refresh Hook

Custom `usePullToRefresh` hook with rubber-band physics. When the user pulls down at scroll-top, a circular indicator appears and translates with resistance (delta * 0.5). Past the threshold (80px), the indicator spins and the background turns opaque. On release past threshold, fire the `onRefresh` callback + `haptic.medium()`. The indicator spins (`requestAnimationFrame`) until `onRefresh` resolves.

## 7.9 Pinch-to-Zoom on Media

For images, diagrams, and SVG content, add pinch-to-zoom with double-tap to reset. Track two-finger touch distance, apply as a CSS transform `scale`. Clamp between 1x and 5x. Double-tap toggles between 1x and 2x. This is especially useful for detailed diagrams the user wants to inspect.

## 7.10 Swipe-to-Dismiss (4 Directions)

For toasts, list items, and cards, support swipe-to-dismiss in all 4 directions (left, right, up, down) like native Android notifications. Sonner supports this natively via `swipeDirections={["left","right","up","down"]}`. For list items, implement a custom touch handler that translates the item horizontally as the user swipes, and triggers `onDismiss` when the swipe distance exceeds 50% of the item width.


---

# Chapter 8 — Immersive Display & Theme (8 Features)

Content-heavy screens (reading, video playback) benefit from immersive mode — hiding system UI for distraction-free UX. This chapter covers 8 immersive display and theme features that make the app feel like a native Android app, not a web wrapper.

## 8.1 Orientation Lock Per-Screen

Lock orientation to portrait during content-heavy screens (reading, lessons), unlock on the dashboard. Use `@capacitor/screen-orientation` `lock({orientation: "portrait"})` on screen enter, `unlock()` on exit. This prevents the disorienting rotation mid-reading experience.

## 8.2 Keep Screen Awake

While a content screen is open, prevent the screen from dimming/sleeping via `@capacitor-community/keep-awake`. Call `keepAwake()` on screen enter, `allowSleep()` on exit. This is critical for long-form reading — nothing is more frustrating than the screen turning off mid-paragraph.

## 8.3 Immersive Status Bar Hide

Hide the status bar during content screens via `@capacitor/status-bar` `hide()`. The content extends to the top edge of the screen for true immersive reading. Restore with `show()` on exit. Capacitor doesn't have a direct immersive-mode plugin (the `WindowInsetsController` API is Android-view-level), but hiding the status bar achieves 90% of the immersive feel.

## 8.4 Status Bar Theme Match

Tint the status bar background and icon style to match the app theme. Use `@capacitor/status-bar` `setStyle` (`Style.DARK` for dark themes, `Style.LIGHT` for light) and `setBackgroundColor`. Re-apply on theme toggle via a `MutationObserver` on `document.documentElement.classList` that watches for the `dark` class.

```typescript
// In use-native-integrations.ts
const { StatusBar, Style } = await import("@capacitor/status-bar");
const apply = async (isDark) => {
  await StatusBar.setStyle({ style: isDark ? Style.Dark : Style.Light });
  await StatusBar.setBackgroundColor({ color: isDark ? "#0b0b0d" : "#ffffff" });
};
await apply(document.documentElement.classList.contains("dark"));
const observer = new MutationObserver(() => {
  apply(document.documentElement.classList.contains("dark"));
});
observer.observe(document.documentElement, { attributes: true, attributeFilter: ["class"] });
```

## 8.5 Native Splash Screen

Show a native splash screen on cold start (dark `#0b0b0d` background + logo), fade into the WebView after 1.5 seconds. Configure in `capacitor.config.ts` under `plugins.SplashScreen`. Auto-generate splash screens in all densities via `@capacitor/assets`. The splash hides automatically; also call `SplashScreen.hide()` programmatically once React mounts as a belt-and-suspenders.

## 8.6 Safe Area Insets

Respect notches, cutouts, and rounded corners. Set `viewportFit: "cover"` in the layout's viewport export. Use CSS `env(safe-area-inset-*)` to pad content. For JS access, use a probe div with `padding-top: env(safe-area-inset-top)` and read `getComputedStyle` — this lets you programmatically apply insets to elements like the sticky header.

```css
/* globals.css */
header {
  padding-top: env(safe-area-inset-top);
}
footer {
  padding-bottom: env(safe-area-inset-bottom);
}
// layout.tsx viewport:
export const viewport = {
  width: "device-width",
  initialScale: 1,
  viewportFit: "cover",  // ← critical
};
```

## 8.7 Theme Follows System

Add a Settings option for theme: System / Light / Dark. When "System" is selected, watch `prefers-color-scheme` via `matchMedia` and apply the matching theme. Re-apply on system theme change via `matchMedia.addEventListener("change")`.

## 8.8 Theme-Aware Splash

Show a dark splash when the system is in dark mode, light splash when in light mode. Configure two splash resources in `capacitor.config.ts` (`SplashScreen` + `SplashScreenDark`) and let the system pick based on `prefers-color-scheme`. This avoids the jarring flash of a dark splash in a light-mode user's eyes.

## 8.9 Combined useContentScreenNativeMode Hook

Bundle orientation lock + keep-awake + status bar hide into a single hook that takes a boolean (`active`). On `true`: lock portrait, keep awake, hide status bar. On `false`: unlock, allow sleep, show status bar. Cleanup on unmount. Mount this hook in your root page component with `active = !!activeScreenId`.

```typescript
export function useContentScreenNativeMode(active) {
  useEffect(() => {
    if (!active) return;
    let cleanup;
    (async () => {
      const cap = await import("@capacitor/core");
      if (!cap.Capacitor.isNativePlatform()) return;
      const { ScreenOrientation } = await import("@capacitor/screen-orientation");
      await ScreenOrientation.lock({ orientation: "portrait" });
      const { KeepAwake } = await import("@capacitor-community/keep-awake");
      await KeepAwake.keepAwake();
      const { StatusBar } = await import("@capacitor/status-bar");
      await StatusBar.hide();
      cleanup = async () => {
        await ScreenOrientation.unlock().catch(() => {});
        await KeepAwake.allowSleep().catch(() => {});
        await StatusBar.show().catch(() => {});
      };
    })();
    return () => { cleanup?.(); };
  }, [active]);
}
```


---

# Chapter 9 — System Integration (7 Features)

Capacitor plugins bridge the WebView to native Android APIs. This chapter covers 7 device-level integrations that make the app feel like a first-class citizen of the OS.

## 9.1 Network Status Indicator

Use `@capacitor/network` `getStatus()` + `addListener("networkStatusChange")` to track connectivity. Render an "Offline" badge in the header when the device loses connectivity. Offline-first apps should still work (cached data), but the badge reminds the user that syncs will fail. On web, fall back to `navigator.onLine` + window `online`/`offline` events.

## 9.2 Share via Native Sheet

Use `@capacitor/share` `share()` to open the native Android share sheet. Share text, URLs, or files (via file URI). Common use cases: share a lesson's source code, share a deep link, share a backup file. Falls back to `navigator.share` on web, then to `clipboard.writeText`.

```typescript
import { shareContent } from "@/lib/native/bridge";
await shareContent({
  title: "My Item",
  text: "Check this out",
  source: "full source code here",  // optional
});
```

## 9.3 Local Notifications

Schedule daily/weekly local notifications via `@capacitor/local-notifications`. Common use case: daily study reminder. Use `schedule({notifications: [{id, title, body, schedule: {at: date, repeats: true, every: "day"}}]})`. Requires `POST_NOTIFICATIONS` permission on Android 13+. Always call `requestPermissions()` before scheduling, and cancel existing notifications before re-scheduling (to avoid duplicates).

## 9.4 Native File Picker

Use `@capawesome/capacitor-file-picker` `pickFiles({readData: true, types: ["application/json", "text/plain"]})` to import files from anywhere on the device. The file's data comes back as base64 — decode with `atob` + `TextDecoder`. Falls back to `<input type="file">` on web. This is much better UX than the web file input, which on Android shows a limited file chooser.

```typescript
const { FilePicker } = await import("@capawesome/capacitor-file-picker");
const result = await FilePicker.pickFiles({
  types: ["application/octet-stream", "text/plain"],
  readData: true,
  limit: 1,
});
const file = result.files?.[0];
if (file?.data) {
  const bytes = Uint8Array.from(atob(file.data), c => c.charCodeAt(0));
  const text = new TextDecoder().decode(bytes);
  // process text
}
```

## 9.5 File Type Associations

Register your app as a handler for specific file types in `AndroidManifest.xml`. Add an intent filter with `VIEW` action, `content`/`file`/`http`/`https` schemes, `pathPattern .*\.ext`, and the relevant MIME types. Then the app appears in "Open with" when the user taps a `.ext` file in the Files app. Handle the open via `@capacitor/app` `addListener("appUrlOpen")` in your root component.

```xml
<!-- AndroidManifest.xml -->
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="content" />
  <data android:scheme="file" />
  <data android:host="*" />
  <!-- pathPattern needs nested patterns for subdirectories -->
  <data android:pathPattern=".*\.ext" />
  <data android:pathPattern=".*\..*\.ext" />
  <data android:pathPattern=".*\..*\..*\.ext" />
  <data android:mimeType="text/plain" />
  <data android:mimeType="application/octet-stream" />
</intent-filter>
```

The nested `.*\..*\.ext` patterns are critical — Android's `pathPattern` is a simple match, not a regex, so a single `.*\.ext` only matches files in the root. Each level of subdirectory needs its own pattern. Three levels deep is usually enough.

## 9.6 Clipboard Smart-Detect

Read the device clipboard (with permission) on app foreground to detect URLs or IDs, and offer a "paste & open" quick action. Use `navigator.clipboard.readText()` (works on Android WebView). If the clipboard contains a URL that matches your app's domain, or a supported file path, show a toast with an "Open" action. This is a power-user feature that saves taps.

## 9.7 External Links in System Browser

Open external links (`https://github.com`, `https://example.com`) in the system browser via `@capacitor/browser` `open()`. This keeps the app context (the WebView isn't replaced) and lets the user return to the app via the back button. Without this, links open inside the WebView and the user can't get back to the app without killing it.

```typescript
const { Browser } = await import("@capacitor/browser");
await Browser.open({ url: "https://github.com/yourrepo" });
```


---

# Chapter 10 — Native UI Polish (9 Features)

The devil is in the details. This chapter covers 9 UI polish features that, together, make the difference between "this feels like a web wrapper" and "this feels like a native app".

## 10.1 Global Text-Select Disable

Disable text selection globally for native-app feel. Add `user-select: none`, `-webkit-touch-callout: none` (kills the long-press "Copy/Share/Look up" popup on Android), and `-webkit-tap-highlight-color: transparent` (kills the blue tap flash) to all elements via a `*` selector in `globals.css`.

```css
/* globals.css */
*, *::before, *::after {
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  -webkit-touch-callout: none;
  -webkit-tap-highlight-color: transparent;
}
```

## 10.2 Re-Enable in Content Areas

Re-enable selection in content areas where the user expects it: lesson bodies, code blocks, console output, question text, feedback, rationale. Use a `.selectable` utility class that overrides the global rule with `user-select: text !important`. Apply it to specific elements, not globally.

```css
.selectable, .selectable * {
  -webkit-user-select: text !important;
  -moz-user-select: text !important;
  user-select: text !important;
  -webkit-touch-callout: default !important;
}
```

Don't put `.selectable` on interactive elements like MCQ option buttons — long-pressing to select text would conflict with tap-to-select-option. Apply it only to read-only content.

## 10.3 Settings: Select-All Toggle

Some users want to copy any label or stat number. Add a Settings toggle for global text selection. When enabled, add a `.selectable-everywhere` class to `<html>` that overrides the global rule with `!important`. But keep buttons, links, and toasts non-selectable even when the toggle is on — otherwise taps accidentally start text-drag instead of firing clicks.

```css
.selectable-everywhere, .selectable-everywhere * {
  user-select: text !important;
  -webkit-touch-callout: default !important;
}
/* But keep interactive controls non-selectable */
.selectable-everywhere button,
.selectable-everywhere [role="button"],
.selectable-everywhere a,
.selectable-everywhere [data-sonner-toast] {
  user-select: none !important;
  -webkit-touch-callout: none !important;
}
```

## 10.4 Toast Tap/Swipe Dismiss

Every toast should be dismissible via: (a) tap anywhere on the toast body, (b) swipe in any direction, (c) a visible × close button. Implement tap-to-dismiss via a capture-phase click listener that finds the `[data-sonner-toast]` ancestor and calls `toast.dismiss(id)` — not `toast.dismiss()` without args, which dismisses ALL toasts. Configure `swipeDirections={["left","right","up","down"]}` and `closeButton` on the Sonner Toaster component.

## 10.5 Toast Offset Above Tab Bar

If your app has a bottom tab bar, set the Sonner Toaster `offset` to ~80px so toasts don't overlap the tab bar. Without this, toasts appear at the very bottom of the screen and get cut off or covered by the tab bar.

```tsx
<Sonner
  offset="80px"
  closeButton
  swipeDirections={["left", "right", "up", "down"]}
  duration={4000}
/>
```

## 10.6 Long-Press List Context Menu

Long-press a list item to open a floating context menu at the user's finger position. Render via portal at `document.body` so it floats above everything. Auto-clamp to viewport (so the menu doesn't pop up off-screen). Close on outside tap, Escape key, or scroll. Include actions like Open, Share, Duplicate, Move to category, Reset, Delete (with inline confirm).

Convert the list item's main content from `<button>` to `<div role="button" tabIndex={0}>` with manual `onClick` + `onKeyDown` (Enter/Space) handlers. Native `<button>` elements intercept touch events on some browsers, preventing the long-press timer from completing. The div approach lets the long-press work while preserving accessibility.

## 10.7 Inline Destructive Confirm

For destructive actions (Delete, Reset, Clear), don't use the native `confirm()` dialog — it's jarring and breaks the native feel. Instead, show an inline "Are you sure?" step inside the context menu itself. Two buttons: Cancel (returns to menu) and Delete (red, executes the action). This keeps the user in the context menu flow and feels like a native app.

## 10.8 Pull-Up Load More

For infinite-scroll lists, add a "pull-up to load more" footer indicator. When the user scrolls near the bottom, show a spinner and trigger the load. When there's no more data, show "No more items". This is the inverse of pull-to-refresh and is the standard pattern for paginated lists (Gmail, Twitter, etc.).

## 10.9 Empty-State Illustrations

Every list screen should have a thoughtful empty state: an icon or illustration, a bold "No items yet" message, a helpful description, and a CTA button ("Add your first item"). This is far better than a blank screen, which makes the user think the app is broken. The CTA should jump straight to the import/create flow.


---

# Chapter 11 — Performance & Build Pipeline (9 Features)

This chapter covers the build pipeline and performance optimizations that produce a small, fast, side-loadable APK.

## 11.1 Next.js Static Export Config

The single most important config change. Convert `output: "standalone"` to `output: "export"`. Add `assetPrefix: "./"` for relative URLs (Capacitor serves from `capacitor://localhost`). Add `trailingSlash: true` so every route resolves to a directory `index.html`. Set `images.unoptimized: true` (no Node server to optimize). Set `typescript.ignoreBuildErrors` and `eslint.ignoreDuringBuilds` to true — they slow the build and the source likely has minor type issues that don't affect runtime.

## 11.2 Strip /api Routes + Inline Client-Side

Delete `src/app/api/`. For each route, port the logic into a store action or lib function. Update `fetch("/api/...")` calls to call the inlined function. See Chapter 2.1.2 for details.

## 11.3 Strip Server-Only Deps

Delete `prisma/`, `src/lib/db.ts`, `.env`. Remove `@prisma/client`, `prisma`, `next-auth`, `next-intl`, `sharp` from `package.json`. See Chapter 2.1.3.

## 11.4 CSS Font url() Bundling Fix

The most common visual bug. See Chapter 3 for the full 4-step fix. Copy fonts to `public/`, write override `@font-face` with absolute URLs, import after vendor CSS, verify in the APK.

## 11.5 Auto-Generate Icons + Splash

Use `@capacitor/assets` to generate Android icons and splash screens in all densities from a single source PNG. Place `resources/icon.png` (1024×1024) and `resources/splash.png` (1024×1024 dark background). Run `npx @capacitor/assets generate --android`. This creates 123 files (icons + splash in all densities) totaling ~770 KB.

## 11.6 Persistent Release Keystore

Generate a persistent keystore with 10000-day validity so the release APK is directly side-loadable (no Play Store upload key needed). Configure `signingConfigs.debug` to point at it, and set `buildTypes.release.signingConfig = signingConfigs.debug`. For Play Store distribution later, swap the keystore path to a real upload key — one line change.

```bash
# Generate keystore (one-time)
keytool -genkeypair -v \
  -keystore /home/z/my-project/scripts/app-release.keystore \
  -storepass android -keypass android \
  -alias app-release \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -dname "CN=App Name, OU=Team, O=Company, L=City, ST=State, C=XX"

# android/app/build.gradle:
android {
  buildTypes {
    release {
      signingConfig signingConfigs.debug
    }
  }
  signingConfigs {
    debug {
      storeFile file('/home/z/my-project/scripts/app-release.keystore')
      storePassword 'android'
      keyAlias 'app-release'
      keyPassword 'android'
    }
  }
}
```

## 11.7 AndroidManifest Customization

Customize `android/app/src/main/AndroidManifest.xml` with permissions, intent filters, and application attributes. Permissions: `INTERNET`, `VIBRATE` (for haptics), `ACCESS_NETWORK_STATE` (network indicator), `POST_NOTIFICATIONS` (Android 13+ for local notifications), `WAKE_LOCK` (keep-awake), `READ_MEDIA_IMAGES`/`VIDEO`/`AUDIO` (file picker on Android 13+), `READ_EXTERNAL_STORAGE` with `maxSdkVersion=32` (file picker on older Android). Application attributes: `android:usesCleartextTraffic="true"` (for HTTP lessons), `android:hardwareAccelerated="true"`. Intent filters: see Chapter 9.5 for file type associations.

## 11.8 Turbopack + minSdk 26

Next.js 16 uses Turbopack by default for faster dev builds. Set `minSdkVersion = 26` in `android/variables.gradle` (Android 8.0+) — this gives you modern WebView features (service workers, async/await, optional chaining) and covers ~98% of Android devices in 2026. Set `versionCode` and `versionName` in `android/app/build.gradle` (e.g. 200 and "2.0.0").

## 11.9 Lazy-Load Native Plugins

All native plugin imports MUST use dynamic `import()` guarded by `Capacitor.isNativePlatform()`. This keeps the web build clean (doesn't bundle native bridges) and avoids runtime errors on web. See Chapter 7.1 for the pattern. Every file that touches a Capacitor plugin should follow this pattern.

The build pipeline: `npm run build` → `npx cap sync android` → `cd android && ./gradlew assembleDebug assembleRelease`. Verify: `aapt2 dump badging` shows correct package + version, `unzip -l` shows fonts bundled, `apksigner verify` succeeds. Copy APKs to `/home/z/my-project/download/`.


---

# Chapter 12 — Accessibility & UX (6 Features)

This chapter covers 6 accessibility and UX features that make the app usable by everyone and delightful for power users.

## 12.1 In-App Settings Panel

Build a Sheet (bottom slide-up) with rows for: Theme toggle, Font & UI Size, Sandbox toggle, Auto-Sync on Load, Sync Cooldown, Duplicate Handling policy, Daily Study Reminder hour, Select All Text toggle. Each row has an icon, title, description, and a control (Switch, Select, or buttons). Persist via the store's settings object.

## 12.2 Font Scale Slider

Settings → Font & UI Size slider (XS, S, M, L, XL, 2XL, 3XL — values 0.85 to 1.3). Apply as a CSS variable `--font-scale` on `<html>` that multiplies the root font-size: `html { font-size: calc(16px * var(--font-scale)); }`. All rem-based Tailwind utilities scale proportionally. This is a one-line CSS change that scales the entire UI for accessibility.

```css
/* globals.css */
html {
  font-size: calc(16px * var(--clil-font-scale));
}
/* Apply the variable from JS:
document.documentElement.style.setProperty('--clil-font-scale', '1.2');
*/
```

## 12.3 First-Launch Tutorial Overlay

On first launch, show a brief tutorial overlay highlighting the tab bar, import button, and key features. Use a stored `hasSeenTutorial` flag (in `@capacitor/preferences` for persistence). The overlay should be dismissible with a "Got it" button and not show again. Keep it to 3-4 highlights max — anything longer feels like a slideshow.

## 12.4 Long-Press Tab → Settings

Power-user shortcut: long-press the footer tab bar (specifically the Settings tab) to open Settings directly, skipping the normal tab switch. This rewards users who learn the gesture and provides instant access to the most-used settings. Fire `haptic.medium()` on the long-press to confirm.

## 12.5 Background Task Notification

When a long-running background task finishes (large file download, library sync, batch import), fire a local notification via `@capacitor/local-notifications`. This wakes up the app if it's backgrounded and lets the user know the task completed. Title: "App Name", body: "Sync complete — 12 items added."

## 12.6 Screen-Reader Labels on Icon Buttons

Add `aria-label` to every icon-only button. Screen readers can't interpret an icon — they need a text label. Without `aria-label`, the button is announced as "button" with no context. With it, it's announced as "Share lesson, button". This is a 30-minute pass that makes the app usable for blind users.

```tsx
<Button
  variant="ghost"
  size="icon"
  onClick={handleShare}
  aria-label="Share lesson"  // ← critical for screen readers
>
  <Share2 className="h-3.5 w-3.5" />
</Button>
```


---

# Chapter 13 — The 20 Most Common Errors & Fixes

Every error in this chapter has been hit and fixed in production. When you encounter one of these, jump to the fix — don't waste time debugging from scratch. The errors are ordered roughly by frequency.

| # | Error | Root Cause + Fix |
|---|-------|-----------------|
| 1 | npm ERESOLVE peer dep conflict | @capacitor-community/sqlite@6 requires @capacitor/core@^6 but you're using v7. Upgrade the plugin to v7 (sqlite@^7.0.3, keep-awake@^7.1.0). Or run `npm install` with `--legacy-peer-deps` as a blanket fix. |
| 2 | cmdline-tools ClassNotFoundException: SdkManagerCli | You downloaded cmdline-tools v10 which has a broken layout. Delete everything and re-download v9 (`commandlinetools-linux-9477386_latest.zip`). v9 extracts cleanly to `{bin, lib}`; v10 extracts to a nested directory missing the classpath jar. |
| 3 | Gradle: Toolchain does not provide JAVA_COMPILER | System has `openjdk-21-jre-headless` (JRE only, no `javac`). Install the portable Temurin JDK 21 (download from GitHub `adoptium/temurin21-binaries` releases). Set `JAVA_HOME` to the extracted `jdk-21.0.5+11/` directory. |
| 4 | 7z extraction without p7zip installed | Can't `apt install p7zip-full` (no sudo). `pip install py7zr` fails on PEP 668 + slow brotli wheel. Fix: download standalone `7zz` binary from GitHub `ip7z/7zip` releases (`7z2408-linux-x64.tar.xz`), extract, use `7zz x file.7z -o/output -y`. |
| 5 | Static export breaks /api routes | Next.js `output: "export"` can't have API routes. Delete `src/app/api/`. Port each route's logic client-side into store actions or lib functions. Update `fetch("/api/...")` calls to call the inlined function. |
| 6 | Prisma not needed in static export | Prisma tries to connect to a DB at build time. Delete `prisma/`, `src/lib/db.ts`, `.env`. Remove `@prisma/client` and `prisma` from `package.json`. The app likely doesn't use Prisma — it was just scaffolded. |
| 7 | Session instability / bash tool timeouts | Long-running foreground processes time out the IM gateway. Run downloads via `nohup setsid bash -c '...' & disown` so they're detached. Poll with short `sleep 30` checks. Write a master `bg-setup.sh` script that does JDK + SDK + npm sequentially in the background. |
| 8 | KaTeX renders as raw text | Three compounding issues: Next.js doesn't rewrite `url()` in node_modules CSS, fonts aren't copied to `out/`, Capacitor can't fetch above `public/`. Fix: copy fonts to `public/katex-fonts/`, write override `@font-face` with absolute `/katex-fonts/` URLs, import after `katex.min.css`. ALSO: if your DSL uses triple-quoted strings, normalize `\\(`→`\(` before rendering. |
| 9 | Toast dismiss only worked via back button | Sonner's default config doesn't enable tap or swipe dismiss. Add `closeButton`, `swipeDirections={["left","right","up","down"]}`, and a capture-phase click listener that finds `[data-sonner-toast]` and calls `toast.dismiss(id)` (not `toast.dismiss()` which dismisses ALL). |
| 10 | Text selection on UI elements (annoying) | Long-pressing UI elements pops up Android's text-selection handles. Fix: global `* { user-select: none; -webkit-touch-callout: none; -webkit-tap-highlight-color: transparent; }`. Re-enable in content areas via `.selectable` class. Add a Settings toggle for global selection. |
| 11 | Data loss on app close | Three issues: (a) debounced write doesn't flush before kill — add `visibilitychange` + `pagehide` + `beforeunload` + 3s interval flush triggers. (b) zustand auto-hydrate races with user interaction — add `skipHydration: true` + manual `rehydrate()`. (c) `localStorage` wiped with cache — use Filesystem `Directory.External` as primary. |
| 12 | CSS url() fonts not loading in APK | Same as #8. The CSS bundle references `url(fonts/X.woff2)` which doesn't resolve under `capacitor://localhost`. Copy fonts to `public/`, write override `@font-face` with `/vendor-fonts/X.woff2` absolute URLs. |
| 13 | localStorage cleared with cache | Android clears `localStorage` when the user clears app cache. Use `@capacitor/filesystem` `Directory.External` (`Android/data/<pkg>/files/`) as the primary store — it survives cache clears. |
| 14 | File picker E404: @capawesome-team not found | The correct package name is `@capawesome/capacitor-file-picker` (without `-team`). Use `^7.2.0` for Capacitor v7. |
| 15 | keep-screen-awake package not found | The package was renamed to `@capacitor-community/keep-awake`. Use `^7.1.0`. The old name `@capacitor-community/keep-screen-awake` doesn't exist on npm. |
| 16 | Intent-filter pathPattern only matches root | Android's `pathPattern` is a simple match, not regex. `.*\.ext` only matches files in the root directory. Add nested patterns: `.*\..*\.ext` (one level deep), `.*\..*\..*\.ext` (two levels). Three levels is usually enough. |
| 17 | Splash screen doesn't show | Verify `@capacitor/assets` generated all densities (`drawable-port-hdpi` through `drawable-port-xxxhdpi`). Check `capacitor.config.ts` `plugins.SplashScreen` config. The splash hides automatically after `launchShowDuration`; also call `SplashScreen.hide()` programmatically once React mounts. |
| 18 | Release APK is 'unsigned' | Configure `signingConfigs.debug` to point at a persistent keystore, and set `buildTypes.release.signingConfig = signingConfigs.debug`. Run `keytool -genkeypair` with 10000-day validity. The release APK will then be signed and directly side-loadable. |
| 19 | WebView can't fetch above public/ | Capacitor serves the WebView from `capacitor://localhost` with no filesystem access above `public/`. Any font/image/asset must be copied to `public/` so it gets bundled into the APK at `assets/public/`. |
| 20 | Backgrounding loses state | The debounced write timer doesn't fire when the app is backgrounded. Add a `visibilitychange` listener that force-flushes pending writes when `document.visibilityState === "hidden"`. Also add `pagehide` + `beforeunload` + 3s interval as backups. |

---

# Chapter 14 — Build & Delivery Checklist

Run through this 30-item checklist for every build. Each phase has a specific goal; don't skip ahead.

## A. Pre-Build Verification

- [ ] Source tree audited (no /api, no prisma, no server-only deps)
- [ ] next.config.ts converted to `output: "export"` with assetPrefix, trailingSlash, images.unoptimized
- [ ] package.json updated with all 15 Capacitor plugins + jszip
- [ ] capacitor.config.ts written with appId, appName, webDir, plugin configs
- [ ] KaTeX (or other vendor) fonts copied to public/vendor-fonts/
- [ ] Override @font-face CSS written and imported after vendor CSS in layout.tsx
- [ ] All native feature hooks written (back-button, storage, haptics, etc.)
- [ ] AndroidManifest.xml customized with permissions + intent filters

## B. Build

- [ ] `npm run build` succeeds (Next.js static export → out/)
- [ ] Verify: `ls out/vendor-fonts/ | wc -l` shows ~60 font files
- [ ] Verify: `grep -l vendor-fonts out/_next/static/chunks/*.css` finds the override
- [ ] `npx cap sync android` succeeds (copies web assets + syncs plugins)
- [ ] `./gradlew assembleDebug` succeeds (→ app-debug.apk)
- [ ] `./gradlew assembleRelease` succeeds (→ app-release.apk)

## C. APK Verification

- [ ] `aapt2 dump badging` shows correct package + versionCode + versionName + minSdk
- [ ] `unzip -l app-release.apk | grep vendor-fonts | wc -l` shows all fonts bundled
- [ ] `apksigner verify app-release.apk` succeeds (APK is signed)
- [ ] Install on a physical Android 8+ device (no emulator)
- [ ] Smoke test: open the app, navigate, verify KaTeX renders, close + reopen to verify persistence

## D. Delivery

- [ ] Copy both APKs to `/home/z/my-project/download/`
- [ ] Upload both APKs to CloudVault (if available) for cross-session persistence
- [ ] Write a README.md with install instructions + feature list
- [ ] Update the worklog at `/home/z/my-project/worklog.md`

## E. Post-Delivery

- [ ] Ask the user to install the release APK and test the key flows
- [ ] Document any issues the user reports for the next iteration
- [ ] Prepare for iteration (the first build always has something to fix)

---

# Appendix A — Copy-Pasteable Config Files

Every config file you need, ready to copy-paste. Adapt the package name, app name, and file extensions to your project.

## A.1 next.config.ts (Static Export)

```typescript
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

## A.2 capacitor.config.ts

```typescript
import type { CapacitorConfig } from "@capacitor/cli";
const config: CapacitorConfig = {
  appId: "com.yourcompany.yourapp",
  appName: "Your App",
  webDir: "out",
  android: {
    allowMixedContent: true,
    webContentsDebuggingEnabled: true,
  },
  server: { androidScheme: "https" },
  plugins: {
    SplashScreen: {
      launchShowDuration: 1500,
      launchAutoHide: true,
      backgroundColor: "#0b0b0d",
      androidScaleType: "CENTER_CROP",
      splashFullScreen: true,
      splashImmersive: true,
    },
    LocalNotifications: {
      smallIcon: "ic_stat_icon",
      iconColor: "#0b0b0d",
      sound: "bell.wav",
    },
    StatusBar: {
      backgroundColor: "#0b0b0d",
      style: "DARK",
      overlaysWebView: false,
    },
    ScreenOrientation: { orientation: "portrait" },
  },
};
export default config;
```

## A.3 package.json Dependencies (Capacitor Plugins)

```json
"dependencies": {
  "@capacitor/android": "^7.0.0",
  "@capacitor/app": "^7.0.0",
  "@capacitor/assets": "^3.0.5",
  "@capacitor/browser": "^7.0.0",
  "@capacitor/core": "^7.0.0",
  "@capacitor/filesystem": "^7.0.0",
  "@capacitor/haptics": "^7.0.0",
  "@capacitor/keyboard": "^7.0.0",
  "@capacitor/local-notifications": "^7.0.0",
  "@capacitor/network": "^7.0.0",
  "@capacitor/preferences": "^7.0.0",
  "@capacitor/screen-orientation": "^7.0.0",
  "@capacitor/share": "^7.0.0",
  "@capacitor/splash-screen": "^7.0.0",
  "@capacitor/status-bar": "^7.0.0",
  "@capacitor-community/keep-awake": "^7.1.0",
  "@capacitor-community/sqlite": "^7.0.3",
  "@capawesome/capacitor-file-picker": "^7.2.0",
  "jszip": "^3.10.1"
}
```

## A.4 AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  <application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:hardwareAccelerated="true"
    android:usesCleartextTraffic="true"
    android:theme="@style/AppTheme">
    <activity
      android:name=".MainActivity"
      android:label="@string/title_activity_main"
      android:theme="@style/AppTheme.NoActionBarLaunch"
      android:launchMode="singleTask"
      android:exported="true"
      android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|smallestScreenSize|screenLayout|uiMode|navigation">
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
      <!-- File type association -->
      <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="content" />
        <data android:scheme="file" />
        <data android:host="*" />
        <data android:pathPattern=".*\.ext" />
        <data android:pathPattern=".*\..*\.ext" />
        <data android:pathPattern=".*\..*\..*\.ext" />
        <data android:mimeType="text/plain" />
        <data android:mimeType="application/octet-stream" />
      </intent-filter>
    </activity>
    <provider
      android:name="androidx.core.content.FileProvider"
      android:authorities="${applicationId}.fileprovider"
      android:exported="false"
      android:grantUriPermissions="true">
      <meta-data android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
    </provider>
  </application>
  <uses-permission android:name="android.permission.INTERNET" />
  <uses-permission android:name="android.permission.VIBRATE" />
  <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
  <uses-permission android:name="android.permission.WAKE_LOCK" />
  <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
  <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
  <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
</manifest>
```

## A.5 variables.gradle

```gradle
ext {
  minSdkVersion = 26
  compileSdkVersion = 35
  targetSdkVersion = 35
  androidxActivityVersion = '1.9.2'
  androidxAppCompatVersion = '1.7.0'
  androidxCoordinatorLayoutVersion = '1.2.0'
  androidxCoreVersion = '1.15.0'
  androidxFragmentVersion = '1.8.4'
  coreSplashScreenVersion = '1.0.1'
  androidxWebkitVersion = '1.12.1'
  junitVersion = '4.13.2'
  androidxJunitVersion = '1.2.1'
  androidxEspressoCoreVersion = '3.6.1'
  cordovaAndroidVersion = '10.1.1'
}
```

## A.6 app/build.gradle Signing Config

```gradle
android {
  defaultConfig {
    applicationId "com.yourcompany.yourapp"
    minSdkVersion rootProject.ext.minSdkVersion
    targetSdkVersion rootProject.ext.targetSdkVersion
    versionCode 200
    versionName "2.0.0"
  }
  buildTypes {
    release {
      minifyEnabled false
      proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
      signingConfig signingConfigs.debug
    }
  }
  signingConfigs {
    debug {
      storeFile file('/home/z/my-project/scripts/app-release.keystore')
      storePassword 'android'
      keyAlias 'app-release'
      keyPassword 'android'
    }
  }
}
```

## A.7 KaTeX @font-face Override (First 5 of 20 Families)

```css
/* src/styles/katex-fonts.css — import AFTER katex.min.css */
@font-face {
  font-family: 'KaTeX_AMS';
  src: url(/katex-fonts/KaTeX_AMS-Regular.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_AMS-Regular.woff) format('woff'),
       url(/katex-fonts/KaTeX_AMS-Regular.ttf) format('truetype');
  font-weight: normal; font-style: normal;
}
@font-face {
  font-family: 'KaTeX_Caligraphic';
  src: url(/katex-fonts/KaTeX_Caligraphic-Bold.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_Caligraphic-Bold.woff) format('woff'),
       url(/katex-fonts/KaTeX_Caligraphic-Bold.ttf) format('truetype');
  font-weight: bold; font-style: normal;
}
@font-face {
  font-family: 'KaTeX_Caligraphic';
  src: url(/katex-fonts/KaTeX_Caligraphic-Regular.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_Caligraphic-Regular.woff) format('woff'),
       url(/katex-fonts/KaTeX_Caligraphic-Regular.ttf) format('truetype');
  font-weight: normal; font-style: normal;
}
@font-face {
  font-family: 'KaTeX_Fraktur';
  src: url(/katex-fonts/KaTeX_Fraktur-Bold.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_Fraktur-Bold.woff) format('woff'),
       url(/katex-fonts/KaTeX_Fraktur-Bold.ttf) format('truetype');
  font-weight: bold; font-style: normal;
}
@font-face {
  font-family: 'KaTeX_Fraktur';
  src: url(/katex-fonts/KaTeX_Fraktur-Regular.woff2) format('woff2'),
       url(/katex-fonts/KaTeX_Fraktur-Regular.woff) format('woff'),
       url(/katex-fonts/KaTeX_Fraktur-Regular.ttf) format('truetype');
  font-weight: normal; font-style: normal;
}
/* ... 15 more @font-face rules for all KaTeX font families ... */
```


---

# Appendix B — Feature Index (58 Features at a Glance)

Quick-reference table of all features. Use this to scan and pick which to implement based on your user's request. **Complexity**: Low (<30 min), Med (1-3 hours), High (3+ hours). **Impact**: Low (nice-to-have), Med (noticeable improvement), High (critical for native feel).

| # | Feature | Category | Plugin / Hook | Complex | Impact |
|---|---------|----------|---------------|---------|--------|
| 1 | Back button bridge | Navigation | @capacitor/app | Med | High |
| 2 | Double-tap exit confirm | Navigation | useBackButton hook | Low | Med |
| 3 | Page transition animations | Navigation | framer-motion | Low | Med |
| 4 | External Filesystem JSON | Storage | @capacitor/filesystem | Med | High |
| 5 | SQLite mirror | Storage | @capacitor-community/sqlite | Med | Med |
| 6 | localStorage sync cache | Storage | native (web API) | Low | High |
| 7 | Multi-trigger flush safety net | Storage | visibilitychange/pagehide | Low | High |
| 8 | Skip-hydrate race fix | Storage | zustand skipHydration | Low | High |
| 9 | Backup ZIP export/import | Storage | jszip + @capacitor/share | Med | Med |
| 10 | Preferences 4th layer | Storage | @capacitor/preferences | Low | Low |
| 11 | Light tap haptic | Haptics | @capacitor/haptics | Low | High |
| 12 | Success/error haptics | Haptics | @capacitor/haptics | Low | High |
| 13 | Long-press confirm haptic | Haptics | @capacitor/haptics | Low | Med |
| 14 | Pull-to-refresh haptic | Haptics | @capacitor/haptics | Low | Med |
| 15 | Toast dismiss haptic | Haptics | @capacitor/haptics | Low | Med |
| 16 | Long-press context menu | Haptics | useLongPress hook | Med | High |
| 17 | Pull-to-refresh hook | Haptics | usePullToRefresh hook | Med | Med |
| 18 | Pinch-to-zoom on media | Haptics | custom touch handler | Med | Low |
| 19 | Swipe-to-dismiss (4-dir) | Haptics | sonner swipeDirections | Low | Med |
| 20 | Orientation lock per-screen | Immersive | @capacitor/screen-orientation | Low | Med |
| 21 | Keep screen awake | Immersive | @capacitor-community/keep-awake | Low | High |
| 22 | Immersive status bar hide | Immersive | @capacitor/status-bar | Low | Med |
| 23 | Status bar theme match | Immersive | @capacitor/status-bar | Low | Med |
| 24 | Native splash screen | Immersive | @capacitor/splash-screen | Low | Med |
| 25 | Safe area insets | Immersive | env() CSS + JS probe | Low | High |
| 26 | Theme follows system | Immersive | matchMedia prefers-color-scheme | Low | Med |
| 27 | Theme-aware splash | Immersive | capacitor.config.ts | Low | Low |
| 28 | Network status indicator | System | @capacitor/network | Low | Med |
| 29 | Share via native sheet | System | @capacitor/share | Low | Med |
| 30 | Local notifications | System | @capacitor/local-notifications | Med | Med |
| 31 | Native file picker | System | @capawesome/capacitor-file-picker | Low | High |
| 32 | File type associations | System | AndroidManifest intent-filter | Med | High |
| 33 | Clipboard smart-detect | System | navigator.clipboard | Med | Low |
| 34 | External links in system browser | System | @capacitor/browser | Low | Med |
| 35 | Global text-select disable | UI Polish | CSS user-select:none | Low | High |
| 36 | Re-enable in content areas | UI Polish | .selectable CSS class | Low | High |
| 37 | Settings: select-all toggle | UI Polish | .selectable-everywhere class | Low | Med |
| 38 | Toast tap/swipe dismiss | UI Polish | sonner config + click handler | Low | High |
| 39 | Toast offset above tab bar | UI Polish | sonner offset prop | Low | Low |
| 40 | Long-press list context menu | UI Polish | portal-rendered menu | High | High |
| 41 | Inline destructive confirm | UI Polish | context menu sub-step | Low | Med |
| 42 | Pull-up load more | UI Polish | IntersectionObserver | Med | Low |
| 43 | Empty-state illustrations | UI Polish | React component | Low | Med |
| 44 | Next.js static export config | Build | next.config.ts | Low | High |
| 45 | Strip /api routes + inline | Build | manual migration | Med | High |
| 46 | Strip server-only deps | Build | delete prisma/db.ts | Low | High |
| 47 | CSS font url() bundling fix | Build | copy fonts + override CSS | Med | High |
| 48 | Auto-generate icons + splash | Build | @capacitor/assets | Low | Med |
| 49 | Persistent release keystore | Build | keytool + build.gradle | Low | High |
| 50 | AndroidManifest customization | Build | XML edits | Med | High |
| 51 | Turbopack + minSdk 26 | Build | next.config + variables.gradle | Low | Med |
| 52 | Lazy-load native plugins | Build | dynamic import() pattern | Low | High |
| 53 | In-app settings panel | A11y/UX | Sheet component | Med | Med |
| 54 | Font scale slider | A11y/UX | CSS variable on <html> | Low | Med |
| 55 | First-launch tutorial overlay | A11y/UX | stored hasSeenTutorial flag | Med | Low |
| 56 | Long-press tab → Settings | A11y/UX | useLongPress on tab | Low | Low |
| 57 | Background task notification | A11y/UX | @capacitor/local-notifications | Med | Low |
| 58 | Screen-reader labels | A11y/UX | aria-label on icon buttons | Low | Med |

---

*End of guide.*
