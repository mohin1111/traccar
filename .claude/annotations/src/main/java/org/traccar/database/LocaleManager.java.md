# LocaleManager.java

**Role:** Loads and caches l10n JSON bundles from the filesystem. Used by `NotificationFormatter` to resolve translated strings for notification templates based on the user's preferred language.
**Fits in:** `@Singleton` injected into `NotificationFormatter`. Reads bundles from `Keys.WEB_LOCALIZATION_PATH` (the same `l10n/` directory served to the frontend).
**Read next:** [[NotificationFormatter.java]], [[Config.java]] (`Keys.WEB_LOCALIZATION_PATH`)

## Public API

- `getBundle(String language)` (line 62) — returns `Map<String, String>` for the requested language (lazy-loaded). Falls back to `en.json` if the requested language file doesn't exist. Results are cached in `languageBundles`.
- `getString(String language, String key)` (line 79) — convenience wrapper: `getBundle(language).get(key)`.
- `getTemplateFile(String root, String path, String language, String fileName)` (line 51) — resolves a Velocity template file path with language fallback to `en`. Used by `TextTemplateFormatter` to find email/notification templates.

## Key flows

### Bundle caching (lines 63-77)
`languageBundles.computeIfAbsent(language, ...)` — only one disk read per language per JVM lifetime. The JSON is a flat `Map<String, String>`.

### Language fallback (lines 52-58)
`getTemplateFile` tries the requested language, then `en`. If neither exists, returns `null`.

## Gotchas / non-obvious

- **`sanitizeLanguage`** (line 83) rejects any language string not matching `^[A-Za-z0-9_-]+$` — prevents path traversal via `../../etc/passwd` style inputs.
- **`languageBundles` is `ConcurrentHashMap`** — thread-safe for concurrent first-time loads (Java 8+ `computeIfAbsent` is atomic per key).
- **`path` must exist** at startup — no graceful fallback if the localization directory is missing; `Files.newInputStream` throws `IOException` wrapped in `RuntimeException`.
- **Same bundle files as the web frontend** — `Keys.WEB_LOCALIZATION_PATH` points to the React app's `l10n/` directory. Indian languages: `hi.json`, `ta.json`, `bn.json`, `ml.json`.

## Line index

- 43-48 — `@Inject` constructor: resolves `WEB_LOCALIZATION_PATH`
- 51-59 — `getTemplateFile` (language-aware template path resolution)
- 62-77 — `getBundle` (lazy-load + cache JSON bundle)
- 79-81 — `getString` convenience method
- 83-88 — `sanitizeLanguage` (path traversal prevention)
