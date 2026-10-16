# Changelog

All notable changes to DeskSearch are documented here.

## [0.7.0] - 2026-10-16

Chinese (zh-CN) UI, CJK-aware keyword search, and everything needed to actually
run and install the desktop app on Windows.

### Added
- **Chinese UI (zh-CN)** - dependency-free i18n layer (`src/ui/src/i18n`): 9 namespaces x 2 languages, 613 keys per language, `t()` with plurals and `{placeholder}` interpolation, English fallback, and locale-aware number/date formatting
- **Language switch** - sidebar, mobile header and Settings -> General -> Language; the explicit choice is persisted, otherwise the UI follows the system locale (`zh*` -> Chinese)
- **CJK bigram search** - the BM25 index now stores character bigrams of CJK runs (`body_cjk`), so unspaced Chinese queries match: `季度营收` finds `第三季度营收报告`. Latin search is unchanged (the bigram branch is skipped without CJK)
- `npm run check:i18n` - dictionary key-parity and behaviour checks; exits non-zero so it can gate CI
- `scripts/build-win.ps1` - PowerShell equivalent of `scripts/build.sh` (wheel / backend / electron / ui / test)

### Fixed
- **App exited at startup with a hand-edited config** - a UTF-8 BOM in `config.json` (written by Windows PowerShell `Set-Content -Encoding UTF8`, Notepad, ...) made `json.load` raise; the file is now read as `utf-8-sig` and any read/parse error falls back to defaults with a warning instead of aborting
- **Desktop window never showed the UI** - Electron hard-coded port 3777 while the backend takes its port from the config; host/port are now resolved from the config and passed to the backend explicitly, and a diagnostic page (with the backend log) replaces the blank window if startup fails
- **`httpx` and `python-multipart` were undeclared runtime dependencies** - `pip install desksearch && desksearch serve` crashed on import
- **POSIX-only `resource` module** - `daemon/service` was unimportable on Windows, and `/api/memory` plus `benchmark` raised `ModuleNotFoundError`
- **Port-in-use check never fired on Windows** - `SO_REUSEADDR` happily binds an occupied port; Windows now uses `SO_EXCLUSIVEADDRUSE`
- **`config.json` was written with the locale encoding** (cp936 on Chinese Windows) - now explicit UTF-8, written atomically
- **`DESKSEARCH_DATA_DIR` was ignored when set after import** - resolved lazily now, so wrappers and test runners get their own data dir
- **`folders add` rebuilt the entire corpus** - it now indexes only the folder being added
- **`NO_PROXY` entries containing a bracket** (e.g. `[::1]`, added by Docker Desktop and some VPN clients) broke every Hugging Face download with `InvalidURL`; such entries are sanitised on import
- **Typing Chinese fired a search for half-typed pinyin** (and saved it to recent searches) - the search input now handles IME composition events and no longer steals Enter/Arrow/Escape from the candidate list; voice search follows the UI language
- **BM25 index could not be migrated on Windows** - an in-place rename fails with `WinError 5` and used to disable BM25 for the whole session; the index directory is versioned instead
- Unseeded RNG with non-unit-norm mock vectors made a ranking assertion flaky
- **The packaged CLI crashed on `--version`** - `click.version_option` resolved the version through `importlib.metadata`, which raises inside a PyInstaller bundle (no `.dist-info` metadata); the version is now passed explicitly, and plugin entry-point discovery degrades to "no entry-point plugins" instead of failing there
- Docker `HEALTHCHECK` pointed at `/api/v1/health`, which does not exist (the route is `/api/health`)
- The embedder's ONNX hint referenced a module that is not in the repository
- **Version strings are in sync** across `pyproject.toml`, `src/desksearch/__init__.py`, `src/ui/package.json`, `electron/package.json` and the service-worker cache name

### Changed
- BM25 index directory is versioned (`bm25-v2`) and refilled from SQLite on a background thread, so upgrading an existing install needs no manual re-index and the server starts listening immediately (a 124k-chunk corpus refills in ~60 s while search keeps working)
- `stats` and the index-size calculation count any `bm25*` directory
- UI bundle grows from 90.7 kB to 150.0 kB raw (21.8 -> 40.0 kB gzip) because both dictionaries are inlined
- BM25 index size roughly doubles for CJK-heavy corpora (measured 88.9 MB -> 190.9 MB for 124k chunks)

## [0.6.4] - 2026-03-28

### Fixed
- **ONNX auto-export on first run** — model is now automatically exported to ONNX (INT8) in an isolated subprocess on first use; no more `FileNotFoundError` for new installs
- **Export script filename mismatch** — `export_onnx.py` now produces correct `-int8.onnx` filenames matching what the embedder expects
- **`_warm_search_engine()` crash** — SDK and daemon passed 4 args to a 3-param function; removed extra `embedder` argument
- **FAISS reconstruction zero-padding** — `_reconstruct_all_locked()` used `ntotal` (includes deleted) instead of live vector count, corrupting IVF training data
- **Soft-deleted vectors lost on restart** — HNSW soft-deleted set is now rebuilt from orphaned FAISS IDs on index reload
- **Missing `await` in search endpoint** — doc embeddings for related-doc feature now actually load
- **Unbounded thread creation** — async search now uses dedicated 2-worker executor instead of default
- **Service worker cache stale assets** — versioned cache name + network-first HTML prevents 404s after upgrades
- **Onboarding dead end** — added manual folder path input when no folders are auto-detected
- **Slow file discovery** — replaced `rglob("*")` with `os.walk` + directory pruning (skips `.git`, `node_modules`, etc. entirely)
- **BERT warning spam** — suppressed expected "weights not used" messages during model download/export
- **Invalid Rust edition** — fixed `edition = "2024"` to `"2021"` in `rust/Cargo.toml`
- **Version mismatches** — synced electron, UI, and Rust versions to match Python package
- **Firefox tempfile leak** — added proper `try/finally` cleanup in browser bookmarks plugin
- **Test marker** — fixed `@pytest.mark.asyncio` → `@pytest.mark.anyio`

### Changed
- Added `onnx` to `[full]` optional dependencies for auto-export support
- Added `src/ui/dist/` to `.gitignore` (intermediate build output)
- UI rebuilt with updated Onboarding component

## [0.6.0] - 2026-03-28

### Added
- **Connector plugin system** — extensible architecture with 4 built-in connectors:
  - Local files (file system scanning with scheduled sync)
  - Email (.eml/.mbox parsing with sender/subject/date extraction)
  - Chrome bookmarks (reads Chrome profile, folder hierarchy)
  - Slack export (ZIP import with username resolution)
- **ConnectorRegistry** — discover, enable/disable, configure, sync connectors via API
- **Connector API** (`/api/connectors/v2/`) — 6 endpoints for managing data sources
- **Advanced search filters** — filter by file type, date range, file size
- **Sort options** — sort by relevance, date, size, or name
- **Export formats** — results as JSON, CSV, or plain text
- **Favorites system** — bookmark important files via API
- **Recent files** — track recently opened documents
- **ONNX model export** — [Starbucks](https://arxiv.org/abs/2410.13230) 2D Matryoshka model exported to ONNX with INT8 quantization
  - Fast tier: 36MB (INT8) / 145MB (FP32)
  - Middle tier: 50MB (INT8) / 199MB (FP32)
  - Pro tier: 64MB (INT8) / 253MB (FP32)
- **Rust native binary** — complete rewrite of hot paths in Rust
  - 13MB standalone binary
  - PDF/DOCX/PPTX/XLSX parsers
  - Hybrid BM25 + dense search with usearch
  - Embedded web frontend
  - File watcher for live re-indexing
  - ONNX embedding via `ort` crate (load-dynamic)
- **Live indexing progress** — real-time progress bar with phase tracking, throughput, current file
- **`desksearch benchmark`** command for performance testing

### Changed
- **UI overhaul** — dark mode (navy-black palette), indigo/purple accent system, glass morphism
- **Settings redesign** — grouped sections with speed tier visual selector
- **Animations** — slide-up transitions, stagger children, floating logo
- **Responsive layout** — mobile-friendly settings and search
- **FastAPI lifespan** — migrated from deprecated `on_event` handlers
- **Lazy imports** — heavy modules (torch, transformers, FAISS) loaded on demand
- **Removed ORJSONResponse** — FastAPI handles JSON natively now

### Fixed
- Collections/topics/duplicates silently broken (wrong method name on store)
- Settings API missing `search_speed` field
- `POST /api/index` with no body returned 422 instead of indexing configured folders
- `/api/status` crashed due to missing `_compute_index_size_mb` function
- Duplicate `disk_stats()` and `vacuum_if_fragmented()` methods in store
- Dense-only search results missing file paths

### Performance
- **10x embedding speedup** — 171 chunks/sec (ONNX) vs 17 chunks/sec (PyTorch)
- Server startup: **1.3s** (lazy imports)
- 431 tests passing
- Deprecation warnings reduced from 124 to 9

## [0.5.0] - 2026-03-25

### Added
- **Starbucks 2D Matryoshka embeddings** — layer and dimension truncation with 3 speed tiers (fast/middle/pro); see [paper](https://arxiv.org/abs/2410.13230)
- Local model caching — loads only the layers needed per tier, no re-download
- Mock embedder for API tests — eliminates torch/FAISS segfault on Apple Silicon cleanup
- `desksearch benchmark` command for reproducible performance testing
- `desksearch doctor` health check command

### Changed
- Default embedding model switched to `ielabgroup/Starbucks-msmarco` (from all-MiniLM-L6-v2)
- Default embedding dimension reduced to 64d (from 384d) — 6x smaller index
- CLS pooling (trained method) replaces mean pooling
- Improved atexit cleanup to prevent segfaults on shutdown

### Performance
- Search latency p50: 1.11 ms (50K docs, 64d HNSW)
- FAISS raw search: 0.08 ms
- 896 queries/sec sustained throughput

## [0.4.0] - 2026-03-15

### Added
- **Matryoshka embedding truncation** — configurable `embedding_dim` (default 64d)
- Batch embedding pipeline (256 chunks/batch, 3–5x throughput improvement)
- Parallel file parsing with 6 worker threads
- Query expansion with synonym table and morphological variants
- Search result LRU cache (256 entries, invalidated on index mutation)
- `desksearch stats` command with detailed index breakdown
- FAISS auto-selection: FlatIP (<1K), HNSW (1K–50K), IVFFlat (>50K)

### Changed
- Chunking is now sentence-aware with configurable overlap
- Incremental indexing uses content hashing (skip unchanged files)

## [0.3.0] - 2026-02-20

### Added
- **Plugin system** — parser, search, and connector plugin types
- Built-in connectors: email (.eml/.mbox), browser bookmarks, clipboard monitor
- Plugin discovery via `entry_points` and `~/.desksearch/plugins/`
- **Web UI** — React 18 + TypeScript + Vite + Tailwind
- Dark mode, live search, file preview panel, analytics dashboard
- WebSocket endpoint for live indexing progress
- Collections and saved searches
- `desksearch daemon install/uninstall` for macOS LaunchAgent
- API authentication via optional bearer token
- PWA manifest for standalone app mode

### Changed
- API server migrated to FastAPI with async routes
- Daemon log streaming via `daemon logs --follow`

## [0.2.0] - 2026-01-25

### Added
- **Hybrid search** — BM25 (Tantivy) + dense (FAISS) with reciprocal rank fusion
- Configurable RRF alpha parameter (0.0 = BM25 only, 1.0 = dense only)
- Filename and recency boosting in result ranking
- Snippet extraction with query term highlighting
- File watcher (watchdog) for automatic re-indexing
- Background daemon with `desksearch daemon start/stop/status`
- REST API: search, index, files, folders, status, health endpoints
- Support for 30+ file formats (PDF, DOCX, PPTX, XLSX, code, email, archives)
- `desksearch folders add/remove/list` commands
- `--json` flag on all CLI commands

### Changed
- Search results now include snippets and relevance scores
- Index storage moved to `~/.desksearch/` with separate BM25 and dense directories

## [0.1.0] - 2025-12-20

### Added
- Initial release
- BM25 full-text search via Tantivy
- Basic CLI: `desksearch search`, `desksearch index`, `desksearch status`
- SQLite metadata store with FTS5 fallback
- Onboarding wizard with folder auto-detection
- Configuration via `~/.desksearch/config.json`
- Rich terminal output with progress bars
- Document parsing for PDF, DOCX, TXT, Markdown, and common code files
- `desksearch serve` to start local web server on port 3777

## [0.6.1] - 2026-03-28

### Fixed
- Tier naming: "regular" → "middle" (fast / middle / pro)
- Similarity metric: dot product (inner product) instead of cosine — matches Starbucks model training
- Removed incorrect L2 normalization from dense index
- Corrected Starbucks paper author order (Zhuang*, Wang*, Zheng, Koopman, Zuccon)
- Duplicate `disk_stats()` and `vacuum_if_fragmented()` methods in store
- Collections/topics/duplicates silently broken (wrong store method name)
- Settings API missing `search_speed` field
- `POST /api/index` with no body returned 422

### Added
- "Powered by Starbucks Embeddings" section in README with paper citation and BibTeX
- Real screenshots of every UI page (home, search, dashboard, settings, data sources, files)
- Animated SVG terminal demo in README
- Starbucks paper reference in Settings UI and pyproject.toml keywords
