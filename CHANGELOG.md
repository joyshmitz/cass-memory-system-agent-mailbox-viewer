# Changelog

All notable changes to the **cass-memory-system-agent-mailbox-viewer** repository are documented here.

This project is a static export bundle for [MCP Agent Mail](https://github.com/Dicklesworthstone/mcp_agent_mail). It ships a scrubbed SQLite mailbox snapshot together with a self-contained browser viewer so teammates can browse agent threads, search messages, and inspect metadata without direct database access.

> There are no tagged releases or GitHub Releases. History is tracked per-commit on the `main` branch.

---

## Unreleased

_No unreleased changes at this time._

---

## 2026-02-21 -- Licensing and repository metadata

Two housekeeping commits landed on the same day, adding legal and social-sharing assets.

### MIT + OpenAI/Anthropic Rider license

[`3fda939`](https://github.com/Dicklesworthstone/cass-memory-system-agent-mailbox-viewer/commit/3fda939e97b4d1f35fcd1a803fb4fdb023ae2f0e)

- Added `LICENSE` file containing the **MIT License with OpenAI/Anthropic Rider**.
- The rider prohibits use by OpenAI, Anthropic, their affiliates, and anyone acting on their behalf without express written permission from Jeffrey Emanuel.
- Covers the Software and all derivative works; breach triggers automatic license termination.

### Social preview image

[`e430ddd`](https://github.com/Dicklesworthstone/cass-memory-system-agent-mailbox-viewer/commit/e430dddd228c5f2327d414782dab4930807520e2)

- Added `gh_og_share_image.png` (1280x640) for consistent OpenGraph previews when the repository URL is shared on social media or chat platforms.

---

## 2025-12-08 -- Initial mailbox export

[`6a809db`](https://github.com/Dicklesworthstone/cass-memory-system-agent-mailbox-viewer/commit/6a809dbba0702c89340dd20f7962f0c0f3ef8000)

First commit. Established the full static viewer bundle generated from the `cass_memory_system` MCP Agent Mail project (23 files, ~8,700 lines).

---

### Mailbox snapshot

- **494 messages** from **71 agents**, exported with the `standard` scrub preset on 2025-12-08.
- Agent names retained; acknowledgement/read flags cleared, file reservations removed (328), agent links removed (131), recipients cleared (1,099).
- Database stored as `mailbox.sqlite3` (1.9 MB) with SHA-256 integrity hash recorded in `manifest.json`.
- Ed25519-signed manifest (`manifest.sig.json`) enables cryptographic bundle verification via `mcp_agent_mail.cli share verify`.

---

### Viewer application (`viewer/`)

A single-page application built with **Alpine.js** and **Tailwind CSS** (CDN), backed by **sql.js** running the full SQLite database in-browser via WebAssembly. No server-side component is required.

#### In-browser SQL engine

- Loads `mailbox.sqlite3` entirely into the browser using sql.js/WebAssembly.
- Detects FTS5 full-text-search tables at startup and uses them when present.
- Supports chunked database reassembly via `chunk_manifest` in the manifest (wired but not exercised in this snapshot).
- Opt-in `explainMode` flag logs `EXPLAIN QUERY PLAN` output to the browser console for SQL diagnostics.

#### Thread and message browsing

- **Thread explorer** -- aggregates messages by `thread_id`, displays per-thread message count, latest subject, importance level, and a 160-character snippet preview.
- **Three view modes** -- Split (message list + reading pane side by side), List (messages only), and Threads (thread-centric navigation).
- **Message detail pane** -- loads the full `body_md` on demand and renders it as Markdown using Marked.js with GitHub Flavored Markdown support, sanitized through DOMPurify.
- **Thread cross-navigation** -- clicking a thread reference in any view jumps to the full thread with scroll-into-view.

#### Search

- **Boolean search engine** -- shunting-yard parser converts query text into an AST supporting `AND`, `OR`, `NOT`, quoted phrases (`"exact match"`), and parenthesized sub-expressions.
- **Dual execution path** -- queries run against FTS5 when available; falls back to SQL `LIKE` on subject and body columns.
- **Debounced input** -- 140 ms debounce prevents per-keystroke SQL overhead.
- **Thread search** -- separate lightweight substring search across thread subjects, identifiers, and snippets.

#### Filtering and sorting

- **Multi-axis filters** -- project, sender, recipient, importance level (urgent / high / normal / low with counts), thread membership, and message kind (user vs. administrative).
- **Administrative message detection** -- regex patterns on subject and body automatically classify auto-handshake and contact-request messages; hidden by default under the "user" kind filter.
- **Sort modes** -- newest first, oldest first, alphabetical by subject, alphabetical by sender, and longest body.
- **Bulk selection** -- select-all and individual checkboxes with mark-as-read (local state only in the static viewer).

#### Performance

- **Virtual scrolling** -- custom `requestAnimationFrame`-based virtual list with dynamic row-height estimation via exponential moving average and configurable overscan. Replaced Clusterize.js for finer control.
- **OPFS caching** -- on browsers supporting the Origin Private File System, the database bytes are written to OPFS after the first network load, keyed by SHA-256. Subsequent visits serve from cache with stale-key invalidation.
- **Asset preloading** -- `<link rel="preload">` hints for `sql-wasm.js`, `sql-wasm.wasm`, and `mailbox.sqlite3` reduce time-to-interactive.
- **Idle-time cache writes** -- OPFS writes are deferred via `requestIdleCallback` (with `setTimeout` fallback) to avoid blocking the main thread during initial render.
- **Batch recipient resolution** -- recipients are loaded in a single SQL query and mapped by message ID, avoiding N+1 query overhead.

#### Security

- **Trusted Types** -- `mailViewerDOMPurify` policy gates all `innerHTML` writes through DOMPurify. A separate default policy covers Clusterize.js compatibility paths. Script URLs are restricted to the `./vendor/` directory.
- **Content Security Policy** -- inline `<meta>` CSP restricts script sources to `self`, Tailwind CDN, jsDelivr, and unpkg; style sources to `self`, Google Fonts, jsDelivr, unpkg, and cdnjs; and blocks `object-src` entirely.
- **Subresource Integrity** -- all vendored scripts carry SRI hashes recorded in both `viewer/vendor_manifest.json` and `manifest.json` under `viewer.sri`.
- **DOMPurify sanitization** -- all user-supplied Markdown is sanitized before DOM insertion, with a plain-text escape fallback when DOMPurify is unavailable.

#### User interface

- **Dark mode** -- toggle button with `localStorage` persistence; respects `prefers-color-scheme: dark` on first visit via a synchronous inline script to prevent flash of wrong theme.
- **Responsive mobile layout** -- media-query breakpoint at 768 px; filter panel auto-hides on scroll-down; message detail renders as a full-screen modal overlay on small screens with body scroll lock.
- **Importance badges** -- color-coded pills: red for urgent, yellow for high priority.
- **Project badges** -- deterministic hash-based color palette (six Tailwind color families) ensures the same project always gets the same badge color.
- **Relative timestamps** -- displays "Just now", "Yesterday", weekday name, or `Mon DD` depending on message age; full timestamp shown separately.
- **Lucide icons** -- icon set loaded from unpkg CDN, re-initialized after virtual list renders.

---

### Deployment infrastructure

- **Root redirect** -- `index.html` at the repository root issues an instant `<meta http-equiv="refresh">` plus a JavaScript `location.replace` to `viewer/index.html`.
- **Cross-Origin Isolation headers** -- `_headers` file sets `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` for hosts that support Netlify/Cloudflare-style header files.
- **COI service worker** -- `coi-serviceworker.js` (v0.1.7) provides the same COOP/COEP headers on GitHub Pages, where `_headers` is not supported, via service worker interception. Requires uncommenting one script tag.
- **`.nojekyll`** -- disables GitHub Pages' Jekyll processing so `_headers` and `.wasm` files are served with correct MIME types.
- **Preview hot-reload** -- `preview-reload.js` polls a local `/__preview__/status` endpoint and auto-refreshes the viewer when the snapshot signature changes during local development.

---

### Manifest and integrity

- `manifest.json` -- schema version `0.1.0`; records export configuration, scrub statistics, hosting-target detection signals, viewer asset SRI hashes, and database SHA-256.
- `manifest.sig.json` -- Ed25519 signature for cryptographic verification via `uv run python -m mcp_agent_mail.cli share verify`.
- `viewer/vendor_manifest.json` -- SHA-256 hashes and upstream source URLs for all vendored libraries.
- `viewer/data/messages.json` + `viewer/data/meta.json` -- pre-rendered JSON caches (494 messages) for the viewer.

---

### Vendored dependencies

| Library | Version | Purpose |
|---|---|---|
| [sql.js](https://github.com/sql-js/sql.js) | 1.10.1 | SQLite engine compiled to WebAssembly |
| [Marked](https://cdn.jsdelivr.net/npm/marked@11.0.0/) | 11.0.0 | Markdown-to-HTML rendering (GFM) |
| [DOMPurify](https://cdn.jsdelivr.net/npm/dompurify@3.0.8/) | 3.0.8 | HTML sanitization for rendered Markdown |
| [Clusterize.js](https://github.com/NeXTs/Clusterize.js) | 0.18.0 | Virtual-list CSS (JS superseded by custom implementation) |

---

### Documentation

- `README.md` -- project overview, quick-start guide, snapshot regeneration instructions, integrity verification, and troubleshooting.
- `HOW_TO_DEPLOY.md` -- step-by-step deployment checklists for GitHub Pages, Cloudflare Pages, Netlify, Amazon S3, and generic static hosts.
