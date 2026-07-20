# SOC-M8

**Snippet & template manager for security analysts - a browser side panel for the sentences you retype every day.**

SOC-M8 ("SOC mate") is a browser extension that puts your recurring incident-analysis phrasing one keystroke away. Instead of scrolling a Notepad++ scratch file for the sentence you wrote fifty times last month, you hit a shortcut, type three letters, fill in the blanks, and paste.

```
SHA256 {{hash}} was checked on VirusTotal; {{detections}} of {{total}} vendors flagged it as malicious.
Entra ID sign-in logs checked - no indicators of malicious behaviour found.
```

Templates live in a plain JSON file. Ship the defaults, edit your own in the built-in editor, or point the extension at a URL serving a static JSON file so a whole SOC works from the same wording.

---

## Why

Incident documentation is repetitive by design. Analysts write the same handful of findings over and over - VirusTotal verdicts, log-review outcomes, containment steps, closure statements - and consistent phrasing genuinely matters for auditability and handover. In practice that knowledge ends up in a personal text file that nobody else can use and that gets slower to search every week it grows.

General-purpose text expanders solve part of this. SOC-M8 is narrower on purpose:

- **Placeholders are the point.** Real analyst sentences contain numbers and hashes that change every time. A snippet without fillable fields still means hand-editing every paste.
- **Retrieval beats organisation.** Mid-incident you don't browse a category tree, you type "virustotal" and expect the right line instantly.
- **Team templates, self-hosted.** Point the extension at your own URL. No third-party SaaS, no account, no telemetry, no analyst text leaving your environment.
- **Minimal permissions by design.** Security teams vet the extensions they install. SOC-M8 requests access to exactly the origin you configure, and nothing else.

---

## Features

### Core

- **Side panel** that stays open alongside your ticketing system, SIEM, or threat-intel tabs
- **Instant search** with keyboard-first navigation - open, type, select, copy, without touching the mouse
- **Categories** for browsing, plus frequency/recency ranking so your most-used templates surface first
- **Click-to-copy** with clear feedback, and copy-as-plaintext or copy-as-markdown depending on where you're pasting

### Templates with placeholders

- `{{placeholder}}` syntax for the parts that change (hashes, counts, hostnames, timestamps)
- Fill-in form on selection, with tab-through fields and sensible defaults
- `{{clipboard}}` token - copy a hash first, and it drops straight into the template
- `{{date}}` / `{{time}}` tokens for timestamped notes

### Template editor

- **Form-based editor** - name, category, body, placeholders. No hand-written JSON required.
- **Live validation** against the project's JSON Schema: duplicate IDs, empty required fields, malformed placeholders, undefined categories, all surfaced as inline errors
- **Import → edit → export** round-trip: load an existing file (from disk or from your endpoint), edit it, export it back out
- **Export to JSON** ready for deployment to a self-hosted endpoint

### Self-hosted template sources

- Configure a URL to any **static JSON file** - a raw file in an internal Git repo, an S3 object, a path on an internal webserver. How you host it is up to you; SOC-M8 only needs to `GET` it.
- **Read-only by design.** The extension never writes back. No auth flow, no API, no write permissions to configure.
- **Per-origin permissions.** When you save a URL, the extension requests host permission for that one origin - which also means cross-origin fetching works regardless of whether your server sends CORS headers.
- **Test URL button** with specific errors: unreachable, returned HTML instead of JSON, or valid JSON that fails schema validation.
- **Local cache of the last good fetch**, so a server hiccup mid-incident never leaves you with an empty panel.
- **Fetched JSON is treated as untrusted**: schema-validated on every load, inserted as plaintext only (`textContent`, never `innerHTML`), `https://` expected with a warning on plain HTTP.

---

## Template sources & precedence

SOC-M8 reads from up to three sources:

| Source | Role | Editable in-app |
| --- | --- | --- |
| **Shipped defaults** | Starter set, so the extension is useful on first launch | No (copy to local to modify) |
| **Remote (your URL)** | The team's authoritative set | No (read-only) |
| **Local** | Your personal additions and overrides | Yes |

**Precedence:** when a remote URL is configured, the remote set becomes the authoritative team library and shipped defaults step aside. Local templates are layered on top and always remain available. With no URL configured, defaults plus local templates are shown.

The reasoning: a team that has gone to the trouble of hosting a template file wants *their* wording to be the standard, not a merge with whatever the extension happened to ship. Personal snippets are never taken away from the individual analyst.

---

## Template format

Templates are plain JSON, versioned with a `schemaVersion` so future format changes don't break already-deployed files.

```json
{
  "schemaVersion": 1,
  "name": "ACME SOC Templates",
  "updated": "2026-07-20",
  "categories": [
    { "id": "malware", "label": "Malware Analysis" },
    { "id": "identity", "label": "Identity & Access" }
  ],
  "templates": [
    {
      "id": "vt-hash-verdict",
      "category": "malware",
      "title": "VirusTotal hash verdict",
      "body": "SHA256 {{hash}} was checked on VirusTotal; {{detections}} of {{total}} vendors flagged it as malicious.",
      "placeholders": [
        { "key": "hash", "label": "SHA256", "default": "{{clipboard}}" },
        { "key": "detections", "label": "Detections" },
        { "key": "total", "label": "Total vendors", "default": "" }
      ],
      "tags": ["virustotal", "hash", "ioc"]
    },
    {
      "id": "entra-signin-clean",
      "category": "identity",
      "title": "Entra ID sign-in logs - no findings",
      "body": "Entra ID sign-in logs checked - no indicators of malicious behaviour found.",
      "tags": ["entra", "azure", "logs"]
    }
  ]
}
```

The same schema and the same parser are used by the editor, the exporter, and the runtime fetch - so anything the editor produces is guaranteed loadable, and anything loadable is editable.

---

## Hosting your own templates

1. Build your template set in the extension's editor (or by hand, if you prefer).
2. Export to JSON.
3. Serve that file from anywhere reachable by the browser over HTTPS.
4. Paste the URL into SOC-M8's settings, grant the per-origin permission, hit **Test URL**.

That's the whole integration. Versioning it in Git and serving the raw file gives you change history and review for free.

---

## Roadmap

**MVP** - side panel, load bundled JSON, search and categories, click-to-copy.

**v0.2** - placeholders and the fill-in form, `{{clipboard}}`.

**v0.3** - template editor with schema validation, import/export.

**v0.4** - remote URL fetch, per-origin permissions, caching, Test URL.

**Later** - context-aware pre-fill (on a VirusTotal page, read the detection count and hash off the page so the template arrives already filled), keyboard shortcut customisation, Firefox support (`sidebar_action` differs enough from `chrome.sidePanel` to be its own piece of work).

---

## Technical notes

- Manifest V3, `chrome.sidePanel` API
- Chromium-first (Chrome, Edge); Firefox planned
- `optional_host_permissions` - no broad `*://*/*` host access
- No analytics, no telemetry, no external calls other than the template URL you configure
- Storage: `chrome.storage.local` for templates, settings, and the remote cache

---

## Status

Early development. Interfaces, schema, and template format may change before v1.0.

## Contributing

Issues and pull requests welcome - especially additions to the default template set. If you have phrasing your SOC relies on, it probably belongs in the defaults.

## Licence

MIT
