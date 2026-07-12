# HFY NEKROCODEXXX v250fx13 — Windows Standalone Recreation Manual

Version: v250fx13 | Dev: Mahmudul Hossain

---

## Chapters

1. Introduction
2. Architecture
3. Prerequisites
4. Package Structure
5. CMD Launcher
6. start.bat Fallback
7. Standalone Window
8. Server-Side Store API
9. OAuth Multi-Account Token Rotation (v250fx13)
10. Library & Favorites Persistence (v250fx10–v250fx12)
11. Chapter Persistence (v250fx10)
12. Port Conflict Protection
13. Browser Simulation & UA Rotation
14. Keepalive
15. README.txt
16. Testing
17. Troubleshooting
18. Build Script
19. File Sizes
20. Puppeteer Dependencies
21. Source Code

---

# Chapter 1: Introduction

The Windows package runs the Next.js server via `node.exe` and opens a standalone Chromium window (Chrome or Edge `--app` mode). No installation required — just extract and run.

**v250fx13** adds comprehensive OAuth multi-account token rotation across all fetch paths, library/favorites persistence fixes, and chapter persistence to IndexedDB.

---

# Chapter 2: Architecture

```
HFY-NEKROCODEXXX.cmd (launcher)
    → node.exe server\server.js (Next.js + proxy + store API)
        → Chrome/Edge --app=http://localhost:3000 (standalone window)
```

The server runs on port 3000. A proxy intercepts `/api/store/*` requests for filesystem persistence.

---

# Chapter 3: Prerequisites

**Build:** Node.js v22+, zip, curl
**Runtime:** Windows 10/11 64-bit, 500MB disk, 4GB RAM, Chrome or Edge (for standalone window)

---

# Chapter 4: Package Structure

```
HFY-NEKROCODEXXX-v250fx13-Windows/
├── HFY-NEKROCODEXXX.cmd    (launcher — starts server, opens window)
├── start.bat                (fallback — runs server only)
├── README.txt               (quick start)
├── server/
│   ├── server.js            (Next.js + proxy + store API)
│   ├── aggregate-counts.js  (periodic aggregation)
│   ├── package.json
│   ├── .env
│   ├── .next/
│   │   ├── BUILD_ID
│   │   ├── build-manifest.json
│   │   ├── required-server-files.json
│   │   ├── static/
│   │   │   └── chunks/
│   │   │       ├── app/
│   │   │       │   └── page-94aba2608e3f02d1.js  (client bundle, 2.4MB)
│   │   │       └── 9365-d1a77d1fb931ad63.js       (zustand stores)
│   │   └── server/
│   │       └── chunks/
│   │           └── 4034.js                         (reddit-scraper)
│   ├── public/             (icons, images)
│   └── node_modules/       (next, react, puppeteer-core, etc.)
```

**Note:** Unlike earlier versions that used SEA (Single Executable Application), v250fx13 uses a `.cmd` launcher with `node.exe`. This is more reliable and doesn't require SEA blob injection.

---

# Chapter 5: CMD Launcher

`HFY-NEKROCODEXXX.cmd`:

```batch
@echo off
title HFY NEKROCODEXXX v250fx13
cd /d "%~dp0"
echo ========================================
echo  HFY NEKROCODEXXX v250fx13
echo  Starting server...
echo ========================================
set NODE_ENV=production&set PORT=3000&set HOSTNAME=127.0.0.1
start /b cmd /c "timeout /t 10 /nobreak >nul && start chrome --app=http://localhost:3000 --window-size=1280,900 2>nul || start msedge --app=http://localhost:3000 --window-size=1280,900 2>nul || start http://localhost:3000"
node server\server.js
pause
```

**How it works:**
1. Sets environment variables (NODE_ENV, PORT, HOSTNAME)
2. Starts a background process that waits 10 seconds, then opens Chrome/Edge in `--app` mode
3. Starts the Node.js server in the foreground
4. The console window stays open — close it to stop the server

---

# Chapter 6: start.bat Fallback

`start.bat`:

```batch
@echo off
cd /d "%~dp0"
set NODE_ENV=production&set PORT=3000
node server\server.js
pause
```

Use this if the CMD launcher doesn't work (antivirus blocking, SmartScreen, etc.). It runs the server directly without opening a browser window — the user opens `http://localhost:3000` manually.

---

# Chapter 7: Standalone Window

The `--app` flag opens a Chromium window with no address bar, no tabs, no bookmarks — just the app content. Like a native application.

```
chrome --app=http://localhost:3000 --window-size=1280,900
```

If Chrome is not installed, it tries Edge:
```
msedge --app=http://localhost:3000 --window-size=1280,900
```

If neither is available, it falls back to the system default browser:
```
start http://localhost:3000
```

---

# Chapter 8: Server-Side Store API

The `server.js` file includes a proxy server that intercepts `/api/store/:key` requests:

```javascript
function handleStoreAPI(req, res) {
    const url = new URL(req.url, 'http://' + (req.headers.host || 'localhost'));
    const match = url.pathname.match(/^\/api\/store\/([^\/]+)$/);
    if (!match) return false;
    const key = decodeURIComponent(match[1]);
    ensureDir();
    if (req.method === 'GET') {
        // Read from {HFY_DATA_DIR}/app-store/{key}.json
    }
    if (req.method === 'POST') {
        // Write to {key}.json (atomic: write .tmp then rename)
    }
    if (req.method === 'DELETE') {
        // Delete {key}.json
    }
    return true;
}
```

**Data directory:** `{HFY_DATA_DIR}/app-store/` or `./data/app-store/` if `HFY_DATA_DIR` is not set.

On Windows, data is stored in `server/data/app-store/` by default.

---

# Chapter 9: OAuth Multi-Account Token Rotation (v250fx13)

## Problem

In v250fx12 and earlier, only the primary OAuth fetch strategy used multi-account token rotation. All other fetch paths (RSS, wiki JSON, search, browser simulation, fallback JSON, score fetch, subreddit feed, RedDownloader) used plain `fetch()` with no `Authorization` header.

This meant that even with 9 accounts connected, most requests went out unauthenticated, causing Reddit to rate-limit the server IP.

## Solution

### `getNextMultiToken()` now refreshes expired tokens

The function is now `async` and automatically attempts to refresh expired tokens using the stored `refreshToken`:

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
        // Refresh expired tokens
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

### All 8 fetch paths now use OAuth rotation

Every `fetch()` call in `4034.js` that was previously unauthenticated now calls `getRedditOAuthToken()` and adds the `Authorization: Bearer {token}` header:

1. Wiki series index `.json`
2. Wiki series/{slug} `.json`
3. Browser simulation `.json`
4. Last resort `.json` fallback
5. Public score fetch `.json`
6. Search `.json`
7. Subreddit RSS feed
8. `fetchWithRedDownloader`

### Pattern

```javascript
const oauthAuth = await getRedditOAuthToken();
const hdrs = { "User-Agent": getRandomUA(), Accept: "application/json" };
if (oauthAuth) hdrs["Authorization"] = `Bearer ${oauthAuth.token}`;
const res = await fetch(url, { headers: hdrs, ... });
```

## Result

With 9 accounts: 9 × 600 QPM = 5,400 QPM (was 600 QPM with single account).

---

# Chapter 10: Library & Favorites Persistence (v250fx10–v250fx12)

## `addToLibrary` merge fix (v250fx12)

`addToLibrary` now merges with existing items instead of overwriting. This prevents the series detail screen's auto-add from destroying the user's favorite state:

```javascript
addToLibrary: (item) => set((s) => {
    const existing = s.items[item.seriesId];
    return {
        items: {
            ...s.items,
            [item.seriesId]: existing ? {
                ...existing,
                title: item.title || existing.title,
                // ... update metadata
                // PRESERVE user state:
                isFavorite: existing.isFavorite,
                readChapters: existing.readChapters,
                progress: existing.progress,
                // ... etc
            } : { /* new item */ }
        }
    };
})
```

## `toggleFavorite` auto-add (v250fx11)

If the series is not in the library, `toggleFavorite` now auto-adds it with `isFavorite: true`.

## `isFav` direct store selector (v250fx11)

Uses a dedicated zustand selector: `useLibrary((s) => !!(s.items[seriesId]?.isFavorite))`

## Synchronous `getItem` (v250fx12)

The persist middleware's `getItem` is synchronous (localStorage only). Server restore is handled via `onRehydrateStorage` to prevent race conditions.

---

# Chapter 11: Chapter Persistence (v250fx10)

A `useEffect` in `SeriesDetailScreen` saves chapters to IndexedDB whenever they change (2-second debounce). This ensures C-FETCH results, crawled chapters, renamed chapters, and reordered chapters survive app restart.

---

# Chapter 12: Port Conflict Protection

```javascript
// server.js checks for EADDRINUSE before starting
const testServer = net.createServer();
testServer.once('error', (e) => {
    if (e.code === 'EADDRINUSE') {
        console.log('Port 3000 in use, waiting 5s...');
        setTimeout(() => process.exit(0), 5000);
    }
});
testServer.once('listening', () => {
    testServer.close();
    startNextServer();
});
```

This prevents the crash-restart loop that killed earlier versions when the old process hadn't fully released the port.

---

# Chapter 13: Browser Simulation & UA Rotation

## Browser Simulation

Full Chrome browser headers are sent to make Reddit think requests come from a real browser:

```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,...
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Sec-Ch-Ua: "Google Chrome";v="131", "Chromium";v="131"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Upgrade-Insecure-Requests: 1
```

As of v250fx13, this also includes the `Authorization: Bearer {token}` header from OAuth rotation.

## UA Rotation

8 browser User-Agent strings are rotated:
- Chrome on Windows, macOS, Linux
- Firefox on Windows, macOS, Linux
- Safari on macOS
- Edge on Windows

---

# Chapter 14: Keepalive

The CMD launcher starts the server in the foreground. The console window must stay open for the server to run. Closing the console window stops the server.

For background operation, use `start /b` in a separate batch file or Windows Task Scheduler.

---

# Chapter 15: README.txt

```text
HFY NEKROCODEXXX v250fx13
========================

REQUIRES: Node.js v18+ installed (https://nodejs.org)

QUICK START:
1. Double-click HFY-NEKROCODEXXX.cmd
2. Wait 10-30 seconds
3. Browser opens automatically at http://localhost:3000

ALTERNATIVE:
1. Double-click start.bat
2. Open http://localhost:3000 in your browser

LEGAL NOTICE:
This application is for personal reading only. Downloaded ebooks
may NOT be distributed, shared, republished, or sold. All story
content is the property of the original r/HFY authors.
```

---

# Chapter 16: Testing

```bash
# Start server
cd server
set PORT=3000
set HFY_DATA_DIR=.\data
set NODE_ENV=production
node server.js

# Test store API
curl -X POST http://localhost:3000/api/store/test -H "Content-Type: application/json" -d "{\"hello\":\"world\"}"
curl http://localhost:3000/api/store/test

# Test multi-token rotation
curl -X POST http://localhost:3000/api/reddit-multi-token -H "Content-Type: application/json" -d "{\"accounts\":[{\"id\":\"u1\",\"username\":\"user1\",\"accessToken\":\"TOKEN_A\",\"refreshToken\":\"r1\",\"expiresAt\":9999999999999,\"clientId\":\"c1\"},{\"id\":\"u2\",\"username\":\"user2\",\"accessToken\":\"TOKEN_B\",\"refreshToken\":\"r2\",\"expiresAt\":9999999999999,\"clientId\":\"c2\"}]}"

# Should rotate: TOKEN_A → TOKEN_B → TOKEN_A
curl http://localhost:3000/api/reddit-multi-token
curl http://localhost:3000/api/reddit-multi-token
curl http://localhost:3000/api/reddit-multi-token
```

---

# Chapter 17: Troubleshooting

**CMD launcher doesn't open:**
- Antivirus/SmartScreen blocking: click "More info" → "Run anyway"
- Use `start.bat` as fallback
- Ensure `server\` folder is in the same directory as the CMD file

**No standalone window:**
- Install Chrome or Edge (the `--app` flag requires Chromium-based browser)
- Falls back to system default browser automatically

**Rate limited even with multiple accounts:**
- Fixed in v250fx13. All fetch paths now use OAuth rotation.
- Ensure accounts are synced: Settings → Reddit Connection → check account count

**Favorites lost on tab switch:**
- Fixed in v250fx12. `addToLibrary` merges instead of overwriting.

**Chapters lost on restart:**
- Fixed in v250fx10. Chapters persist to IndexedDB.

**"EADDRINUSE" error:**
- Another process is using port 3000. Wait 5 seconds for the old process to exit, or change the PORT environment variable.

**Puppeteer browser login doesn't work:**
- Install Chrome or Edge
- Puppeteer-core launches the system browser (doesn't download its own)

---

# Chapter 18: Build Script

```bash
#!/bin/bash
set -e

# 1. Build webapp
cd /path/to/src
bun install
bunx next build --webpack
cp -r .next/static .next/standalone/.next/
cp -r public .next/standalone/

# 2. Create Windows package directory
mkdir -p win-package/server
cp -r .next/standalone/* win-package/server/
cp package.json win-package/server/

# 3. Create CMD launcher
cat > win-package/HFY-NEKROCODEXXX.cmd << 'EOF'
@echo off
title HFY NEKROCODEXXX v250fx13
cd /d "%~dp0"
echo ========================================
echo  HFY NEKROCODEXXX v250fx13
echo  Starting server...
echo ========================================
set NODE_ENV=production&set PORT=3000&set HOSTNAME=127.0.0.1
start /b cmd /c "timeout /t 10 /nobreak >nul && start chrome --app=http://localhost:3000 --window-size=1280,900 2>nul || start msedge --app=http://localhost:3000 --window-size=1280,900 2>nul || start http://localhost:3000"
node server\server.js
pause
EOF

# 4. Create start.bat
cat > win-package/start.bat << 'EOF'
@echo off
cd /d "%~dp0"
set NODE_ENV=production&set PORT=3000
node server\server.js
pause
EOF

# 5. Create README.txt
cat > win-package/README.txt << 'EOF'
HFY NEKROCODEXXX v250fx13
========================
REQUIRES: Node.js v18+ (https://nodejs.org)
QUICK START: Double-click HFY-NEKROCODEXXX.cmd
EOF

# 6. Download node.exe for Windows
curl -L -o win-package/node.exe https://nodejs.org/dist/v22.0.0/win-x64/node.exe

# 7. Create ZIP
cd win-package
zip -r ../HFY-NEKROCODEXXX-v250fx13-Windows.zip . -x "*/data/*"
```

---

# Chapter 19: File Sizes

| Component | Size |
|-----------|------|
| ZIP total | ~45 MB (without node.exe) or ~87 MB (with node.exe) |
| server/ (webapp) | ~57 MB |
| node.exe (Windows) | ~80 MB |
| HFY-NEKROCODEXXX.cmd | ~500 bytes |
| start.bat | ~90 bytes |

---

# Chapter 20: Puppeteer Dependencies

The webapp uses `puppeteer-core` for real browser login (capturing Reddit session cookies). Puppeteer-core does NOT download its own browser — it uses the system-installed Chrome or Edge.

**Requirements:**
- Chrome 109+ or Edge 109+ installed on the system
- The browser must be in the default installation path

If the browser is not found, the real browser login option will not work, but all other login methods (OAuth, direct login, cookie injection) still function.

---

# Chapter 21: Source Code

The full source code is included in the package at `server/`. Key files:

- `server/server.js` — Next.js server with proxy and store API
- `server/aggregate-counts.js` — periodic aggregation
- `server/.next/static/chunks/app/page-94aba2608e3f02d1.js` — client bundle (2.4MB)
- `server/.next/static/chunks/9365-d1a77d1fb931ad63.js` — zustand stores
- `server/.next/server/chunks/4034.js` — reddit-scraper (all fetch strategies)

To modify client code, edit the page chunk directly. It is bundled but not minified, so it remains readable.

To modify server code, edit route files in `.next/server/app/api/` or the chunks in `.next/server/chunks/`.

---

*HFY NEKROCODEXXX v250fx13 — Windows Standalone Recreation Manual*
*Dev: Mahmudul Hossain*
