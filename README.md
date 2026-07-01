# HFY NEKROCODEXXX — r/HFY Story Organizer & Reader (Alpha v0.45)

**⚠️ This is alpha (v0.45). Fork it, edit it, use it any way you like, but eemember, this is not a downloader. Expect bugs, missing features, and breaking changes.**

A cyberpunk-themed web application for organizing and reading HFY (Humanity, Fuck Yeah!) stories from Reddit's r/HFY subreddit. 

**This is a reader and organizer, NOT a downloader.** Chapter text content is cached in-browser for offline reading during your session only — it is erased on every restart. Chapter links and series data persist, but the actual story text does not.

---

## What Is This?

HFY NEKROCODEXXX is a self-hosted web application that indexes the entire r/HFY story archive and provides a clean, organized interface for reading stories directly from Reddit.

### Key principle: Reading = Supporting

Every time you read a chapter through this app, the app fetches the chapter content directly from Reddit's servers. This means:

- **Reddit sees your visit** — Each chapter read counts as a view on the original Reddit post
- **Upvote buttons work** — Each chapter has upvote and downvote buttons that link directly to the Reddit post, making it easy to upvote the stories you enjoy
- **Authors get credit** — Unlike offline readers, this app ensures authors get the engagement they deserve

The app shows upvote counts for each chapter (fetched when you read or interact with the chapter), and the total series upvote count is displayed in the series header. Upvote counts are erased on restart and re-fetched as you read — keeping the data fresh.

### Features

- **Browse 5,533 series** with sorting by score, chapter count, word count, and alphabetical order
- **Read chapters** in a full-screen reader with 22 themes, 27 fonts, infinite scroll, and automatic read tracking
- **Upvote integration** — Upvote/downvote buttons on every chapter link directly to Reddit, and upvote counts are displayed per-chapter and as a series total
- **Chapter discovery** via 8 methods: RSS crawling, Wayback Machine, HFY Foundation, Reddit JSON API, OAuth crawl, WebSocket parser, next-chapter link following, and Reddit search
- **Wiki link indicators** showing which sources have indexed each series (r/HFY wiki, HFY Foundation, r/Sexyspacebabes)
- **Library** with favorites, reading progress tracking, and resume-reading
- **Custom series adder** — Add your own series with chapters via Reddit URLs
- **Chapter management** — Sort by chapter number, reorder, detect gaps, search Reddit for missing chapters, delete chapters you've added
- **PWA installable** on any device
- **Android APK** with embedded Node.js server

### What gets erased on restart

- ❌ Chapter text content (story text cached in browser)
- ❌ Chapter upvote counts (re-fetched as you read)
- ❌ Cached chapter content in IndexedDB

### What persists across restarts

- ✅ Series list (5,533 baked-in series)
- ✅ Chapter links (titles + Reddit URLs)
- ✅ Library (favorites, reading progress)
- ✅ Read tracking (which chapters you've read)
- ✅ Settings (themes, fonts, background images)
- ✅ URL overrides (custom wiki URLs)
- ✅ Custom series you've added

### Important: This Is NOT a Downloader

This application does NOT include:

- ❌ Chapter downloading (TXT/EPUB/PDF)
- ❌ File export or merge functionality
- ❌ Bulk download queues
- ❌ Save-to-device features
- ❌ Any downloader integration

**If you are considering forking this project:** Please do NOT add downloading, exporting, or file-saving features. The r/HFY community relies on Reddit engagement (upvotes, comments, views) to sustain its writers. Tools that extract content from Reddit reduce engagement and directly harm the community by discouraging authors from continuing their stories. Adding download features to this app would cause **grave and severe harm** to the HFY community.

---

## How It Was Built

### Web Application

The webapp is built with Next.js 16 (standalone output mode) using TypeScript, React 19, and Tailwind CSS 4. The UI uses a custom cyberpunk design system with:

- Zustand for state management (settings, library, navigation, URL overrides)
- Framer Motion for animations
- shadcn/ui (new-york style) as the component foundation
- IndexedDB for client-side caching of chapter content (erased on restart)
- Prisma + SQLite for server-side user data

The series data (5,533 series, 72,248 chapters) is baked into 22 TypeScript chunk files totaling ~14MB, parsed from CSV/JSON exports of the r/HFY wiki and HFY Foundation index. Each series entry includes title, author, chapter list (title + Reddit URL + post ID), and wiki URLs for r/HFY, HFY Foundation, and r/Sexyspacebabes.

The Reddit integration uses a tiered fallback system:
1. User OAuth (600 QPM, single account)
2. App-only OAuth (100 QPM, anonymous)
3. RedDownloader (authless .json API)
4. CORS proxy pool (386 proxies)
5. Baked data (always available, no network needed)

### Android APK

The APK embeds a full Node.js runtime (Termux arm64 build) inside the app package. On launch, the app extracts the Node.js binary, shared libraries, and the Next.js server from APK assets, starts the server on 127.0.0.1:3000, and loads the webapp in a Capacitor WebView.

Key details: Node.js packaged as libnode.so (W^X bypass), SQLite compiled from source with FTS5/SESSION extensions (16KB alignment for Android 15), foreground service with wake lock, splash screen with no-cors polling, `allowNavigation` for localhost WebView access.

---

## How to Install and Use the Webapp

### Prerequisites

- Node.js 18+ (or Bun 1.3+)
- 4GB RAM minimum (8GB recommended for building)
- 2GB disk space for node_modules + build output

### Windows

1. **Install Node.js 18+** from [nodejs.org](https://nodejs.org) — choose the "LTS" version
2. **Download the source code** (ZIP file from GitHub) and extract it to a folder, e.g., `C:\hfy-nekrocodexxx`
3. Open **PowerShell** (press `Win + X`, then select "Windows PowerShell" or "Terminal")
4. Navigate to the project folder:
   ```powershell
   cd C:\hfy-nekrocodexxx
   ```
5. Install dependencies:
   ```powershell
   npm install
   ```
   This will take 2-5 minutes depending on your internet speed.
6. Build the production server:
   ```powershell
   npm run build
   ```
   This will take 3-10 minutes depending on your CPU.
7. Start the server:
   ```powershell
   npm run start
   ```
8. Open your browser to **http://localhost:3000**
9. The app is now running. Bookmark the URL for easy access.

**To run in development mode** (faster startup, hot reload, but slower page loads):
```powershell
npm run dev
```
Then open **http://localhost:3000** in your browser.

**To stop the server:** Press `Ctrl + C` in the PowerShell window.

**Note:** Chapter text content is cached during your session but erased when you restart the server. You'll need to re-read chapters to cache them again.

### Linux

1. **Install Node.js 18+** using your package manager:
   ```bash
   # Ubuntu/Debian:
   curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt install -y nodejs
   
   # Or using nvm (recommended):
   curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
   source ~/.bashrc
   nvm install 18
   ```
   
   Alternatively, install [Bun](https://bun.sh) (faster):
   ```bash
   curl -fsSL https://bun.sh/install | bash
   source ~/.bashrc
   ```

2. **Download and extract the source code:**
   ```bash
   # If you downloaded a ZIP:
   unzip hfy-nekrocodexxx.zip
   cd hfy-nekrocodexxx
   
   # Or clone from GitHub:
   git clone https://github.com/YOUR_USERNAME/hfy-nekrocodexxx.git
   cd hfy-nekrocodexxx
   ```

3. Install dependencies:
   ```bash
   npm install        # or: bun install
   ```

4. Build the production server:
   ```bash
   npm run build      # or: bun run build
   ```

5. Start the server:
   ```bash
   npm run start      # or: bun run start
   ```

6. Open your browser to **http://localhost:3000**

**To run in development mode:**
```bash
npm run dev          # or: bun run dev
```

### Android (via Termux — no APK needed)

You can run the full webapp on Android using Termux:

1. Install **Termux** from [F-Droid](https://f-droid.org/packages/com.termux/) (do NOT use the Play Store version — it's outdated)
2. Open Termux and run:
   ```bash
   pkg update && pkg install -y nodejs git
   git clone https://github.com/YOUR_USERNAME/hfy-nekrocodexxx.git
   cd hfy-nekrocodexxx
   npm install
   npm run build
   npm run start
   ```
3. Open your phone's browser to **http://localhost:3000**
4. For a full-screen experience, open Chrome → menu → "Add to Home Screen"

### Android (APK)

If you have the pre-built APK:

1. **Enable "Install from unknown sources"** in Android Settings → Security
2. **Install the APK** (file manager → tap the .apk file)
3. **Open the app** — it will start the embedded server and load automatically
4. The app works offline for browsing series lists, but chapter reading requires internet (content is fetched from Reddit on each read)

---

## Features

### Browsing
- Sort by: Newest, Score, Chapter Count, Word Count, A-Z
- Search by series name
- Wiki indicator icons on each card (hardcoded, HFY Foundation, r/HFY wiki, SSB wiki, fetched live)
- Long-press a card to add/remove from library
- Custom series adder button (icon in sort bar)

### Reading
- Infinite scroll reader with automatic chapter preloading
- 22 reader themes (void dark, blood red, cyber blue, terminal green, etc.)
- 27 font options (serif, cyberpunk display, monospace, custom font URL)
- Adjustable font size, line height, and margins
- Automatic read tracking (chapters marked as read when scrolled past 80%)
- Swipe-to-go-back gesture (touchscreen)
- Read progress shown on library cards
- Chapter content fetched fresh from Reddit on each read (cached for session only)

### Upvote Integration
- Upvote count displayed to the left of upvote/downvote buttons on each chapter
- Upvote/downvote buttons (18px) link directly to the Reddit post
- Total series upvote count shown in series header (sum of all chapter scores)
- Scores fetched when reading or interacting with chapters
- Scores erased on restart and re-fetched as you read

### Chapter Discovery (8 methods)
1. **Cache** — Load from IndexedDB (instant, but empty on first run)
2. **Wayback Machine** — Sync from archive.org snapshots
3. **HFY Foundation** — Search community index
4. **Wayback Crawl** — Crawl archived Reddit pages
5. **Reddit JSON API** — Crawl via JSON method
6. **OAuth Crawl** — Authenticated crawl
7. **WebSocket Parser** — Real-time chapter streaming
8. **Chapter Discovery Engine** — Follow "next chapter" links via RSS

### Chapter Management
- **GAPS** — Detect missing chapter numbers in the sequence
- **SORT** — Sort chapters by extracted number (Part N, Chapter N) while keeping unnumbered chapters in place
- **REORDER** — Move chapters up/down or type a position number to jump
- **DELETE** — Long-press a chapter to delete it from the series (doesn't affect the Reddit post)
- **Reddit Chapter Search** — Search r/HFY for chapters by name, filter duplicates, sort by name or number
- **URL Overrides** — Long-press any wiki icon to edit its URL

### Reddit Integration
- **Quick Connect** — Anonymous app-only OAuth (100 QPM, no login needed)
- **Single Account Login** — Connect one Reddit account for 600 QPM
- **In-App Browser** — Browse Reddit logged-in via proxy

### Customization
- 25 app themes
- 23 built-in background images with rotation
- Custom background image upload (multiple images supported)
- Scanline overlay, glitch effect, glow intensity controls
- PWA installable with custom app icon

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | Next.js 16.1.3 (standalone output) |
| Language | TypeScript 5 |
| UI Library | React 19 |
| Styling | Tailwind CSS 4 |
| Components | shadcn/ui (new-york) |
| State | Zustand 5 (persisted) |
| Animation | Framer Motion 12 |
| Database | Prisma + SQLite |
| Reddit | RSS scraping + OAuth 2.0 PKCE |
| Mobile | Capacitor 8 |
| Package Manager | Bun / npm |
| Icons | lucide-react + custom PNG icons |

---

## Project Structure

```
hfy-nekrocodexxx/
├── src/
│   ├── app/
│   │   ├── layout.tsx          # Root layout (27 Google Fonts)
│   │   ├── page.tsx            # Main app (splash, router, background)
│   │   ├── globals.css         # Tailwind styles
│   │   ├── reddit-callback/    # OAuth callback
│   │   └── api/                # API routes
│   ├── components/
│   │   ├── cyber/              # Custom cyberpunk components
│   │   ├── screens/            # 6 main screens
│   │   ├── ui/                 # shadcn/ui components
│   │   └── InAppBrowser.tsx
│   └── lib/
│       ├── stores.ts           # Zustand stores
│       ├── read-tracker.ts     # Read tracking
│       ├── idb-cache.ts        # IndexedDB cache
│       ├── reddit-scraper.ts   # Reddit scraper
│       ├── hardcoded-data/     # 22 chunk files (5,533 series)
│       └── ...
├── public/                     # Icons, backgrounds, PWA manifest
├── android/                    # Capacitor Android project
├── mobile-web/                 # Splash page for APK
├── next.config.ts
├── package.json
└── capacitor.config.ts
```

---

## Forking & Improving

This project is open for forking. Here are suggestions for how the community can improve it:

### Improve Chapter Discovery
- **Pagination:** The Reddit search API returns max 100 results. Implement pagination to find all chapters for series with 1000+ posts
- **Smart crawling:** Instead of searching, crawl from the last known chapter's "next chapter" link recursively
- **User submission:** Let users paste a wiki page HTML and auto-extract chapter links

### Better Reader Features
- **Text-to-speech** using the Web Speech API
- **Reading statistics** (words read, time spent, chapters completed)
- **Bookmarks** within chapters (save scroll position per chapter)
- **Font import** from device storage

### Enhanced OAuth
- **Automatic token refresh** before expiry (currently manual)
- **OAuth callback server** — Run a tiny HTTP server on the device to catch the OAuth redirect automatically

### Performance
- **Service Worker caching** for chapter content
- **Virtual scrolling** for chapter lists with 1000+ chapters
- **Web Workers** for heavy parsing

### UI/UX
- **Reading streaks and gamification** (daily reading goals, streak counter)
- **Series recommendations** based on reading history
- **Comment viewer** — Read Reddit comments in-app alongside the chapter
- **Tag system** — User-defined tags for organizing series

### Data
- **Export/Import library** — Backup and restore your reading progress
- **Sync across devices** — Cloud sync of library and read progress
- **Statistics dashboard** — Total chapters read, words consumed, time spent

### Contributing
1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-improvement`
3. Build and test: `npm run build && npm run start`
4. Submit a pull request with a clear description of what changed and why

### Data Updates
The baked series data (5,533 series in 22 chunk files) was parsed from CSV/JSON exports. To update with newer chapters:
1. Export the r/HFY wiki page to CSV/JSON
2. Run the parse script: `python3 scripts/parse-csvs.py`
3. Rebuild the chunk files
4. Rebuild the app: `npm run build`

---

## ⚠️ Warning to Forkers

**DO NOT add downloading, exporting, or file-saving features to this application.**

The r/HFY community is sustained by reader engagement on Reddit — upvotes, comments, and views motivate authors to continue writing. Tools that extract content from Reddit:

- **Reduce engagement** — Readers consume content off-platform instead of on Reddit
- **Discourage authors** — Writers see fewer upvotes and comments, leading to abandoned stories
- **Harm the community** — Fewer active writers means fewer new stories for everyone
- **May violate Reddit's Terms of Service** — Automated scraping and redistribution can result in IP bans

This app is designed as a **reader and organizer only**. Chapter content is fetched fresh from Reddit on each read, ensuring authors get the views and engagement they deserve. 

**Adding download features would cause grave and severe harm to the HFY community. Please don't do it. I don't approve it, the community doesn't approve it, doing so will gravely harm the community**

---

## License

This project is provided as-is for personal use. The series data is sourced from publicly available Reddit content. Reddit is a trademark of Reddit Inc. This project is not affiliated with or endorsed by Reddit.

## Credits

- **r/HFY community** — For writing and maintaining the stories
- **HFY Foundation** — Community-maintained story index
- **RedReader** (QuantumBadger) — UI inspiration
