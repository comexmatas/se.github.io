# se.github.io
App for registration of prices in Normal Sweden

### Goals
- **Single-file app**: All logic in `index.html` (no backend server). Data lives in `data/products.csv` in this GitHub repo.
- **Search**: Filter by `Product`, `Brand`, `Category`, and `Product ID`.
- **Capture**: Edit `Price` directly per row.
- **Sync**: Commit changes back to `data/products.csv` in the repository via GitHub API.

### Constraints & Saving Strategy
- The site is static and cannot write to disk. All persistence uses the GitHub REST API (`/contents` and, for large files, Git Data API for blobs/trees).
- Users must authenticate with a GitHub personal access token (PAT) that has `repo` access to this repository. Token is held in `sessionStorage` only.
- Semicolon-delimited CSV (`;`) is the canonical format. We preserve headers and write UTF‑8.

### Data Source & Schema
- Source file: `data/products.csv`.
- Expected headers (subset used by UI): `Product ID`, `Product`, `Price`, `Brand`, `Category`, `URL`, `Image URL`.
- Unique key: `Product ID` (falls back to `row_{index}` if missing, but rows without an ID are filtered out for editing).
- The app may add a volatile `lastSyncedAt` field in-memory prior to writing.

### Tech Choices
- **CSV parsing/generation**: Papa Parse (`papaparse` via CDN), delimiter set to `;`.
- **Transport**: GitHub REST API v3 using `fetch` with `Authorization: token <PAT>`.
- **UI**: Vanilla HTML/CSS/JS in `index.html`. No frameworks.
- **Client state**: In-memory arrays and Maps; drafts in `localStorage`; token in `sessionStorage`.

### Authentication
- Initial screen prompts for a GitHub token. We validate by calling the repo endpoint.
- On success, we show the main app and keep the token in session for the tab lifetime. Logout clears it.

### User Flows
1. **Authenticate**
   - Enter GitHub token; on success, main app becomes available.
2. **Load data**
   - Click "Load Data" → fetch `data/products.csv`. For files > ~1MB, use Git Data API to obtain the blob by SHA (via repo tree).
   - Parse CSV and build in-memory rows; enable Sync.
3. **Search/filter**
   - Debounced search across `Product`, `Brand`, `Category`, `Product ID`.
4. **Edit price**
   - Price column is editable. Numeric validation allows up to two decimals; normalizes commas to dots; formats to two decimals on blur.
   - Dirty rows are highlighted; edited count shown in status bar. Drafts auto-saved to `localStorage` and restored for up to 24h.
5. **Sync**
   - Pull latest CSV from GitHub, parse, then merge: local user edits win on conflicts. Optional overwrite warning if remote changes are newer than `lastSyncedAt`.
   - Rebuild CSV with `;` delimiter and PUT to `contents` API with commit message. Store returned `sha` for subsequent updates. Save `lastSyncTime` locally.

### Architecture (Single `index.html`)
- **Screens**:
  - Auth container: token input, submit, loading/inline error.
  - Main app: header, controls (search, Load Data, Sync), status bar, table/pagination, image preview modal.
- **Key functions**:
  - `validateToken()` → verifies PAT access to repo.
  - `loadCSVFromGitHub(filePath)` → fetches CSV (contents API; Git Data API for large), parses via Papa Parse.
  - `applySearch(query)` → filters `allRows` into `filteredRows`.
  - `render()` / `buildTable()` / `buildPagination()` → UI rendering with page size 100.
  - `onPriceInputChange()` / `onPriceInputBlur()` → validation, tracking `dirtyById`, `localChanges`, error states, draft save.
  - `attemptSync()` → fetch remote, `mergeDatabase(local, remote)`, `detectOverwrites()`, then `pushToGitHub()`.
  - `pushToGitHub()` → PUT encoded CSV to `data/products.csv`, managing `sha` and `lastSyncTime`.
- **Mobile & UX**:
  - Touch-optimized button handlers to avoid ghost clicks; responsive layout; larger touch targets; iOS/Android quirks handled.
  - Product names link to `URL` in a new tab. `Image URL` shows a thumbnail and opens a modal.
  - Keyboard shortcuts: Ctrl/Cmd+S to sync; Ctrl/Cmd+L to focus search.

### Validation & UX
- Price: only digits with optional decimal (two places), commas normalized to dots; error styling blocks Sync when errors exist.
- Status shows total rows, current page, and edited count; dirty rows highlighted.
- Draft recovery: restores only values that differ from original CSV.

### Performance
- Pagination at 100 rows/page to limit DOM size.
- Large-file path uses Git Data API (trees→blob) to avoid 1MB contents API limits.
- Lightweight rendering; no frameworks.

### Milestones & Acceptance Criteria
- **M1: Auth & Load**
  - Token validation succeeds/fails with clear feedback.
  - CSV loads from repo; large files handled via blob; table renders first page.
- **M2: Search & Navigate**
  - Debounced search filters rows; pagination works; mobile responsive.
- **M3: Edit & Validate**
  - Price edits validate and persist across pagination during session; drafts auto-save/restore.
- **M4: Sync & Merge**
  - PUT to GitHub updates `data/products.csv` with commit message, correct `sha` handling.
  - Local-wins merge is applied; optional overwrite warning when applicable; UI resets dirty state post-sync.
