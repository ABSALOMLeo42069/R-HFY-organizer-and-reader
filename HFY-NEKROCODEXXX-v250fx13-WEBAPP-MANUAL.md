# HFY NEKROCODEXXX v250fx13 — Complete Webapp Recreation Manual

Version: v250fx13 | Dev: Mahmudul Hossain

---

## Table of Contents

1. Introduction
2. Architecture
3. Prerequisites
4. Base Source
5. Change Log (v250fx4 → v250fx13)
6. OAuth Multi-Account Token Rotation (v250fx13)
7. Library & Favorites Persistence (v250fx10–v250fx12)
8. Server-Side Store API (v250fx10)
9. Chapter Persistence (v250fx10)
10. Scores File Recursion Fix (v250fx10)
11. Full Source Code
12. Aggregation
13. Testing
14. Troubleshooting
15. API Reference
16. Configuration
17. Deployment

---

# Chapter 1: Introduction

HFY NEKROCODEXXX is a cyberpunk web application for reading HFY stories from Reddit. It is a Next.js 16 standalone application with 5,393 series and 72,248 chapters hardcoded into the build.

**v250fx13** is the current version. It includes all fixes from v250fx4 through v250fx13, with the most significant change being comprehensive OAuth multi-account token rotation across every fetch path in the application.

---

# Chapter 2: Architecture

A local Node.js server runs on port 3000. A React single-page application is served to the browser. All data is persisted to the server filesystem.

## Reddit Fetch Strategy (9 strategies, all with OAuth rotation)

As of v250fx13, every fetch strategy now uses OAuth token rotation. When multiple Reddit accounts are connected (up to 9), each request rotates to the next account's token in round-robin order, providing up to 5,400 QPM (9 × 600 QPM per account).

1. **OAuth JSON** (Strategy 0, primary) — `oauth.reddit.com/comments/{postId}` with Bearer token from multi-account rotation
2. **Cookie-based fetch** (Strategy -1/0.5) — uses injected Reddit session cookies for authenticated `.json` access
3. **Browser simulation** (Strategy 0.7) — sends full Chrome headers (Sec-Fetch-*, Sec-Ch-Ua, Accept-Encoding) with OAuth token
4. **Direct RSS** (Strategy 1) — `.rss` Atom feed with OAuth Authorization header
5. **CORS proxy pool** (Strategy 2) — 386 public proxy servers with health checking and rotation
6. **Direct .json fallback** (Strategy 3) — last resort `.json` fetch with OAuth token
7. **fetchWithRedDownloader** — alternate JSON fetcher with OAuth token
8. **Wayback Machine** — archived content from web.archive.org
9. **hfy.foundation** — indexed content from the community project

User-Agent rotates between 8 browser strings (Chrome, Firefox, Safari, Edge across Windows/Mac/Linux).

---

# Chapter 3: Prerequisites

- Node.js v22 or later
- Bun (for building from source)
- Next.js 16.2.10
- Zip, Curl

---

# Chapter 4: Base Source

```bash
bun install
bunx next build --webpack
cp -r .next/static .next/standalone/.next/
cp -r public .next/standalone/
```

The build produces a self-contained server in `.next/standalone/` that does not require `npm install` on the target machine.

---

# Chapter 5: Change Log (v250fx4 → v250fx13)

## v250fx4 (base)

- Browser simulation (full Chrome headers for Reddit)
- User-Agent rotation (8 browser UAs)
- Windows EXE standalone window fix (process.execPath instead of \_\_dirname)

## v250fx5–v250fx9

- Font minimum 3px
- Custom series PNG icons
- Merge indicator icons
- Library tab AUTHOR sort removed
- Download chapters auto C-FETCH (saves score + content + marks fetched)
- Merge safety check (all chapters must be c-fetched first)
- Downloaded chapter count always visible on series cards
- Multi-OAuth fix (cookie accounts filtered from token rotation)
- System browser login button
- Real browser backend (puppeteer-core)
- Favorites persistence fix (serverWrite function shadowing bug)
- Chapter count fix (displays real total)
- Store endpoint crash fix
- Browser session routing fix
- C-FETCH REHASH/CONTINUE/STOP fix (batch processing + AbortController)
- C-FETCH batch size slider (1–500 in Settings)
- Rename SAVE button fix (renameStateRef avoids closure staleness)
- aggregate-counts.js crash fix (fetch → http.request)
- Port conflict protection (EADDRINUSE check)
- Browser simulation (complete Chrome header set)
- User-Agent rotation (8 browser UAs)
- Long-press menu button
- CONTINUE fix (chapters.filter instead of per-chapter check)
- Windows EXE .CMD launcher
- TDZ fix (renameStateRef moved after declaration)

## v250fx10

- **Chapter persistence to IndexedDB** — added `useEffect` that saves chapters to IDB whenever they change (C-FETCH, crawl, rename, reorder). Previously only delete and rename persisted; C-FETCH and crawl results were lost on restart.
- **Scores file recursion bug fixed** — `loadAllScoresFromServer` was treating the zustand persist wrapper `{state:{scores:...}}` as a flat map, causing infinite nesting on each save (11+ levels deep). Now properly unwraps the persist wrapper.
- **Library `getItem` made async** — previously returned null immediately without waiting for the server fetch, causing library to appear empty on Android restart. Now awaits the server response.
- **Series scores `getItem` fetches from server** — previously only checked localStorage (cleared on Android restart). Now fetches from server as fallback.
- **Migration for corrupted scores file** — added migration function that unwraps nested recursion and cleans invalid keys. Bumped version to 2.

## v250fx11

- **Star/favorite display on Browse cards** — added gold star indicator on series cards in the Browse tab when a series is favorited.
- **Series detail star button fix** — `toggleFavorite` now auto-adds the series to the library if not already present (previously a no-op).
- **`toggleFavorite` store fix** — the zustand store's `toggleFavorite` function now auto-adds to library with `isFavorite: true` if the series is not in the library.
- **`isFav` reads directly from store** — uses a dedicated zustand selector instead of a stale `libItem` variable.

## v250fx12

- **Star size reduced** — series detail star reduced to 1/3 size (size 7, was 20). Browse card star reduced to size 6.
- **`addToLibrary` merge fix** — `addToLibrary` now merges with existing items instead of overwriting. Previously, when the series detail screen auto-added a series, it would overwrite the existing item with `isFavorite: false`, destroying the user's favorite state on every tab switch.
- **Async `getItem` race condition fixed** — reverted to synchronous `getItem` (localStorage only) to prevent stale server data from overwriting in-memory favorite toggles. Server restore handled via `onRehydrateStorage`.

## v250fx13 (current)

- **Comprehensive OAuth token rotation across ALL fetch paths** — audited every `fetch()` call in `4034.js` (the reddit-scraper server chunk). Found 8 fetch paths that were NOT using OAuth token rotation, even when multiple accounts were connected. All 8 now call `getRedditOAuthToken()` and add `Authorization: Bearer {token}` to the request headers.
- **`getNextMultiToken` token refresh** — `getNextMultiToken()` is now `async` and automatically refreshes expired tokens using the stored `refreshToken` before giving up. Previously, expired tokens were simply skipped, causing the rotation to fall back to the anonymous app-only token (100 QPM) after ~1 hour.
- **`getRedditOAuthToken` awaits `getNextMultiToken`** — updated to properly `await` the now-async `getNextMultiToken()` function.

---

# Chapter 6: OAuth Multi-Account Token Rotation (v250fx13)

## Problem

In v250fx12 and earlier, the `/api/reddit-multi-token` route existed and could store multiple accounts in `globalThis.redditMultiTokens` with round-robin rotation. The client-side code already synced accounts to the server via `syncAccountsToServer()`.

However, the actual fetch functions in `4034.js` (the reddit-scraper) did NOT use the multi-token rotation for most fetch paths. Only Strategy 0 (OAuth via `oauth.reddit.com`) used `fetchWithOAuth`, which called `getRedditOAuthToken`. All other strategies used plain `fetch()` with no `Authorization` header.

This meant that even with 5 accounts logged in, all RSS fetches, wiki JSON fetches, search fetches, browser simulation fetches, and fallback JSON fetches went out with no account token — causing Reddit to rate-limit the server IP.

## Solution

### Fix 1: `getNextMultiToken` now refreshes expired tokens

**Before:**
```javascript
function getNextMultiToken() {
    const store = getMultiTokenStore();
    if (store.size === 0) return null;
    const accounts = Array.from(store.entries()).filter(([, v]) =>
        v.accessToken && Date.now() < v.expiresAt - 5 * 60 * 1000
        && v.clientId !== "cookies" && v.clientId !== "cookie-injection"
    );
    if (accounts.length === 0) return null;
    const idx = (global.redditTokenRotationIndex || 0) % accounts.length;
    global.redditTokenRotationIndex = idx + 1;
    const [, tokenData] = accounts[idx];
    return { token: tokenData.accessToken, username: tokenData.username };
}
```

**After:**
```javascript
async function getNextMultiToken() {
    const store = getMultiTokenStore();
    if (store.size === 0) return null;
    const allAccounts = Array.from(store.entries()).filter(([, v]) =>
        v.accessToken && v.clientId !== "cookies" && v.clientId !== "cookie-injection"
    );
    if (allAccounts.length === 0) return null;
    let liveAccounts = allAccounts.filter(([, v]) => Date.now() < v.expiresAt - 5 * 60 * 1000);
    // If no live accounts, try to refresh expired ones
    if (liveAccounts.length === 0) {
        for (const [id, v] of allAccounts) {
            if (v.refreshToken && v.clientId && v.clientId !== "cookies") {
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
                } catch (e) { /* log and continue */ }
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

### Fix 2: `getRedditOAuthToken` awaits `getNextMultiToken`

```javascript
async function getRedditOAuthToken() {
    const multi = await getNextMultiToken();  // now async
    if (multi) return { token: multi.token, tier: "user", username: multi.username };
    const userToken = getUserToken();
    if (userToken) return { token: userToken, tier: "user" };
    const appToken = await getAppOnlyToken();
    if (appToken) return { token: appToken, tier: "app" };
    return null;
}
```

### Fix 3: All 8 fetch paths now use OAuth rotation

Every `fetch()` call in `4034.js` that was previously making unauthenticated requests now calls `getRedditOAuthToken()` and adds the `Authorization: Bearer {token}` header. The 8 patched paths are:

1. **Wiki series index `.json`** (Strategy 1 in `fetchSeriesIndex`)
2. **Wiki series/{slug} `.json`** (Strategy 1 in `fetchSeriesWikiPage`)
3. **Browser simulation `.json`** (Strategy 0.7 in `fetchPostContent`)
4. **Last resort `.json` fallback** (Strategy 3 in `fetchPostContent`)
5. **Public score fetch `.json`** (score fetch after RSS parse)
6. **Search `.json`** (Strategy 1 in `searchSeries`)
7. **Subreddit RSS feed** (`fetchSubredditFeed`)
8. **`fetchWithRedDownloader`** (alternate JSON fetcher)

Additionally, the RSS chapter fetch (Strategy 1 in `fetchPostContent`) was patched in v250fx12 to use OAuth.

### Pattern used for each fix

```javascript
// Before:
const res = await fetch(url, {
    headers: { "User-Agent": getRandomUA(), Accept: "application/json" },
    signal: AbortSignal.timeout(8000),
    cache: "no-store"
});

// After:
const oauthAuth = await getRedditOAuthToken();
const hdrs = { "User-Agent": getRandomUA(), Accept: "application/json" };
if (oauthAuth) hdrs["Authorization"] = `Bearer ${oauthAuth.token}`;
const res = await fetch(url, {
    headers: hdrs,
    signal: AbortSignal.timeout(8000),
    cache: "no-store"
});
```

## Verification

With 9 accounts synced, the rotation cycles through all 9 in round-robin order:

```
GET 1:  user1 (index 0)
GET 2:  user2 (index 1)
GET 3:  user3 (index 2)
...
GET 9:  user9 (index 8)
GET 10: user1 (index 0)  ← wraps around
```

This provides 9 × 600 QPM = 5,400 QPM instead of 600 QPM from a single account.

---

# Chapter 7: Library & Favorites Persistence (v250fx10–v250fx12)

## `addToLibrary` merge fix (v250fx12)

**Problem:** `addToLibrary` overwrote existing library items with `isFavorite: false` whenever the series detail screen auto-added a series. This destroyed the user's favorite state on every tab switch.

**Fix:** `addToLibrary` now checks if the series already exists in the library. If it does, it merges — preserving `isFavorite`, `readChapters`, `progress`, `lastReadChapterId`, and all other user state. Only metadata fields (`title`, `author`, `coverAccent`, `chapterCount`, `score`) are updated.

```javascript
addToLibrary: (item) => set((s) => {
    const existing = s.items[item.seriesId];
    return {
        items: {
            ...s.items,
            [item.seriesId]: existing ? {
                ...existing,
                title: item.title || existing.title,
                author: item.author || existing.author,
                coverAccent: item.coverAccent || existing.coverAccent,
                chapterCount: item.chapterCount ?? existing.chapterCount,
                score: item.score ?? existing.score,
                // PRESERVE user state
                isFavorite: existing.isFavorite,
                lastReadChapterId: existing.lastReadChapterId,
                lastReadChapterTitle: existing.lastReadChapterTitle,
                lastReadPosition: existing.lastReadPosition,
                lastReadTimestamp: existing.lastReadTimestamp,
                progress: existing.progress,
                readChapters: existing.readChapters,
                dateAdded: existing.dateAdded
            } : {
                // New item — create fresh
                seriesId: item.seriesId,
                title: item.title,
                author: item.author,
                coverAccent: item.coverAccent || "#00FFF0",
                isFavorite: item.isFavorite ?? false,
                // ... other defaults
            }
        }
    };
})
```

## `toggleFavorite` auto-add (v250fx11)

**Problem:** `toggleFavorite` was a no-op when the series was not in the library.

**Fix:** If the series is not in the library, `toggleFavorite` now auto-adds it with `isFavorite: true`.

## `isFav` direct store selector (v250fx11)

**Problem:** `isFav` read from a stale `libItem` variable that didn't update when the store changed.

**Fix:** `isFav` now uses a dedicated zustand selector that reads directly from the store:

```javascript
const isFav = useLibrary((s) => seriesId ? !!(s.items[seriesId]?.isFavorite) : false);
```

## Synchronous `getItem` (v250fx12)

**Problem:** The async `getItem` in the library store's persist middleware caused a race condition — stale server data would overwrite in-memory favorite toggles.

**Fix:** `getItem` is now synchronous (localStorage only). Server restore is handled separately via `onRehydrateStorage`, which only fetches from the server when localStorage is empty (Android restart scenario).

---

# Chapter 8: Server-Side Store API (v250fx10)

The webapp's `server.js` includes a proxy server that intercepts `/api/store/:key` requests and persists them to the filesystem. This provides durable storage that survives Android restarts (where localStorage is cleared).

**Data directory:** `{HFY_DATA_DIR}/app-store/` (or `./data/app-store/` on desktop)

**Endpoints:**
- `GET /api/store/:key` — returns `{ ok: true, data: ... }` or 404
- `POST /api/store/:key` — writes JSON body to `{key}.json`
- `DELETE /api/store/:key` — deletes the file

**Atomic writes:** writes to a `.tmp.{pid}` file first, then renames to the final filename to prevent corruption.

---

# Chapter 9: Chapter Persistence (v250fx10)

A `useEffect` in `SeriesDetailScreen` saves chapters to IndexedDB whenever they change:

```javascript
useEffect(() => {
    if (!seriesId || chapters.length === 0) return;
    const timer = setTimeout(() => {
        (async () => {
            const { idbGet, idbSet } = await import(/* chunk */ 3375);
            const cacheKey = `hfy-series-cache-v3-${seriesId}`;
            const existing = await idbGet(cacheKey);
            const chaptersToSave = chapters.map((c, i) => ({
                postId: c.id, id: c.id, title: c.title,
                url: c.redditPostUrl, redditPostUrl: c.redditPostUrl,
                author: c.author || "unknown",
                createdUtc: c.createdUtc || 0,
                score: c.score || 0,
                wordCount: c.wordCount || 0,
                discovered: c.discovered,
                orderIndex: i
            }));
            await idbSet(cacheKey, {
                ...(existing || {}),
                id: seriesId,
                title: series?.title || existing?.title || seriesId,
                chapters: chaptersToSave,
                cachedAt: Date.now()
            });
        })();
    }, 2000); // 2s debounce
    return () => clearTimeout(timer);
}, [chapters, seriesId, series?.title]);
```

This ensures C-FETCH results, crawled chapters, renamed chapters, and reordered chapters all survive app restart.

---

# Chapter 10: Scores File Recursion Fix (v250fx10)

**Problem:** `loadAllScoresFromServer` treated the zustand persist wrapper `{state:{scores:{...}}}` as a flat map. On each save, it would nest the wrapper inside itself, causing 11+ levels of recursion and corrupting the scores file.

**Fix:** The function now unwraps the persist wrapper before iterating:

```javascript
let scoresData = legacyData.data;
let unwrapDepth = 0;
while (scoresData && scoresData.state && scoresData.state.scores && unwrapDepth < 20) {
    scoresData = scoresData.state.scores;
    unwrapDepth++;
}
const finalData = scoresData || {};
```

A migration function was also added to the `useSeriesScores` store to unwrap any existing corrupted files on load.

---

# Chapter 11: Full Source Code

The full source code is included in every distribution ZIP. Key files:

- `server.js` — Next.js standalone server with proxy and store API
- `aggregate-counts.js` — periodic aggregation of downloaded/chapter counts
- `.next/static/chunks/app/page-94aba2608e3f02d1.js` — main client bundle (2.4MB)
- `.next/static/chunks/9365-d1a77d1fb931ad63.js` — zustand stores
- `.next/server/chunks/4034.js` — reddit-scraper with all fetch strategies

---

# Chapter 12: Aggregation

`aggregate-counts.js` runs on startup and every 120 seconds:

- Scans `app-store/hfy-downloaded-*.json` for downloaded chapter counts
- Scans `user-chapters/*.json` for real chapter counts
- POSTs to `/api/store/hfy-downloaded-counts` and `/api/store/hfy-chapter-counts`

Uses `http.request()` instead of `fetch()` because `fetch` is not available in the Android Node.js runtime.

---

# Chapter 13: Testing

```bash
# Start server
PORT=3000 HFY_DATA_DIR=./data node server.js

# Test store API
curl -X POST http://localhost:3000/api/store/test -H 'Content-Type: application/json' -d '{"hello":"world"}'
curl http://localhost:3000/api/store/test

# Test multi-token rotation
curl -X POST http://localhost:3000/api/reddit-multi-token \
    -H 'Content-Type: application/json' \
    -d '{"accounts":[{"id":"u1","username":"user1","accessToken":"TOKEN_A","refreshToken":"r1","expiresAt":9999999999999,"clientId":"c1"},{"id":"u2","username":"user2","accessToken":"TOKEN_B","refreshToken":"r2","expiresAt":9999999999999,"clientId":"c2"}]}'

# Should rotate: TOKEN_A → TOKEN_B → TOKEN_A → ...
curl http://localhost:3000/api/reddit-multi-token
curl http://localhost:3000/api/reddit-multi-token
curl http://localhost:3000/api/reddit-multi-token
```

---

# Chapter 14: Troubleshooting

**Rate limited even with multiple accounts:** Ensure all accounts are synced via Settings → Reddit Connection. Check `/api/reddit-multi-token` GET returns `totalAccounts` > 1. All fetch paths now use OAuth rotation as of v250fx13.

**Favorites lost on tab switch:** Fixed in v250fx12. `addToLibrary` now merges instead of overwriting.

**Star shows dim in series detail:** Fixed in v250fx11. `isFav` reads directly from the store.

**Chapters lost on restart:** Fixed in v250fx10. Chapters now persist to IndexedDB.

**Scores file corrupted (recursive nesting):** Fixed in v250fx10. Migration unwraps corrupted files.

**Server crashes with EADDRINUSE:** Port conflict protection added. Server waits 5s and retries.

---

# Chapter 15: API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/store/[key]` | GET/POST/DELETE | Generic key-value store (filesystem-backed) |
| `/api/store-scores-all` | GET | All series scores |
| `/api/reddit-multi-token` | GET/POST | Multi-account token rotation |
| `/api/reddit-auth` | GET | Reddit OAuth PKCE initiation |
| `/api/reddit-token` | POST | OAuth token exchange |
| `/api/reddit-user-token` | POST | Single user token storage |
| `/api/reddit-cookies` | GET/POST | Cookie management |
| `/api/proxy-browser` | GET/POST/DELETE | HTTP proxy + browser session |
| `/api/proxy-stats` | GET | Proxy statistics |
| `/api/series` | GET | Series list |
| `/api/series/[slug]` | GET | Series detail |
| `/api/series/[slug]/chapters` | GET/POST/DELETE | Chapter CRUD |
| `/api/series/[slug]/fetch-cache` | POST | C-FETCH (NDJSON stream) |
| `/api/series/[slug]/continue-fetch` | POST | Continue crawl |
| `/api/series/[slug]/fetch-missing` | POST | Fetch missing chapters |
| `/api/series/[slug]/detect-gaps` | GET | Detect numbering gaps |
| `/api/series/[slug]/reparse` | POST | Reparse wiki page |
| `/api/series/[slug]/add-chapter` | POST | Add chapter manually |
| `/api/chapter/[id]` | GET | Chapter content |
| `/api/crawl` | POST | RSS crawl |
| `/api/search` | GET | Search series |
| `/api/search-series` | GET | Search series index |
| `/api/download` | POST | Download chapters |
| `/api/auto-discover` | POST/GET | Auto-discover pipeline |
| `/api/hfy-foundation-search` | GET | hfy.foundation search |
| `/api/live-wiki` | GET | Live wiki fetch |
| `/api/mass-import` | POST | Mass import chapters |
| `/api/purge-cache` | POST | Purge cache |
| `/api/version` | GET | Version info |
| `/api/wiki-sources` | GET | Wiki source list |

---

# Chapter 16: Configuration

**Environment variables:**
- `PORT` — server port (default 3000)
- `HOSTNAME` — bind address (default 0.0.0.0)
- `HFY_DATA_DIR` — data directory for persistent storage
- `NODE_ENV` — set to `production`
- `KEEP_ALIVE_TIMEOUT` — HTTP keep-alive timeout

**C-FETCH batch size:** Configurable in Settings (1–500, default 5). Controls how many chapters are fetched per batch in REHASH/CONTINUE.

---

# Chapter 17: Deployment

```bash
# Extract webapp ZIP
unzip HFY-NEKROCODEXXX-v250fx13-webapp.zip -d hfy
cd hfy

# Start server
PORT=3000 HFY_DATA_DIR=./data NODE_ENV=production node server.js

# Open in browser
open http://localhost:3000
```

No `npm install` or build step required. All dependencies are bundled.

---

*HFY NEKROCODEXXX v250fx13 — Complete Webapp Recreation Manual*
*Dev: Mahmudul Hossain*
