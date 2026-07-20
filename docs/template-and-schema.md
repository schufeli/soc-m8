# SOC-M8 - Template & Configuration Format

`schemaVersion: 1` · draft

This document defines the two files SOC-M8 reads and writes:

| File | `kind` | Contains | Source |
| --- | --- | --- | --- |
| Template file | `soc-m8-templates` | Categories, templates, variants, placeholders | Shipped default, remote URL, or local editor export |
| User config | `soc-m8-config` | Pins, favourites, quick slots, local templates, settings | Local browser storage; user-exportable |

Both carry a `kind` discriminator so import paths can reject the wrong file with a specific error ("this looks like a template file, not a config backup") instead of a schema dump.

---

## 1. Template file

```json
{
  "kind": "soc-m8-templates",
  "schemaVersion": 1,
  "name": "ACME SOC Templates",
  "updated": "2026-07-20",
  "categories": [
    { "id": "malware", "label": "Malware Analysis" },
    { "id": "identity", "label": "Identity & Access" }
  ],
  "templates": []
}
```

### 1.1 Template object

```json
{
  "id": "entra-signin-review",
  "category": "identity",
  "title": "Entra ID sign-in log review",
  "aliases": ["entra", "azure ad", "aad", "signin", "sign-in"],
  "tags": ["logs", "identity"],
  "placeholders": [
    { "key": "user", "label": "Affected user", "type": "email", "required": true }
  ],
  "variants": [
    {
      "id": "clean",
      "label": "No findings",
      "body": "Entra ID sign-in logs for {{user}} were reviewed - no indicators of malicious behaviour found."
    },
    {
      "id": "suspicious",
      "label": "Suspicious sign-ins found",
      "body": "Entra ID sign-in logs for {{user}} show successful sign-ins from {{country}} at {{time}}, inconsistent with the user's normal pattern.",
      "placeholders": [
        { "key": "country", "label": "Origin country", "type": "text", "required": true },
        { "key": "time", "label": "Sign-in time", "type": "datetime" }
      ]
    },
    {
      "id": "unavailable",
      "label": "Logs unavailable",
      "body": "Entra ID sign-in logs for {{user}} could not be reviewed - no log data available for the relevant period."
    }
  ]
}
```

**Fields**

| Field | Required | Notes |
| --- | --- | --- |
| `id` | yes | Stable, unique within the file. Treat as permanent - see §3.3. |
| `category` | yes | Must match a declared category `id`. |
| `title` | yes | Shown in the panel; primary search field. |
| `aliases` | no | Hidden search terms. Product renames, abbreviations, colloquialisms. |
| `tags` | no | Secondary search terms; also usable as filters. |
| `placeholders` | no | Template-level, shared by all variants. |
| `variants` | one of | Two or more alternative bodies. |
| `body` | one of | Shorthand for a single-variant template. |

### 1.2 Variant groups

A template has **either** a `body` (simple case) **or** a `variants` array (two or more outcomes for the same finding).

At load time both normalise to the same internal shape: `body: "..."` becomes a single variant with `id: "default"`. Authors get the short form, the runtime only ever handles one structure.

Rules:

- Variant `id` must be unique within its template. The addressable identity of a variant is `templateId#variantId` - this is what pins and quick slots bind to (§3).
- Variants inherit the template's `placeholders` and may declare additional ones. A variant may not redefine an inherited `key`.
- **Only placeholders actually referenced in the chosen variant's body are prompted for.** In the example above, picking "No findings" asks for `user` only; picking "Suspicious sign-ins found" asks for `user`, `country`, and `time`. Unused declarations are never shown.
- Variant order in the file is the display order. Put the most common outcome first.
- Search matches against the template's `title`/`aliases`/`tags` and each variant's `label` and `body`, so typing "logs unavailable" jumps straight to that variant.

### 1.3 Placeholders

```json
{
  "key": "url",
  "label": "Malicious URL",
  "type": "url",
  "required": true,
  "transform": ["defang"],
  "default": "{{clipboard}}",
  "hint": "Paste the full URL including scheme"
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `key` | yes | Referenced as `{{key}}`. Lowercase, `[a-z0-9_]`. |
| `label` | yes | Field label in the fill-in form. |
| `type` | no | Default `text`. Drives validation and input widget. |
| `required` | no | Default `false`. Blocks copy while empty. |
| `transform` | no | Ordered list applied to the value on output. |
| `default` | no | Literal, or a dynamic token (`{{clipboard}}`, `{{date}}`, `{{time}}`). |
| `hint` | no | Helper text under the field. |
| `options` | enum only | Array of `{value, label}`. |
| `pattern` | text only | Custom regex, for site-specific formats like ticket IDs. |

#### Types

| Type | Validation | Input widget |
| --- | --- | --- |
| `text` | none, or `pattern` | single line |
| `textarea` | none | multi-line |
| `number` | numeric; optional `min`/`max` | number input |
| `hash` | 32, 40, or 64 hex chars (MD5 / SHA-1 / SHA-256) | single line, detected subtype shown as a badge |
| `sha256` | exactly 64 hex chars | single line |
| `ip` | valid IPv4 or IPv6 | single line |
| `domain` | valid hostname | single line |
| `url` | scheme + host; accepts already-defanged input | single line |
| `email` | RFC-ish local@domain | single line |
| `date` / `time` / `datetime` | parseable | native picker |
| `enum` | must be one of `options` | dropdown |

Validation is **advisory, not blocking** - invalid input shows an inline warning and a copy button that says "Copy anyway". Real incidents contain malformed data, and a tool that refuses to let you write down what you actually saw is a tool people stop using. `required` is the only hard block, and only on emptiness.

Validation runs against the **raw input**; transforms are applied to the **output**. So a `url` field validates `https://evil.example.com` normally, and emits `hxxps://evil[.]example[.]com`.

#### Transforms

Defined on the template by the author, applied automatically at output. The analyst doesn't choose per use.

| Transform | Effect |
| --- | --- |
| `defang` | `http`→`hxxp`, `.`→`[.]`, `@`→`[@]`, `://`→`[://]` |
| `refang` | inverse of `defang`, for pasting into tooling |
| `upper` / `lower` | case |
| `trim` | strip surrounding whitespace (implicit on every field) |

Applied in array order. `["trim", "defang", "lower"]` is left-to-right.

`defang` is idempotent and input-tolerant: already-defanged input passes through unchanged rather than double-defanging into `hxxp[[.]]`. Implement it as refang-then-defang internally.

---

## 2. Search

The search index is built once per template load, and scores across:

| Field | Weight |
| --- | --- |
| `title` | highest |
| `aliases` | highest |
| variant `label` | high |
| `tags` | medium |
| category label | medium |
| variant `body` | low |

Matching is fuzzy (subsequence with a typo allowance), so "vtsh" reaches "VirusTotal SHA256 verdict". Body text is indexed at low weight because analysts often remember the sentence but not the title - but it must never outrank a title match.

Ties break on usage frequency, then recency. Pinned items sort above unpinned at equal relevance; they do not bypass relevance entirely.

Aliases are free to collide across templates ("vt" on three VirusTotal templates is fine and expected) - ranking sorts it out.

---

## 3. User config

Local, never uploaded, exportable as a single JSON backup.

```json
{
  "kind": "soc-m8-config",
  "configVersion": 1,
  "exported": "2026-07-20T09:12:00Z",
  "settings": {
    "remoteUrl": "https://intranet.example.com/soc/templates.json",
    "outputFormat": "plaintext",
    "density": "compact"
  },
  "favourites": ["vt-hash-verdict#malicious", "entra-signin-review#clean"],
  "pins": ["ransomware-containment#initial"],
  "quickSlots": {
    "1": "entra-signin-review#clean",
    "2": "vt-hash-verdict#malicious",
    "3": null
  },
  "localTemplates": { "kind": "soc-m8-templates", "schemaVersion": 1, "templates": [] },
  "usage": { "vt-hash-verdict#malicious": 214, "entra-signin-review#clean": 187 }
}
```

### 3.1 Favourites vs pins

Deliberately separate:

- **Favourites** - a durable personal shortlist. Filterable as its own view.
- **Pins** - a small, volatile set stuck to the top of the panel. Meant for the campaign you're working *this week*, and expected to be cleared often.

### 3.2 Quick slots

`Alt+1` … `Alt+9`, each bound to a `templateId#variantId`.

A slot with unfilled placeholders opens the fill-in form with the first field focused, rather than refusing to fire. The keystroke is a shortcut to *the template*, not to a finished string - otherwise slots are only usable for the trivial templates, which are also the easiest to find by search anyway.

Slot bindings are user config, so they survive a template file update and follow the analyst to a new machine via export.

### 3.3 Dangling references

Pins, favourites, quick slots, and usage counters all reference `templateId#variantId` in a file the user may not control. When the remote set changes, references can break.

Two consequences:

**For template authors:** template and variant `id`s are a public contract. Change a `title` freely; never recycle or rename an `id`. Retiring a template means removing it, not repurposing its `id` for something else - a recycled `id` silently rebinds someone's `Alt+3` to unrelated text.

**For the client:** a reference to a missing template is **kept, not deleted**. It renders greyed out and unavailable, with a tooltip explaining the template is no longer in the current set. If the template returns - a reverted remote file, a reconnected server, a restored cache - the binding re-links automatically. Silently dropping bindings on a transient fetch failure would quietly destroy a user's setup.

### 3.4 Import

On import, validate `kind` and `configVersion` first, then merge or replace at the user's choice. A config file containing `localTemplates` that fail template-schema validation imports the settings and rejects the templates with a specific message, rather than failing wholesale.

---

## 4. Non-persistence

Placeholder *values* are never written to storage. They live in memory for the duration of the fill-in interaction and are discarded on copy or dismiss. Usage counters record *which* template was used, never *what was typed into it*.

This is a stated guarantee of the tool, not an implementation detail - SOC-M8 handles incident data, and "we don't keep what you typed" needs to be true and auditable in the source.
