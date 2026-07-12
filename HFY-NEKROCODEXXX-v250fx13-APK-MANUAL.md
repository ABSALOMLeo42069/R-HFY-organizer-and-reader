# HFY NEKROCODEXXX v250fx13 — APK Recreation Manual

Version: v250fx13 | Dev: Mahmudul Hossain

---

## Chapters

1. Introduction
2. Architecture
3. Prerequisites
4. Base APK
5. NodeRunner.java
6. NodeForegroundService.java
7. MainActivity.java
8. AndroidManifest
9. Gradle
10. Building
11. Signing
12. WiFi Lock
13. Auto-Restart
14. 250kbps Keepalive
15. Aggregate Fix
16. Port Protection
17. Browser Simulation
18. UA Rotation
19. OAuth Multi-Account Token Rotation (v250fx13)
20. Library & Favorites Persistence (v250fx10–v250fx12)
21. Server-Side Store API (v250fx10)
22. Chapter Persistence (v250fx10)
23. Troubleshooting
24. Build Script
25. File Sizes
26. Native Libraries
27. Source Code

---

# Chapter 1: Introduction

The APK bundles Node.js + Next.js + Java bridge into a single Android application. The Node.js server runs on port 3000 inside the app, and a WebView displays the React frontend. Features include a 250kbps WiFi lock, auto-restart on crash, and persistent filesystem storage.

**v250fx13** adds comprehensive OAuth multi-account token rotation across all fetch paths, library/favorites persistence fixes, and chapter persistence to IndexedDB.

---

# Chapter 2: Architecture

```
MainActivity (WebView)
    → NodeForegroundService (wakelock + wifilock + keepalive)
        → NodeRunner (native Node.js process)
            → server.js (Next.js + proxy + store API)
```

The server uses 9 fetch strategies, all with OAuth token rotation when multiple accounts are connected.

---

# Chapter 3: Prerequisites

- JDK 17 or later
- Android SDK (platform-34, build-tools 36.0.0)
- Gradle 8.14 or later
- Bun (for building webapp from source)
- apksigner, zipalign (from build-tools)
- Keystore for signing

---

# Chapter 4: Base APK

Take the base APK (built from the Android project with Capacitor), replace `assets/server/` with the latest webapp build.

```bash
# Copy base APK
cp base.apk HFY-v250fx13.apk

# Replace server files (uncompressed with -0 for Node.js memory-mapping)
cd /tmp/apk-update
zip -0 HFY-v250fx13.apk \
    assets/server/.next/server/chunks/4034.js \
    assets/server/.next/static/chunks/app/page-94aba2608e3f02d1.js \
    assets/server/.next/static/chunks/9365-d1a77d1fb931ad63.js \
    assets/server/.next/BUILD_ID \
    assets/server/package.json
```

---

# Chapter 5: NodeRunner.java

- Accepts `server.js` or `main.js`
- `volatile Process` + `volatile boolean keepRunning`
- Auto-restart: 3s delay after crash
- Sets `HFY_DATA_DIR` to `getFilesDir()/hfy-data/`

```java
package com.nekro.hfycyberdeck;

public class NodeRunner {
    private volatile Process nodeProcess;
    private volatile boolean keepRunning = false;

    // startServer():
    //   1. Extract native libraries (libnode.so, etc.)
    //   2. Extract server files to getFilesDir()
    //   3. Test node binary with --version
    //   4. Launch via linker64 with environment variables
    //   5. Start log reader thread (auto-restart on crash)

    // stopServer():
    //   keepRunning = false (prevents auto-restart)
    //   destroy nodeProcess
}
```

---

# Chapter 6: NodeForegroundService.java

- `PARTIAL_WAKE_LOCK` + `WIFI_MODE_FULL_HIGH_PERF` (permanent, `setReferenceCounted(false)`)
- 250kbps keepalive (250KB download + 250KB upload every 1 second)
- `onTaskRemoved` / `onDestroy`: restart unless `userRequestedStop`
- `START_STICKY` for system-initiated restart

---

# Chapter 7: MainActivity.java

- WebView with JavaScript, DOM storage, file access
- `AndroidFolderPicker` JS interface for download directory selection
- Fullscreen immersive mode
- `onDestroy`: sets `userRequestedStop` + stops service

---

# Chapter 8: AndroidManifest.xml

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
<uses-permission android:name="android.permission.VIBRATE" />
```

`foregroundServiceType="dataSync"`, `extractNativeLibs="true"`.

---

# Chapter 9: Gradle

```groovy
dependencies {
    implementation project(':capacitor-android')
    // NOT flatDir AAR — misses transitive dependencies
}
```

Only safe to skip: `lintVitalAnalyzeRelease`, `lintVitalReportRelease`, `lintVitalRelease`.
Never skip: `compressReleaseAssets`, `stripReleaseDebugSymbols`, `mergeReleaseNativeLibs`, `packageRelease`.

---

# Chapter 10: Building

```bash
# 1. Build webapp
cd src/
bun install
bunx next build --webpack
cp -r .next/static .next/standalone/.next/
cp -r public .next/standalone/

# 2. Build APK from Android project
cd android/
./gradlew assembleRelease

# 3. Replace server in APK
cp app/build/outputs/apk/release/app-release.apk HFY-v250fx13.apk
zip -0 HFY-v250fx13.apk assets/server/.next/server/chunks/4034.js
zip -0 HFY-v250fx13.apk assets/server/.next/static/chunks/app/page-94aba2608e3f02d1.js
zip -0 HFY-v250fx13.apk assets/server/.next/static/chunks/9365-d1a77d1fb931ad63.js
zip -0 HFY-v250fx13.apk assets/server/.next/BUILD_ID
zip -0 HFY-v250fx13.apk assets/server/package.json
```

---

# Chapter 11: Signing

```bash
# Zipalign
zipalign -v -p 4 HFY-v250fx13.apk HFY-v250fx13-aligned.apk

# Sign with apksigner (v2 + v3)
apksigner sign \
    --ks hfy-release.keystore \
    --ks-key-alias hfy-nekro \
    --ks-pass pass:nekrocodexxx2026 \
    --key-pass pass:nekrocodexxx2026 \
    --out HFY-v250fx13-signed.apk \
    HFY-v250fx13-aligned.apk

# Verify
apksigner verify --verbose HFY-v250fx13-signed.apk
```

**Note:** Different versions may use different signing keys. Users must uninstall the previous version before installing a new one.

---

# Chapter 12: WiFi Lock

```java
WifiManager.WifiLock wifiLock = wifiManager.createWifiLock(
    WifiManager.WIFI_MODE_FULL_HIGH_PERF, "hfy-wifi-lock"
);
wifiLock.setReferenceCounted(false);
wifiLock.acquire();
```

`WIFI_MODE_FULL_HIGH_PERF` — maximum throughput, no power-saving.

---

# Chapter 13: Auto-Restart

If the Node.js process exits:
1. Log reader thread detects process exit
2. Waits 3 seconds
3. Checks `keepRunning` flag
4. Calls `startServer()` again

If the foreground service is killed:
1. `onTaskRemoved` restarts (unless `userRequestedStop`)
2. `onDestroy` restarts (unless `userRequestedStop`)
3. `START_STICKY` tells the system to restart

---

# Chapter 14: 250kbps Keepalive

```java
// Downloads 250KB from rotating CDN URLs every second
// Uploads 250KB of random data to rotating echo services every second
// Total: 500KB/s = ~41 GB/day (designed for WiFi)
```

Prevents Android 14+ from freezing the background process.

---

# Chapter 15: Aggregate Fix

`aggregate-counts.js` uses `http.request()` instead of `fetch()` because `fetch` is not available in the Android Node.js runtime. This was a crash bug in earlier versions.

---

# Chapter 16: Port Protection

```javascript
// server.js checks for EADDRINUSE before starting
const testServer = net.createServer();
testServer.once('error', (e) => {
    if (e.code === 'EADDRINUSE') {
        console.log('Port in use, waiting 5s...');
        setTimeout(() => process.exit(0), 5000);
    }
});
testServer.once('listening', () => {
    testServer.close();
    startNextServer();
});
```

---

# Chapter 17: Browser Simulation

Full Chrome browser headers are sent to make Reddit think requests come from a real browser:

```
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Sec-Ch-Ua: "Google Chrome";v="131", "Chromium";v="131"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Accept-Encoding: gzip, deflate, br
Upgrade-Insecure-Requests: 1
```

---

# Chapter 18: UA Rotation

8 browser User-Agent strings are rotated:

- Chrome on Windows, macOS, Linux
- Firefox on Windows, macOS, Linux
- Safari on macOS
- Edge on Windows

---

# Chapter 19: OAuth Multi-Account Token Rotation (v250fx13)

## Problem

In v250fx12 and earlier, the multi-account token rotation only worked for the primary OAuth fetch strategy (Strategy 0). All other fetch paths (RSS, wiki JSON, search, browser simulation, fallback JSON, score fetch, subreddit feed, RedDownloader) used plain `fetch()` with no `Authorization` header.

This meant that even with 9 accounts connected, most requests went out unauthenticated, causing Reddit to rate-limit the server IP.

## Solution

### Server-side changes (`4034.js`)

**1. `getNextMultiToken()` is now async and refreshes expired tokens:**

```javascript
async function getNextMultiToken() {
    const store = getMultiTokenStore();
    if (store.size === 0) return null;
    const allAccounts = Array.from(store.entries()).filter(([, v]) =>
        v.accessToken && v.clientId !== "cookies" && v.clientId !== "cookie-injection"
    );
    if (allAccounts.length === 0) return null;
    let liveAccounts = allAccounts.filter(([, v]) => Date.now() < v.expiresAt - 5 * 60 * 1000);
    if (liveAccounts.length === 0) {
        // Try to refresh expired tokens
        for (const [id, v] of allAccounts) {
            if (v.refreshToken && v.clientId) {
                try {
                    const refreshRes = await fetch("https://www.reddit.com/api/v1/access_token", {
                        method: "POST",
                        headers: {
                            "Content-Type": "application/x-www-form-urlencoded",
                            Authorization: `Basic ${Buffer.from(v.clientId + ":").toString("base64")}`,
                            "User-Agent": USER_AGENT_DFR
                        },
                        body: new URLSearchParams({
                            grant_type: "refresh_token",
                            refresh_token: v.refreshToken
                        }).toString(),
                        signal: AbortSignal.timeout(10000),
                        cache: "no-store"
                    });
                    if (refreshRes.ok) {
                        const refreshData = await refreshRes.json();
                        if (refreshData.access_token) {
                            v.accessToken = refreshData.access_token;
                            v.expiresAt = Date.now() + (refreshData.expires_in || 3600) * 1000;
                            store.set(id, v);
                            liveAccounts.push([id, v]);
                        }
                    }
                } catch (e) { /* continue */ }
            }
        }
    }
    if (liveAccounts.length === 0) return null;
    const idx = (global.redditTokenRotationIndex || 0) % liveAccounts.length;
    global.redditTokenRotationIndex = idx + 1;
    const [, tokenData] = liveAccounts[idx];
    return { token: tokenData.accessToken, username: tokenData.username };
}
```

**2. `getRedditOAuthToken()` awaits the async `getNextMultiToken()`:**

```javascript
async function getRedditOAuthToken() {
    const multi = await getNextMultiToken();
    if (multi) return { token: multi.token, tier: "user", username: multi.username };
    const userToken = getUserToken();
    if (userToken) return { token: userToken, tier: "user" };
    const appToken = await getAppOnlyToken();
    if (appToken) return { token: appToken, tier: "app" };
    return null;
}
```

**3. All 8 previously-unauthenticated fetch paths now use OAuth:**

Each fetch path follows this pattern:

```javascript
// Before:
const res = await fetch(url, {
    headers: { "User-Agent": getRandomUA(), Accept: "application/json" },
    ...
});

// After:
const oauthAuth = await getRedditOAuthToken();
const hdrs = { "User-Agent": getRandomUA(), Accept: "application/json" };
if (oauthAuth) hdrs["Authorization"] = `Bearer ${oauthAuth.token}`;
const res = await fetch(url, {
    headers: hdrs,
    ...
});
```

The 8 patched paths:
1. Wiki series index `.json`
2. Wiki series/{slug} `.json`
3. Browser simulation `.json`
4. Last resort `.json` fallback
5. Public score fetch `.json`
6. Search `.json`
7. Subreddit RSS feed
8. `fetchWithRedDownloader`

### Client-side (already present from v250fx12)

The client already had `syncAccountsToServer()` which POSTs all connected accounts to `/api/reddit-multi-token`. This was not changed in v250fx13 — only the server-side fetch functions were fixed.

## Verification

With 9 accounts, rotation cycles: user1 → user2 → ... → user9 → user1 → ...

This provides 9 × 600 QPM = 5,400 QPM.

---

# Chapter 20: Library & Favorites Persistence (v250fx10–v250fx12)

## `addToLibrary` merge fix (v250fx12)

`addToLibrary` now merges with existing items instead of overwriting. This prevents the series detail screen's auto-add from destroying the user's favorite state.

## `toggleFavorite` auto-add (v250fx11)

If the series is not in the library, `toggleFavorite` auto-adds it with `isFavorite: true`.

## `isFav` direct store selector (v250fx11)

Uses `useLibrary((s) => !!(s.items[seriesId]?.isFavorite))` instead of a stale variable.

## Synchronous `getItem` (v250fx12)

`getItem` in the persist middleware is synchronous (localStorage only). Server restore is handled via `onRehydrateStorage`.

---

# Chapter 21: Server-Side Store API (v250fx10)

The server intercepts `/api/store/:key` requests and persists them to `{HFY_DATA_DIR}/app-store/{key}.json`. This provides durable storage that survives Android restarts.

On Android, `HFY_DATA_DIR` is set to `getFilesDir()/hfy-data/` by `NodeRunner`.

---

# Chapter 22: Chapter Persistence (v250fx10)

A `useEffect` saves chapters to IndexedDB on every change (2-second debounce). This ensures C-FETCH results, crawled chapters, renamed chapters, and reordered chapters survive app restart.

---

# Chapter 23: Troubleshooting

**"App not installed"** — Uninstall previous version (different signing keys).

**"net::ERR_CONNECTION_REFUSED"** — Server crashed. Check logcat. Port conflict protection and aggregate-counts fix should prevent this.

**Rate limited even with multiple accounts** — Fixed in v250fx13. All fetch paths now use OAuth rotation.

**Favorites lost on tab switch** — Fixed in v250fx12. `addToLibrary` merges instead of overwrites.

**Chapters lost on restart** — Fixed in v250fx10. Chapters persist to IndexedDB.

---

# Chapter 24: Build Script

```bash
#!/bin/bash
set -e

# 1. Build webapp
cd /path/to/src
bun install
bunx next build --webpack
cp -r .next/static .next/standalone/.next/
cp -r public .next/standalone/

# 2. Build base APK
cd android
./gradlew assembleRelease

# 3. Replace server
cp app/build/outputs/apk/release/app-release.apk HFY-v250fx13.apk
SERVER_DIR=.next/standalone
cd $SERVER_DIR
zip -0 /path/to/HFY-v250fx13.apk \
    assets/server/.next/server/chunks/4034.js \
    assets/server/.next/static/chunks/app/page-94aba2608e3f02d1.js \
    assets/server/.next/static/chunks/9365-d1a77d1fb931ad63.js \
    assets/server/.next/BUILD_ID \
    assets/server/package.json

# 4. Zipalign
zipalign -v -p 4 HFY-v250fx13.apk HFY-v250fx13-aligned.apk

# 5. Sign
apksigner sign \
    --ks hfy-release.keystore \
    --ks-key-alias hfy-nekro \
    --ks-pass pass:nekrocodexxx2026 \
    --key-pass pass:nekrocodexxx2026 \
    --out HFY-v250fx13-signed.apk \
    HFY-v250fx13-aligned.apk

# 6. Verify
apksigner verify --verbose HFY-v250fx13-signed.apk
```

---

# Chapter 25: File Sizes

| Component | Size |
|-----------|------|
| APK total | ~431 MB |
| libnode.so (ARM64) | ~49 MB |
| Webapp (server/) | ~191 MB |
| Native libraries | ~100 MB |
| classes.dex | ~7 MB |

---

# Chapter 26: Native Libraries

```
lib/arm64-v8a/
├── libnode.so          (49 MB — Node.js runtime)
├── libsqlite3.so       (1.6 MB)
├── libssl.so.3         (875 KB)
├── libcrypto.so.3      (5.2 MB)
├── libicudata.so.78    (33 MB)
├── libicui18n.so.78    (3.4 MB)
├── libicuuc.so.78      (2.0 MB)
├── libc++_shared.so    (1.4 MB)
├── libffi.so           (86 KB)
├── libz.so.1            (73 KB)
└── libcares.so.2       (253 KB)
```

---

# Chapter 27: Source Code

The full source code is included in the APK at `assets/server/`. Key files:

- `assets/server/server.js` — Next.js server with proxy and store API
- `assets/server/aggregate-counts.js` — periodic aggregation
- `assets/server/.next/static/chunks/app/page-94aba2608e3f02d1.js` — client bundle
- `assets/server/.next/static/chunks/9365-d1a77d1fb931ad63.js` — zustand stores
- `assets/server/.next/server/chunks/4034.js` — reddit-scraper (all fetch strategies)

Java source is in the Android project (Capacitor-based).

---

*HFY NEKROCODEXXX v250fx13 — APK Recreation Manual*
*Dev: Mahmudul Hossain*
