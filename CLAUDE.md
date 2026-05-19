# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

GPT Voyager Plus is a Chrome Extension (Manifest V3) that adds a sidebar panel to ChatGPT (`chatgpt.com`, `chat.openai.com`) and 豆包 (`www.doubao.com`). It provides conversation timeline, formula workspace, Mermaid diagram rendering, prompt library, and conversation index features. Forked from [GPT-Voyager](https://github.com/Duang777/GPT-Voyager).

## Build / no build

There is no `package.json`, build config, or `src/` directory in this repo. The source files (TypeScript/React under `src/`) live in a separate private repository. Only built artifacts — `content.js`, `content.js.map`, `background.js`, `background.js.map` — are committed here. Source maps in the `.map` files reference `../src/` paths (see "Source architecture" below) for debugging in DevTools.

To load and test locally: open `chrome://extensions/`, enable Developer Mode, click "Load unpacked", and select this directory.

## Architecture

### Entry points

- **`manifest.json`** — Declares `background.js` as the service worker and `content.js` as the content script injected at `document_idle` on supported hosts. Permissions: `storage` only.
- **`background.js`** (`src/background/index.ts`) — Minimal service worker. Logs install/update events and responds to `GV_PING` runtime messages for liveness checks.
- **`content.js`** (single ~9MB bundle, `src/content/index.tsx`) — Mounts the entire React sidebar into a Shadow DOM host element (`#gpt-voyager-host`) appended to `document.body`. Shipped styles are scoped via `:host` in the Shadow DOM. The panel is responsive (min 280px, max 640px, default 360px, resizable) with a toggle button.

### Source module map (from source maps)

| Source file | Purpose |
|---|---|
| `src/content/index.tsx` | Entry — creates Shadow DOM host, renders `<App />` via `ReactDOM.createRoot` |
| `src/content/App.tsx` | Main shell: workspace tabs (总览, 导出中心, 星标对话, 会话时间线, 公式工作台, Mermaid 工作台, 分类管理, 会话索引), panel state persistence (`chrome.storage.local`), chat content width clamping |
| `src/content/chatPlatform.ts` | Platform detection and DOM abstraction. Detects `chatgpt` vs `doubao` from hostname. Provides platform-specific selectors for message nodes (`data-message-author-role` vs `data-message-id`), composer textareas, conversation IDs from URL paths (`/c/xxx` vs `/chat/123`), and conversation titles. |
| `src/content/conversationIndex.ts` | Scans the page sidebar for conversation links, indexes them (id, title, platform, timestamp), persists to `chrome.storage.local` under `gpt_voyager_conversation_index_v1` with storage scope support. |
| `src/content/conversationTimeline.ts` | Extracts Q/A message blocks from the conversation DOM, builds a scrollable timeline with heading extraction (H1 tags from assistant responses), supports keyword/role/tag filtering and markdown export. |
| `src/content/conversationFormula.ts` | Extracts KaTeX and MathJax formulas from conversation messages (both inline `$...$` and display `$$...$$` modes). Formula tab in the sidebar. |
| `src/content/conversationMermaid.ts` | Extracts Mermaid diagram code blocks from messages. |
| `src/content/conversationExport.ts` | Exports current conversation to Markdown (via Turndown) or HTML. Handles math token preservation, DOM cleanup (button/input/style/script removal), file naming sanitization. |
| `src/content/mermaidRender.ts` | Client-side Mermaid diagram rendering from code blocks. |
| `src/content/promptLibrary.ts` | Prompt template management — CRUD for prompt snippets with `{{variable}}` substitution, tag-based categorization, variable presets. Stored under `gpt_voyager_prompt_library_v1`. Supports quick-import and insertion into the page composer. |
| `src/content/formulaFavoritesStore.ts` | Formula bookmarking — stores conversation-scoped formula favorites with LaTeX, MathML, and display mode. |
| `src/content/mermaidFavoritesStore.ts` | Mermaid diagram bookmarking. |
| `src/content/timelineAnnotationsStore.ts` | Per-node annotations on the timeline (highlighting, tags). |
| `src/content/classificationStore.ts` | Conversation classification — folders and tags for organizing conversations. |
| `src/content/settingsStore.ts` | User preferences — scan interval, density, sort order, export format, prompt insert mode, account isolation toggle, chat width percent. |
| `src/content/dataBackup.ts` | Full JSON backup/restore covering all stores (conversation index, classification, prompt library, formula/mermaid favorites, timeline annotations, settings). Schema version 1. |
| `src/content/accountScope.ts` | Detects logged-in account email from `__NEXT_DATA__` script tag or page DOM to derive a storage scope key for account isolation. |
| `src/content/storageScope.ts` | Storage key scoping — prefix keys with a normalized scope segment (`gpt_voyager_xxx_v1__scope_value`) to support multi-account isolation within `chrome.storage.local`. |

### Key bundled libraries (included in content.js)

- **React 18** + **ReactDOM/client** — UI framework
- **Turndown** — HTML → Markdown conversion for conversation export
- **KaTeX** — Math rendering (source of extracted formulas)
- **DOMPurify** — HTML sanitization
- **html2canvas** — DOM screenshots
- **Mermaid** — Diagram rendering in the Mermaid workspace tab
- **esbuild** bundler runtime (based on `__commonJS`/`__toESM` helper patterns)

### Storage conventions

All data is stored in `chrome.storage.local`. Keys follow the pattern `gpt_voyager_{entity}_v1` with optional `__scope` suffix for account isolation (e.g., `gpt_voyager_prompt_library_v1__doubao_user@example.com`). The `storageScope.ts` module normalizes scope segments (lowercase, alphanumeric + underscore + hyphen only, max 64 chars).

Key storage keys:
- `gpt_voyager_conversation_index_v1` — indexed conversations
- `gpt_voyager_classification_v1` — folders/tags
- `gpt_voyager_prompt_library_v1` — prompt templates
- `gpt_voyager_formula_favorites_v1` — formula bookmarks
- `gpt_voyager_mermaid_favorites_v1` — mermaid bookmarks
- `gpt_voyager_timeline_annotations_v1` — timeline node annotations
- `gpt_voyager_panel_state` — sidebar width and collapsed state
- `gpt_voyager_settings` — user preferences

### Platform abstraction pattern

All platform-specific logic (ChatGPT vs 豆包) goes through `chatPlatform.ts` functions with a `platform` parameter defaulting to `gvGetActiveChatPlatform()`. This ensures the sidebar works identically on both platforms while handling their DOM differences (message node selectors, URL patterns, composer selectors, title cleaning, etc.).
