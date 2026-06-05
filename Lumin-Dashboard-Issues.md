# Lumin Dashboard — Master Issues, Bug & Feature Gap Report
### Full deep-code audit of every source file

> Every item is traced to a specific file, line number, and exact variable name.
> Covers: bugs, missing configs, hardcoded values, architecture gaps, organisation, performance.

---

## ⚠️  SECTION 0 — DEPLOY / RUNTIME PERSISTENCE (Root Cause of Data Loss)

### DEPLOY-01 · `runtime-config.json` Is Wiped on Every Render Redeploy (User-Reported)
**File:** `dashboard/server.js` line 37: `const RUNTIME_CONFIG_PATH = path.join(__dirname, 'runtime-config.json')`
**Root cause:** The entire runtime config — feature flags, active model, presence settings, bot config, rate limits, key stats, migration config — is written to the **local filesystem** at `dashboard/runtime-config.json`. On Render (and any ephemeral PaaS — Railway, Fly, Heroku, Docker), every redeploy creates a **fresh container**. The file does not exist in the fresh container. On startup the server writes a blank `{}` to `runtimeConfig`, so **every feature flag reverts to its source-code default**, every override is lost, all API key stats reset to zero.
- **What resets on every deploy:** `ENABLE_GEMMA`, `ENABLE_RAG`, `CACHE_ENABLED`, `CROSS_CONTEXT_ENABLED`, `WEEKLY_SUMMARY_ENABLED`, `ENABLE_FUNCTION_CALLING`, `ENABLE_WEB_SEARCH`, `ENABLE_IMAGE/VIDEO/AUDIO/FILE_PROCESSING`, custom presence status, active model override, all bot config overrides, all rate limit overrides, migration enable/disable flag, all API key usage stats
- **Fix:** Move `runtimeConfig` persistence to MongoDB (a new `bot_runtime_config` collection with a single document). This survives deploys because MongoDB is external to the container.
- **Secondary fix:** Add a "Persist to DB" button in the dashboard that force-saves all current runtime settings to MongoDB, and a startup routine that restores from DB before the config file.

---

## SECTION 1 — CRITICAL BUGS

### BUG-01 · API Key Stats: `dailyRequests` Field Doesn't Exist — Always Shows 0
**File:** `dashboard/public/js/commands.js` → `renderApiKeysPanel()`
`k.dailyRequests||0` is rendered for "Daily Reqs" but `getApiKeyStats()` in `managers/ApiKeyManager.js` never returns a `dailyRequests` field. The field doesn't exist; it's always 0.
**Fix:** Add daily counter tracking to `ApiKeyManager.js` (reset at UTC midnight) or rename to `totalRequests` which does exist.

### BUG-02 · API Key Stats: `errorCount` Field Doesn't Exist — Errors Always 0
**File:** `dashboard/public/js/commands.js` → `renderApiKeysPanel()`
Renders `k.errorCount||0` but `getApiKeyStats()` returns `errors`, not `errorCount`. Error counter permanently shows 0 in the UI even when keys are failing.
**Fix:** `k.errorCount` → `k.errors`

### BUG-03 · Stat Grid Has Two Duplicate Cards Showing Identical `totalHistories`
**File:** `dashboard/public/js/app.js` → `buildStatGrid()`
Two consecutive cards: `{l:'Chat Sessions', v:fmtN(d.totalHistories)}` and `{l:'Histories', v:fmtN(d.totalHistories)}` — same source value, different labels. One slot is completely wasted. `totalServers_s` (server settings count) is fetched but never displayed anywhere.
**Fix:** Replace one duplicate with `totalServers_s`.

### BUG-04 · Server Settings Modal Has Wrong/Nonexistent Field Names
**File:** `dashboard/public/js/app.js` → `window._viewServerSettings`
Hardcoded editable fields: `['embedColor','maxHistoryLength','customPersonality','responseMode']`. Neither `maxHistoryLength` nor `responseMode` exist in `DEFAULT_SERVER_SETTINGS` (`managers/StateManager.js`). The real fields are `responseFormat` and the max history is `STATE_CONFIG.MAX_MESSAGES`. **Six actual settings are completely invisible:** `selectedModel`, `responseFormat`, `showActionButtons`, `continuousReply`, `overrideUserSettings`, `serverChatHistory`, `allowedChannels`, `gemmaEnabled`.
**Fix:** Auto-generate fields from the actual `DEFAULT_SERVER_SETTINGS` schema.

### BUG-05 · `allHistories` Message Count Always 0
**File:** `dashboard/server.js` → `/api/cmd/all-histories`
`messageCount: Array.isArray(state.chatHistories[id]) ? state.chatHistories[id].length : 0` — `chatHistories[id]` is always `{[channelId]: [...messages]}` (never a flat array), so the check is always false and count always 0.
**Fix:** `Object.values(state.chatHistories[id] || {}).flat().length`

### BUG-06 · Settings Save Silently Drops Falsy/Empty Values
**File:** `dashboard/public/js/app.js` → `window._saveServerSettings`
`if(val) data[k] = val` — cannot save `false`, `0`, or empty string to any setting. Cannot turn off `continuousReply`, clear `customPersonality`, or reset `embedColor` to blank.
**Fix:** `if(val !== undefined && val !== null)`

### BUG-07 · Feature Flags Toggle Is a Silent No-Op for 9 of 13 Flags (Critical)
**Files:** `dashboard/server.js` toggle endpoint, `modules/shared/runtimeFlags.js`, `modules/message/MessageProcessor.js`
`runtimeFlags.js` only wires 4 flags to `getFlag()`: `ENABLE_RAG`, `CROSS_CONTEXT_ENABLED`, `CACHE_ENABLED`, `WEEKLY_SUMMARY_ENABLED`. The other 9 flags (`ENABLE_GEMMA`, `ENABLE_FUNCTION_CALLING`, `ENABLE_WEB_SEARCH`, `ENABLE_IMAGE_PROCESSING`, `ENABLE_VIDEO_PROCESSING`, `ENABLE_AUDIO_PROCESSING`, `ENABLE_FILE_PROCESSING`, `PDF_ENABLED_FOR_GEMINI`, `CYCLE_GEMMA_WITH_GEMINI`) are read as **static ES module imports** in `MessageProcessor.js`. The dashboard saves them to disk and shows a success toast, but the bot never reads the updated value — full restart required.
**The UI never communicates this distinction.**
**Fix:** Extend `runtimeFlags.js` to cover all flags and update all consumers; OR add a "requires restart" label to affected flags in the UI.

### BUG-08 · `ENABLE_GEMMA` Toggle Has Zero Runtime Effect
**File:** `modules/message/MessageProcessor.js` line 150: `if (ENABLE_GEMMA) {` — static import.
Same as BUG-07, called out separately because it's the most frequently toggled flag and most confusing.

### BUG-09 · Per-Model Rate-Limit Windows Lost on Restart
**File:** `managers/ApiKeyManager.js` → `dumpKeyStats()` / `loadKeyStats()`
`keyModelRateLimits` (per-key, per-model sliding windows) is **not** included in `dumpKeyStats()`. After restart the bot thinks all models are fresh and may burst-request until backing off.
**Fix:** Include window data in `dumpKeyStats()`, restoring only windows still within the 60-second window.

### BUG-10 · `imageConfig` Not Defined in `config.js` — Image Daily Limit Always Hardcoded to 10
**File:** `managers/QueueManager.js` line 287: `const limit = config.imageConfig?.maxPerDay || 10`
`config.js` never exports an `imageConfig` object. The `?.` silently returns undefined, so the daily image generation limit is permanently 10, with no way to change it from code or dashboard.
**Fix:** Add `export const IMAGE_CONFIG = { MAX_PER_DAY: 10, PER_MINUTE_COOLDOWN_MS: 60_000 }` to `modules/config.js` and consume it in `QueueManager.js`.

### BUG-11 · `WeeklySummaryJob.js` Uses Raw String Literal for Model — Bypasses MODELS Map
**File:** `commands/summary/WeeklySummaryJob.js` line 42: `const SUMMARY_MODEL = 'gemini-3.1-flash-lite'`
This is a raw string, not `MODELS['gemini-3.1-flash-lite']`. If the model API name changes, this breaks silently with no error at import time.

### BUG-12 · `SessionSummaryJob.js` Uses Raw String Literal for Model
**File:** `commands/summary/SessionSummaryJob.js` line 35: `const SUMMARY_MODEL = 'gemma-3-12b-it'`
Same issue as BUG-11 — raw string, not from MODELS map. Will silently use a dead model name.

### BUG-13 · Dead Auth Code Throughout Frontend
**Files:** `dashboard/public/js/config.js` exports `getToken()→''`, `setToken()→{}`, `hasToken()→false`, `clearToken()→{}`. `app.js` and `api.js` import and call them. All are no-ops after the cookie-auth rewrite.

---

## SECTION 2 — MASTER CONFIG PANEL: EVERY SETTING THAT NEEDS A UI

> These are **all** the values from every `modules/config.js`, `managers/config.js`, `memory/config.js`, `database/config.js`, and key files that currently have zero dashboard UI. Every one of them must be editable at runtime.

### CONFIG-01 · AI Routing & Model Settings (currently only partially exposed)
| Setting | Current Value | Location |
|---|---|---|
| `ENABLE_GEMMA` | `true` | modules/config.js |
| `GEMMA_DEFAULT_MODEL` | `'gemma-4-26b'` (key in MODELS) | modules/config.js |
| `GEMMA_FALLBACK_MODEL` | `'gemma-4-31b'` (key in MODELS) | modules/config.js |
| `CYCLE_GEMMA_WITH_GEMINI` | `false` | modules/config.js |
| `DEFAULT_MODEL` | `'gemini-3.1-flash-lite'` | modules/config.js |
| `MODEL_FALLBACK_CHAIN` | `['gemini-3.1-flash-lite', 'gemini-3.5-flash']` | modules/config.js |
| `MODEL_CALL_THRESHOLDS` | `{'gemini-3.1-flash-lite': 500}` | modules/config.js |
| `GEMMA_DAILY_LIMIT_PER_KEY` | `1500` | modules/config.js |

### CONFIG-02 · Generation Parameters (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `GENERATION_DEFAULTS.TEMPERATURE` | `1.0` | modules/config.js |
| `GENERATION_DEFAULTS.TOP_P` | `0.95` | modules/config.js |
| Gemma thinking enabled | `false` (hardcoded) | modules/config.js `getGemmaConfig()` |
| Gemma thinking level | `'high'` (hardcoded, only used if thinking=true) | modules/config.js |
| Gemini 3 thinking | disabled (commented out) | modules/config.js |
| Gemini 2 thinking budget | `-1` dynamic (commented out) | modules/config.js |

### CONFIG-03 · Safety Settings (ZERO dashboard exposure)
All 5 categories currently hardcoded to `BLOCK_NONE`:
- `HARM_CATEGORY_HARASSMENT`
- `HARM_CATEGORY_HATE_SPEECH`
- `HARM_CATEGORY_SEXUALLY_EXPLICIT`
- `HARM_CATEGORY_DANGEROUS_CONTENT`
- `HARM_CATEGORY_CIVIC_INTEGRITY`

Each should have a dropdown: `BLOCK_NONE` / `BLOCK_LOW_AND_ABOVE` / `BLOCK_MEDIUM_AND_ABOVE` / `BLOCK_ONLY_HIGH`

### CONFIG-04 · Queue & Processing Limits (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `MAX_QUEUE_DEPTH_PER_USER` | `5` | modules/config.js |
| `MAX_FUNCTION_CALL_TURNS` | `5` | ResponseHandler.js (hardcoded) |
| `PROCESSING_TIMEOUT_MS` | `90_000` (90s) | MessageProcessor.js (hardcoded) |
| `TYPING_INTERVAL_MS` | `4_000` | ResponseHandler.js (hardcoded) |
| `TYPING_TIMEOUT_MS` | `120_000` (2 min) | ResponseHandler.js (hardcoded) |
| `UPDATE_DEBOUNCE_MS` | `350` | ResponseHandler.js (hardcoded) |
| `CHAR_THRESHOLD` | `150` | ResponseHandler.js (hardcoded) |
| `MAX_RETRY_ATTEMPTS` | `3` | ResponseHandler.js (hardcoded) |
| `RETRY_DELAYS.DEFAULT` | `1500` | ResponseHandler.js (hardcoded) |
| `RETRY_DELAYS.RATE_LIMIT` | `2000` | ResponseHandler.js (hardcoded) |
| `RETRY_DELAYS.FILE_ERROR` | `1000` | ResponseHandler.js (hardcoded) |
| `KEY_SWITCH_HOLD_MS` | `1500` | modules/config.js |
| `PER_KEY_DAILY_LIMIT` | `500` | QueueManager.js (hardcoded) |
| `RAM_MEDIA_SUSPEND_THRESHOLD_MB` | `380` | modules/config.js |

### CONFIG-05 · Image Generation Limits (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `IMAGE_CONFIG.MAX_PER_DAY` | `10` (hardcoded fallback, BUG-10) | QueueManager.js |
| `IMAGE_RATE_LIMIT.PER_MINUTE_COOLDOWN_MS` | `60_000` | QueueManager.js (hardcoded) |
| Summary daily limit per user | `10` | QueueManager.js `SUMMARY_RATE_LIMIT` (hardcoded) |

### CONFIG-06 · State & Bot Behavior (partially exposed, wrong fields)
| Setting | Current Value | Location |
|---|---|---|
| `STATE_CONFIG.MAX_MESSAGES` | `50` | modules/config.js |
| `STATE_CONFIG.CONTEXT_BREAK_THRESHOLD` | `1_800_000` (30 min) | modules/config.js |
| `RESOURCE_CONFIG.STATE_SAVE_INTERVAL` | `300_000` (5 min) | modules/config.js |
| `RESOURCE_CONFIG.STATS_LOG_INTERVAL` | `900_000` (15 min) | modules/config.js |
| `BOT_CONFIG.DEFAULT_RESPONSE_FORMAT` | `'Normal'` | modules/config.js |
| `BOT_CONFIG.HEX_COLOUR` | `'#5B7C99'` | modules/config.js |
| `BOT_CONFIG.WORK_IN_DMS` | `true` | modules/config.js |

### CONFIG-07 · Media Processing (partially exposed — global toggles only, no per-format)
| Setting | Current Value | Location |
|---|---|---|
| `ENABLE_IMAGE_PROCESSING` | `true` | modules/config.js |
| `ENABLE_VIDEO_PROCESSING` | `false` | modules/config.js |
| `ENABLE_AUDIO_PROCESSING` | `false` | modules/config.js |
| `ENABLE_FILE_PROCESSING` | `false` | modules/config.js |
| `PDF_ENABLED_FOR_GEMINI` | `false` | modules/config.js |
| `GEMMA_SUPPORTED_MIME_PREFIXES` | `['image/']` | modules/config.js (hardcoded) |
| `GEMMA_SUPPORTED_EXTENSIONS` | 7 image exts | modules/config.js (hardcoded) |

### CONFIG-08 · Attachment Format Granular Control (ZERO dashboard exposure)
`FileValidator.js` defines accepted formats by category. Currently only global category toggles exist. Needed: per-format enable/disable for:
- **Images direct:** `.jpg .jpeg .png .webp .gif .heif .tiff .bmp`
- **Images convertible:** `.svg .avif .ico .psd .eps .raw .cr2 .nef`
- **Video direct:** `.mp4 .mov .mpeg .mpg .webm .avi .wmv .3gpp .flv`
- **Video convertible:** `.mkv .vob .ogv .ts .m2ts .divx`
- **Audio direct:** `.mp3 .wav .aiff .aac .ogg .flac .m4a .opus`
- **Audio convertible:** `.wma .amr .mid .midi .ra`
- **Documents — Office:** `.doc .docx .xls .xlsx .csv .tsv .pptx .rtf`
- **Documents — Markup:** `.html .xml .md`
- **Documents — Code:** `.py .java .js .css .json .sql .log .c .cpp .h .hpp .cs .php .rb .go .rs .swift .kt .scala .sh .bat .yml .yaml .ini .cfg .conf`
- **Video conversion options:** `MOVFLAGS`, `PIX_FMT`, `VIDEO_CODEC`, `AUDIO_CODEC`
- **Audio conversion:** `FORMAT`, `CODEC`, `BITRATE`

### CONFIG-09 · Poll Config (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `POLL_CONFIG.maxPollsPerMinute` | `3` | modules/config.js |
| `POLL_CONFIG.maxResultsPerMinute` | `5` | modules/config.js |
| `POLL_CONFIG.autoRespondToPolls` | `true` | modules/config.js |
| `POLL_CONFIG.minVotesForAnalysis` | `1` | modules/config.js |

### CONFIG-10 · Upload / Pastebin Config (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `UPLOAD_CONFIG.SITE_URL` | `'https://bin.mudfish.net'` | modules/config.js |
| `UPLOAD_CONFIG.ENDPOINT` | `'/api/text'` | modules/config.js |
| `UPLOAD_CONFIG.TTL_MINUTES` | `10080` (7 days) | modules/config.js |
| `UPLOAD_CONFIG.TIMEOUT_MS` | `3000` | modules/config.js |

### CONFIG-11 · Message Fetch Config (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `MESSAGE_FETCH_CONFIG.MAX_ADDITIONAL` | `99` | modules/config.js |
| `MESSAGE_FETCH_CONFIG.DEFAULT_COUNT` | `1` | modules/config.js |

### CONFIG-12 · Database Connection Config (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `DB_CONNECTION_CONFIG.MAX_POOL_SIZE` | `10` | modules/config.js |
| `DB_CONNECTION_CONFIG.MIN_POOL_SIZE` | `2` | modules/config.js |
| `DB_CONNECTION_CONFIG.SERVER_SELECTION_TIMEOUT_MS` | `5_000` | modules/config.js |
| `DB_CONNECTION_CONFIG.SOCKET_TIMEOUT_MS` | `30_000` | modules/config.js |
| `DB_CONNECTION_CONFIG.MAX_IDLE_TIME_MS` | `60_000` | modules/config.js |
| `DB_RETRY_CONFIG.MAX_ATTEMPTS` | `3` | modules/config.js |
| `DB_RETRY_CONFIG.BASE_DELAY_MS` | `1_000` | modules/config.js |
| `DB_RETRY_CONFIG.MAX_DELAY_MS` | `5_000` | modules/config.js |
| `DB_VECTOR_SEARCH_CONFIG.NUM_CANDIDATES_MULTIPLIER` | `10` | modules/config.js |
| `DB_VECTOR_SEARCH_CONFIG.DEFAULT_LIMIT` | `4` | modules/config.js |
| `DB_VECTOR_SEARCH_CONFIG.SCORE_THRESHOLD` | `0.72` | modules/config.js |

### CONFIG-13 · Memory System Settings (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `MEMORY_RECENT_WINDOW` | `10` | modules/config.js |
| `MEMORY_MAX_RAG_RESULTS` | `3` | modules/config.js |
| `MEMORY_SCORE_THRESHOLD` | `0.72` | modules/config.js |
| `MEMORY_TIME_GAP_MS` | `30_000` (30s) | modules/config.js |
| `MEMORY_RAG_CUTOFF_MS` | `300_000` (5 min) | modules/config.js |
| `MEMORY_MAX_INLINE_CTX` | `6_000` chars | modules/config.js |

### CONFIG-14 · Memory Cache Settings (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `MEMORY_CACHE_TTL_MS` | `120_000` (2 min) | modules/config.js |
| `MEMORY_CACHE_MAX_SIZE` | `200` | modules/config.js |
| `MEMORY_CACHE_MIN_QUERY_LEN` | `10` chars | modules/config.js |
| `MEMORY_CACHE_SEMANTIC_SIM` | `0.92` | modules/config.js |
| `MEMORY_CHUNK_SIZE` | `8` | modules/config.js |
| `MEMORY_CHUNK_OVERLAP` | `2` | modules/config.js |
| `MEMORY_INDEX_BATCH_SIZE` | `3` | modules/config.js |
| `MEMORY_PERSONAL_CACHE_TTL_MS` | `300_000` (5 min) | modules/config.js |

### CONFIG-15 · Cluster Engine Settings (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `CLUSTER_MAX` | `20` | modules/config.js |
| `CLUSTER_NUM_BASELINE` | `5` | modules/config.js |
| `CLUSTER_MIN_MEMORIES` | `30` | modules/config.js |
| `CLUSTER_TOP_TO_SEARCH` | `2` | modules/config.js |
| `CLUSTER_MIN_SIMILARITY` | `0.45` | modules/config.js |
| `CLUSTER_REINDEX_INTERVAL` | `150` messages | modules/config.js |
| `CLUSTER_MAX_KMEANS_ITERS` | `10` | modules/config.js |
| `CLUSTER_CONVERGENCE_THRESHOLD` | `0.001` | modules/config.js |
| `CLUSTER_CACHE_TTL_MS` | `900_000` (15 min) | modules/config.js |
| `CLUSTER_MAX_PER_CLUSTER` | `8` | modules/config.js |
| `CLUSTER_EMBEDDINGS_TTL_MS` | `120_000` (2 min) | modules/config.js |
| `CLUSTER_EMBEDDING_LIMITS.CLUSTER_SAMPLE` | `300` | modules/config.js |
| `CLUSTER_EMBEDDING_LIMITS.CLUSTER_TIME_BUCKETS` | `6` | modules/config.js |
| `CLUSTER_EMBEDDING_LIMITS.FALLBACK_SEARCH` | `30` | modules/config.js |

### CONFIG-16 · Embedding Service Settings (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `EMBEDDING_MODEL` | `'gemini-embedding-2'` | modules/config.js |
| `EMBEDDING_CACHE_MAX_SIZE` | `50` | modules/config.js |
| `EMBEDDING_MAX_CONCURRENT` | `3` | modules/config.js |
| `EMBEDDING_DIM` | `768` | modules/config.js |
| `EMBEDDING_MRL_SHORT_DIM` | `256` | modules/config.js |
| `EMBEDDING_MRL_CENTROID_DIM` | `64` | modules/config.js |
| `EMBEDDING_REDIS_TTL` | `86400` (24h) | modules/config.js |
| `EMBEDDING_REDIS_PREFIX` | `'lumin:emb:'` | modules/config.js |
| `EMBEDDING_ENABLE_PDF` | `false` | modules/config.js |
| `EMBEDDING_ENABLE_VIDEO` | `false` | modules/config.js |
| `EMBEDDING_ENABLE_AUDIO` | `false` | modules/config.js |
| `EMBEDDING_IMAGE_LIMIT.MAX_COUNT` | `6` | modules/config.js |
| `EMBEDDING_PDF_LIMIT.MAX_COUNT` | `1` | modules/config.js |
| `EMBEDDING_PDF_LIMIT.MAX_PAGES` | `6` | modules/config.js |
| `EMBEDDING_VIDEO_LIMIT.MAX_COUNT` | `1` | modules/config.js |
| `EMBEDDING_VIDEO_LIMIT.MAX_SECONDS` | `120` | modules/config.js |
| `EMBEDDING_AUDIO_LIMIT.MAX_COUNT` | `1` | modules/config.js |
| `EMBEDDING_AUDIO_LIMIT.MAX_SECONDS` | `80` | modules/config.js |

### CONFIG-17 · Cache Layer Control (only L3 exposed, L1/L2 inaccessible) (User-Reported)
| Setting | Current Value | Location |
|---|---|---|
| `CACHE_ENABLED` (Redis L3) | `false` | modules/config.js |
| L1 in-memory query cache | always on | MemorySystem.js |
| L2 in-memory embedding cache | always on | EmbeddingService.js |
| `MEMORY_CACHE_TTL_MS` (L1 TTL) | `120_000` | modules/config.js |
| `MEMORY_CACHE_MAX_SIZE` (L1 size) | `200` | modules/config.js |
| `EMBEDDING_REDIS_TTL` (L3 TTL) | `86400` | modules/config.js |

### CONFIG-18 · Session Summary Settings (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `BOT_MSGS_PER_SUMMARY` | `50` | SessionSummaryJob.js (hardcoded) |
| `MAX_MSGS_IN_WINDOW` | `120` | SessionSummaryJob.js (hardcoded) |
| `DAILY_DIGEST_CUTOFF_MS` | `86_400_000` (24h) | SessionSummaryJob.js (hardcoded) |
| `REDIS_SESSION_TTL` | `25h` | SessionSummaryJob.js (hardcoded) |
| Session summary model | `'gemma-3-12b-it'` (raw string, BUG-12) | SessionSummaryJob.js |
| Weekly summary model | `'gemini-3.1-flash-lite'` (raw string, BUG-11) | WeeklySummaryJob.js |
| `WEEKLY_SUMMARY_ENABLED` | `true` | modules/config.js (feature flag) |

### CONFIG-19 · File Processing Timeouts (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `PROCESSING_TIMEOUTS.VIDEO_WAIT_MS` | `10_000` | FileValidator.js (hardcoded) |
| `PROCESSING_TIMEOUTS.GIF_WAIT_MS` | `5_000` | FileValidator.js (hardcoded) |
| `PROCESSING_TIMEOUTS.MAX_ATTEMPTS` | `60` | FileValidator.js (hardcoded) |
| `FILE_NAME_MAX_LENGTH` | `100` | FileValidator.js (hardcoded) |

### CONFIG-20 · Migration Config (partially exposed — only enable/disable)
| Setting | Current Exposure | Notes |
|---|---|---|
| `MIGRATION_CONFIG.ENABLE_MIGRATION` | exposed (toggle) | |
| `MIGRATION_CONFIG.BATCH_SIZE` | **NOT exposed** | 50 hardcoded |
| `MIGRATION_CONFIG.BATCH_DELAY_MS` | **NOT exposed** | 100ms hardcoded |

---

## SECTION 3 — COMMAND-SPECIFIC MODEL CONTROL (User-Reported)

Every command uses a hardcoded model constant — none are configurable from the dashboard. The dashboard must expose a "Command Models" table.

| Command File | Model Used | Type |
|---|---|---|
| `commands/search.js` | Fallback chain: `gemini-3.1-flash-lite` → `gemini-3.5-flash` → `gemma-4-26b-a4b-it` | Hardcoded array |
| `commands/summary/SummaryExecutor.js` | `DEFAULT_MODEL` (`gemini-3.1-flash-lite`) | Hardcoded via `DEFAULT_MODEL` |
| `commands/summary/WeeklySummaryJob.js` | `'gemini-3.1-flash-lite'` | **Raw string literal** (BUG-11) |
| `commands/summary/SessionSummaryJob.js` | `'gemma-3-12b-it'` | **Raw string literal** (BUG-12) |
| `commands/fun/DigestHandler.js` | `MODELS['gemini-3.5-flash']` + `DEFAULT_MODEL` fallback | Hardcoded `FUN_MODEL` |
| `commands/fun/AnniversaryHandler.js` | `MODELS['gemini-3.5-flash']` + `DEFAULT_MODEL` fallback | Hardcoded `FUN_MODEL` |
| `commands/fun/ComplimentHandler.js` | `MODELS['gemini-3.5-flash']` + `DEFAULT_MODEL` fallback | Hardcoded `FUN_MODEL` |
| `commands/fun/StarterHandler.js` | `MODELS['gemini-3.5-flash']` + `DEFAULT_MODEL` fallback | Hardcoded `FUN_MODEL` |
| `commands/birthday/BirthdayScheduler.js` | `MODELS['gemini-3.5-flash']` + `DEFAULT_MODEL` fallback | Hardcoded `BIRTHDAY_MODEL` |
| `commands/quote/QuoteScheduler.js` | `MODELS['gemini-3.5-flash']` + `DEFAULT_MODEL` fallback | Hardcoded `QUOTE_MODEL` |
| `commands/game/Akinator.js` | `DEFAULT_MODEL` | `GAME_MODEL` |
| `commands/game/WouldYouRather.js` | `DEFAULT_MODEL` | `GAME_MODEL` |
| `commands/game/NeverHaveIEver.js` | `DEFAULT_MODEL` | `GAME_MODEL` |
| `commands/game/TruthDareSnap.js` | `DEFAULT_MODEL` | `GAME_MODEL` |

**Add:** `runtimeConfig.commandModels` object keyed by command name. Each command reads its model from `getFlag('commandModels.search.primary')` etc., with fallback to hardcoded default.

---

## SECTION 4 — INDIVIDUAL TOOL TOGGLE CONTROL (User-Reported)

### TOOL-01 · Global On/Off Is the Only Control — No Per-Tool Toggle
`ENABLE_FUNCTION_CALLING` kills all 28 tools or none. The dashboard must expose individual toggles for:

| Tool Name | Description |
|---|---|
| `manage_personal_memory` | Store/remove user memories |
| `manage_server_fact` | Store/remove server-wide facts |
| `search_memory` | Vector search personal memories |
| `check_sessions` | Search old session summaries |
| `set_reminder` | Schedule reminders |
| `set_birthday` | Store birthday data |
| `set_timezone` | Store user timezone |
| `check_time_elapsed` | Time since last message |
| `get_message_timestamp` | Get timestamp of a message |
| `get_current_datetime` | Get current time |
| `search_gif` | GIPHY GIF search |
| `get_server_emojis` | List server emoji |
| `get_server_stickers` | List server stickers |
| `check_user_profile` | Look up Discord user profile |
| `create_poll` | Create a Discord poll |
| `send_dm` | Send a DM to a user |
| `send_server_message` | Send to a channel |
| `edit_bot_message` | Edit a previous bot message |
| `delete_bot_message` | Delete a bot message |
| `pin_message` | Pin a message |
| `create_thread` | Create a thread |
| `add_reaction` | Add an emoji reaction |
| `get_server_info` | Get server metadata |
| `get_channel_info` | Get channel metadata |
| `fetch_meme` | Fetch a meme (multi-source) |
| `search_giphy_sticker` | GIPHY sticker search |
| `google_search` | Grounded web search |
| `ignore_user` | Add user to ignore list |

**Implementation:** Store disabled set in `runtimeConfig.disabledTools[]`. In `FunctionRegistry.js`, filter `functionTools` against the disabled set at call time. Each toggle must take effect immediately (no restart).

### TOOL-02 · `search_memory` Sub-Settings Not Configurable
The `search_memory` tool behaviour is governed by:
- `MEMORY_MAX_RAG_RESULTS = 3` — how many vector hits are injected per prompt
- `MEMORY_SCORE_THRESHOLD = 0.72` — minimum similarity to include
- `MEMORY_RAG_CUTOFF_MS = 300_000` — dedup exclusion window (5 min)
- `ENABLE_RAG` — whether auto-RAG fires on every message (already exposed as flag, but not linked to `search_memory` in UI)
None of these are labeled as "search_memory settings" in any UI.

### TOOL-03 · `fetch_meme` Tool Sub-Config (ZERO dashboard exposure)
| Setting | Current Value | Location |
|---|---|---|
| `MEME_API_TIMEOUT` | `8000` ms | FunctionExecutor.js (hardcoded) |
| `MEME_API_BATCH` | `15` posts per subreddit | FunctionExecutor.js (hardcoded) |
| `RECENT_WINDOW` dedup | `15` memes | FunctionExecutor.js (hardcoded) |
| Enable meme-api.com source | always on | cannot disable |
| Enable Reddit direct source | always on | cannot disable |
| Enable GIPHY meme source | on if `GIPHY_API_KEY` set | cannot disable independently |
| `TOPIC_TO_SUBS` mapping | 100+ entries hardcoded | FunctionExecutor.js |

**Add:** A "Meme Tool Config" sub-section with source toggles, timeout/batch sliders, and an editable topic→subreddit mapping table.

### TOOL-04 · `search_gif` / `search_giphy_sticker` Not Independently Toggleable
Both GIPHY tools share the same `GIPHY_API_KEY`. They can only be enabled/disabled together via the `ENABLE_FUNCTION_CALLING` global. Cannot disable GIF search while keeping sticker search enabled (or vice versa).

---

## SECTION 5 — COMMAND ENABLE/DISABLE AND TIMED LOCKDOWN (User-Reported)

### CMD-01 · No Per-Command Enable/Disable
The only disable mechanism is the global `isGlobalLockdown()` (all-or-nothing). There is no ability to disable individual commands:
- `/search` (search.js)
- `/summary` (SummaryExecutor.js / SummaryHandler.js)
- `/birthday` (BirthdayHandler.js / BirthdayScheduler.js)
- `/reminder` (ReminderHandler.js / ReminderScheduler.js)
- `/fun` (DigestHandler, AnniversaryHandler, ComplimentHandler, StarterHandler, RouletteHandler)
- `/game` (Akinator, WouldYouRather, NeverHaveIEver, TruthDareSnap)
- `/quote` (QuoteHandler / QuoteScheduler)
- `/realive` (realive.js)
- `/timezone` (timezone.js)

**Add:** `runtimeConfig.disabledCommands[]` array. In each command handler's entry point (before processing), check `getFlag('disabledCommands').includes(commandName)` and reply with a "disabled" embed.

### CMD-02 · Lockdown Has No Timed/Scheduled Option (User-Reported)
`isGlobalLockdown()` is a plain boolean (`state.globalLockdown = true/false`). There is no:
- `lockdownUntil: <timestamp>` for auto-expiry
- `lockdownDuration: <minutes>` input in the UI
- Countdown timer showing when lockdown will end
- Scheduled unlock

**Add:** A `lockdownUntil` field in runtimeConfig. A dashboard "Lock for X minutes" input. A setInterval that clears lockdown when `Date.now() > lockdownUntil`.

---

## SECTION 6 — MISSING FEATURES (USER-REPORTED + CODE-ANALYSIS)

### FEAT-01 · Per-Model API Usage Breakdown Not Shown (User-Reported)
`getApiKeyStats()` returns `modelStats[]` per key (requestsThisMinute, rateLimited, cooldown, secondsUntilReset) but `renderApiKeysPanel()` never renders it. Data exists — UI ignores it.

### FEAT-02 · Embedding Model Usage Not Tracked (User-Reported)
`EmbeddingService.js` uses `EMBEDDING_MODEL` (`gemini-embedding-2`) via the same `GOOGLE_API_KEY*` keys. These calls are never counted in `keyUsageStats`. Key request counts in dashboard understate actual usage.

### FEAT-03 · Giphy API Usage Not Tracked or Shown (User-Reported)
`FunctionExecutor.js` uses `GIPHY_API_KEY` for `search_gif`, `fetch_meme` (source 3), and `search_giphy_sticker`. Zero tracking of call counts, success/fail, or quota consumption.

### FEAT-04 · Thinking Mode Not Configurable (User-Reported)
Thinking is hardcoded in `getGemmaConfig()`, `getGemini3Config()`, `getGemini2Config()`. No UI to enable/disable per model family or set level/budget.

### FEAT-05 · Migration Shows Checkboxes but No Values or Preview (User-Reported)
Checkboxes show field names only (e.g. `selectedModel`) without the default value that would be written, affected count, or run history.

### FEAT-06 · No Personality / System Prompt Dedicated Editor
The `defaultPersonality` in `config.js` is 5,000+ words. Currently only editable by opening the raw `config.js` file in the config editor — the entire 350-line file.

### FEAT-07 · No Activities Editor
`config.js` → `activities[]` array. Dashboard shows activity preset buttons but has no add/edit/remove UI.

### FEAT-08 · Redis Status and Controls Missing
No Redis health indicator (connected / disconnected / not configured), no cache flush button, no key count display.

### FEAT-09 · Reminders and Birthdays Are View-Only
No `PUT`/`POST` endpoints to create, edit, or individually delete reminders or birthday entries from the dashboard.

### FEAT-10 · No Per-Server Model Override Bulk View
Each server can have `selectedModel` in `serverSettings`. No bulk table showing which servers have model overrides vs using default.

---

## SECTION 7 — ORGANISATION / STRUCTURE (User-Reported)

### ORG-01 · Models Page Is Massively Overloaded (User-Reported)
Current `section-models` contains: model cards, API key panel, 13 feature flag toggles, 5 media toggles, 11 bot config fields, 7 rate-limit fields, migration config, field selection checkboxes. Six distinct functional areas crammed onto one page.

**Proposed new navigation structure:**
```
Overview    : Stats, health, quick actions
Servers     : Server list, per-server settings
Users       : User list, per-user settings
─────────────────────────────────────────────
AI          : Models / Thinking / Safety settings  
API Keys    : Key stats, rotation, per-model usage, Giphy status
Feature Flags: All toggles (live vs restart-required labelled)
Function Tools: Per-tool enable/disable + sub-config
─────────────────────────────────────────────
Commands    : Per-command model, enable/disable, rate limits
Cache       : L1/L2/L3 controls, stats, flush
Memory      : RAG settings, cluster config, embedding config
─────────────────────────────────────────────
Bot Settings: Queue, timeouts, generation params, session summary
Migration   : Dedicated page with preview + history
Lockdown    : Timed lockdown, command disabling
─────────────────────────────────────────────
Config Editor: Per-file editors (personality, activities, raw JSON)
Database    : MongoDB status, collection sizes, tools
Advanced    : Node.js, files, shell
```

### ORG-02 · Migration Must Be Its Own Page (User-Reported)
High-stakes destructive operation buried at the bottom of Models page.

### ORG-03 · API Key Management Must Be Its Own Page (User-Reported)
Needs: all keys with full stats, model breakdown, daily counts, rotation controls, cooldown timers, Giphy key status.

### ORG-04 · Admin Commands Grid Has No Search or Categories
38 command cards in a flat unsearchable grid. Needs a filter input and category grouping (User Management, Server, Usage/Stats, Bot Control).

---

## SECTION 8 — UI BUGS & INCONSISTENCIES

### UI-01 · `overview.js` Is 34 Bytes (Empty)
### UI-02 · `servers.js` Is 390 Bytes (Mostly Empty)  
### UI-03 · `announce.js` Is a Stub
### UI-04 · `totalServers_s` Sent in Stats Stream But Never Rendered
### UI-05 · Modals Use Inline `style=` Attributes Instead of CSS Classes
### UI-06 · `_saveRuntimeConfig` Quick-Save Only Saves 2 of Many Fields
`activeModel` and `globalEmbedColor` only — user has no idea the form silently discards everything else.
### UI-07 · No Unified "Restart Required" Banner
Some saves say "restart", some say "next request", some say nothing.
### UI-08 · `clearAllUsage` Uses Native `confirm()` Instead of `toastConfirm()`
### UI-09 · `restart`, `clearReminders`, `clearBirthdays` Also Use Native `confirm()`
### UI-10 · `renderApiKeysPanel` Called From 3 Places — Stale Data Risk
### UI-11 · Model Cards Don't Differentiate Gemma vs Gemini Visually
No indication which models need `ENABLE_GEMMA: true` to be usable.
### UI-12 · Server Settings and User Settings Use Different Edit Patterns
Server settings → labeled form fields. User settings → raw JSON textarea. Should be unified.
### UI-13 · `_saveRuntimeConfig` Raw JSON Editor and Quick-Save Form Both Exist — Silently Conflict
Changes via quick-save don't update the JSON textarea; changes via textarea don't update quick-save fields. Two unsynchronized paths to the same data.

---

## SECTION 9 — PERFORMANCE & SECURITY

### PERF-01 · `getDiskUsage()` Runs `execSync("df -h")` Every Second Per Connected Client
**File:** `dashboard/server.js` → `handleStatsStream()`
Child process spawn per second per client = expensive under any load.
**Fix:** Cache disk usage for 30 seconds minimum.

### PERF-02 · `loadModels()` Makes 7 Sequential API Calls
Should be `Promise.all([getModels(), getApiKeyStats(), getFeatureFlags(), getMigrationConfig(), getBotConfig(), getRateLimits(), getMigrateFields()])`.

### SEC-01 · File Browser Write Has No Extension Allowlist
**File:** `dashboard/server.js` → `/api/files` (PUT)
Allows writing any file within `ROOT_DIR` including new `.js` files that will be `import()`'d on next restart.

---

## COMPLETE ISSUE SUMMARY TABLE

| ID | Category | Severity | Title |
|---|---|---|---|
| DEPLOY-01 | Architecture | **Critical** | runtime-config.json wiped on every Render redeploy |
| BUG-01 | Data | High | `dailyRequests` field missing — always 0 |
| BUG-02 | Data | High | `errorCount` vs `errors` — errors always 0 |
| BUG-03 | UI | Medium | Stat grid duplicate totalHistories entry |
| BUG-04 | Data | High | Server settings modal wrong/missing field names |
| BUG-05 | Data | Medium | allHistories message count always 0 |
| BUG-06 | Logic | Medium | Settings save drops falsy/empty values |
| BUG-07 | Logic | **Critical** | Feature flag toggle silent no-op for 9/13 flags |
| BUG-08 | Logic | High | ENABLE_GEMMA toggle has no runtime effect |
| BUG-09 | Persistence | Medium | Per-model rate-limit windows lost on restart |
| BUG-10 | Config | High | imageConfig missing — image limit hardcoded to 10 |
| BUG-11 | Code | Medium | WeeklySummaryJob uses raw model string literal |
| BUG-12 | Code | Medium | SessionSummaryJob uses raw model string literal |
| BUG-13 | Code | Low | Dead auth token stub code |
| CONFIG-01 | Config | High | AI routing settings not all editable |
| CONFIG-02 | Config | High | Generation params (temp, topP, thinking) not editable |
| CONFIG-03 | Config | Medium | Safety settings not configurable |
| CONFIG-04 | Config | High | Queue/processing/retry limits all hardcoded |
| CONFIG-05 | Config | Medium | Image/summary rate limits hardcoded |
| CONFIG-06 | Config | Medium | State and bot behavior constants not editable |
| CONFIG-07 | Config | Medium | Media processing — category-only toggles |
| CONFIG-08 | Config | Medium | No per-format attachment enable/disable |
| CONFIG-09 | Config | Low | Poll config not exposed |
| CONFIG-10 | Config | Low | Upload/pastebin config not exposed |
| CONFIG-11 | Config | Low | Message fetch config not exposed |
| CONFIG-12 | Config | Low | Database connection config not exposed |
| CONFIG-13 | Config | Medium | Memory system settings not exposed |
| CONFIG-14 | Config | Medium | Memory cache settings not exposed |
| CONFIG-15 | Config | Medium | Cluster engine settings not exposed |
| CONFIG-16 | Config | Medium | Embedding service settings not exposed |
| CONFIG-17 | Config | Medium | L1/L2/L3 cache no granular control |
| CONFIG-18 | Config | Medium | Session summary settings not exposed |
| CONFIG-19 | Config | Low | File processing timeouts not exposed |
| CONFIG-20 | Config | Low | Migration batch size/delay not exposed |
| CMD-MODEL-01..14 | Config | High | 14 command files use hardcoded model constants |
| TOOL-01 | Feature | High | No per-tool function toggle |
| TOOL-02 | Feature | Medium | search_memory sub-settings not configurable |
| TOOL-03 | Feature | Medium | fetch_meme sub-config not exposed |
| TOOL-04 | Feature | Low | search_gif and sticker not independently toggleable |
| CMD-01 | Feature | High | No per-command enable/disable |
| CMD-02 | Feature | Medium | Lockdown has no timed/scheduled option |
| FEAT-01 | Feature | High | Per-model API usage breakdown not shown |
| FEAT-02 | Feature | High | Embedding model usage not tracked |
| FEAT-03 | Feature | High | Giphy API usage not tracked |
| FEAT-04 | Feature | High | Thinking mode not configurable |
| FEAT-05 | Feature | Medium | Migration shows no values or preview |
| FEAT-06 | Feature | Medium | No dedicated personality editor |
| FEAT-07 | Feature | Low | No activities editor |
| FEAT-08 | Feature | Medium | No Redis status/flush controls |
| FEAT-09 | Feature | Low | Reminders/birthdays view-only |
| FEAT-10 | Feature | Low | No per-server model override bulk view |
| ORG-01 | Structure | High | Models page massively overloaded |
| ORG-02 | Structure | High | Migration needs its own page |
| ORG-03 | Structure | High | API Keys needs its own page |
| ORG-04 | Structure | Medium | Admin commands no search or categories |
| UI-01..13 | UI | Low–Med | Various UI inconsistencies (13 items) |
| PERF-01 | Performance | Medium | execSync every second per client |
| PERF-02 | Performance | Low | loadModels 7 sequential API calls |
| SEC-01 | Security | Medium | File browser write no extension allowlist |

**Total: 67 distinct issues**
- 1 critical architecture (deploy persistence)
- 14 bugs (2 critical, 6 high, 4 medium, 2 low)
- 20 missing config controls
- 14 command model hardcodes
- 4 tool control gaps
- 2 command control gaps
- 10 missing features
- 4 structure/org
- 13 UI inconsistencies
- 2 performance, 1 security

