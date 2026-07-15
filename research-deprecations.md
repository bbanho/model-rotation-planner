# Research: Gemini API Model Deprecations & Rotation Inputs

Source: official Gemini API "Gemini deprecations" page
https://ai.google.dev/gemini-api/docs/deprecations
(Fetched 2026-07-15. Today's date per user: 2026-07-15.)

## What the page is

The deprecations page lists known deprecation schedules for stable (GA) and
preview models. A "deprecation" = announcement that Google no longer supports
a model and will "shut it down" soon. Once "shutdown", the endpoint is gone
(HTTP 404 / 410). Already-shutdown models are shown with gray backgrounds.

Deprecation announcements are made on the Release notes page; earliest shutdown
dates are tracked on the deprecations page.

## Why this matters for rotation

When a requested model is shut down, the gateway currently raises a terminal
`UpstreamError(502)` on HTTP 404 (see `upstream_service.py` `call_static`).
A model-rotation layer should instead transparently fall back to the model's
**recommended replacement** from this table — before failing.

## Recommended-replacement mapping (the rotation backbone)

Extracted from the deprecations table (only the rows relevant to a Flash
proxy are kept; full table is much larger):

### Gemini 3 family (current targets)
| Model | Released | Shutdown | Recommended replacement |
|---|---|---|---|
| `gemini-3.5-flash` | 2026-05-19 | none announced | — (current GA target) |
| `gemini-3.1-flash-lite` | 2026-05-07 | 2027-05-07 | — |
| `gemini-3.1-flash-lite-preview` | 2026-03-03 | 2026-05-25 | `gemini-3.1-flash-lite` |
| `gemini-3.1-flash-image-preview` | 2026-02-26 | none | — |
| `gemini-3.1-pro-preview` | 2026-02-19 | none | — |
| `gemini-3-pro-image-preview` | 2025-11-20 | none | — |
| `gemini-3-flash-preview` | 2025-12-17 | none | `gemini-3.5-flash` |
| `gemini-3-pro-preview` | 2025-11-18 | 2026-03-09 | `gemini-3.1-pro-preview` |

### Gemini 2.5 Flash family (legacy, still alive until Oct 2026)
| Model | Released | Shutdown | Recommended replacement |
|---|---|---|---|
| `gemini-2.5-flash` | 2025-06-17 | 2026-10-16 | `gemini-3.5-flash` |
| `gemini-2.5-flash-image` | 2025-10-02 | 2026-10-02 | `gemini-3.1-flash-image-preview` |
| `gemini-2.5-flash-lite` | 2025-07-22 | 2026-10-16 | `gemini-3.1-flash-lite` |
| `gemini-2.5-flash-lite-preview-09-2025` | 2025-09-25 | 2026-03-31 | `gemini-3.1-flash-lite` |
| `gemini-2.5-flash-preview-05-20` | 2025-05-20 | 2025-11-18 | `gemini-3.5-flash` |
| `gemini-2.5-flash-image-preview` | 2025-05-07 | 2026-01-15 | `gemini-2.5-flash-image` |
| `gemini-2.5-flash-preview-09-25` | 2025-09-25 | 2026-02-17 | `gemini-3.5-flash` |

### Gemini 2.0 Flash family (SHUT DOWN 2026-06-01)
| Model | Shutdown | Recommended replacement |
|---|---|---|
| `gemini-2.0-flash` | 2026-06-01 | `gemini-2.5-flash` |
| `gemini-2.0-flash-001` | 2026-06-01 | `gemini-2.5-flash` |
| `gemini-2.0-flash-lite` | 2026-06-01 | `gemini-2.5-flash-lite` |
| `gemini-2.0-flash-lite-001` | 2026-06-01 | `gemini-2.5-flash-lite` |

> Confirmed by user: `gemini-2.0-flash` was shut down 2026-06-01. As of
> 2026-07-15 it returns errors. Must NOT be in any supported-model list.

## Key facts for the rotation design

1. **Preview models deprecate with >= 2 weeks notice.** Stable/GA models get
   longer lead time (e.g. `gemini-2.5-flash` lives until 2026-10-16).
2. **No machine-readable "active/deprecated" flag** in the models list API
   (`GET /v1beta/models`). The forum (discuss.ai.google.dev #169201) confirms
   the response includes deprecated/retired models with no status field. So the
   gateway must maintain its own allow-list / replacement map — cannot rely on
   the live models endpoint to filter.
3. **Replacement is model-specific**, not a single global fallback. A good
   rotation map is `requested_model -> [ordered list of fallbacks]`.
4. Today (2026-07-15) the safe Flash targets are: `gemini-3.5-flash` (GA, no
   shutdown), `gemini-2.5-flash` (alive to 2026-10-16), `gemini-3.1-flash-lite`
   (alive to 2027-05-07), `gemini-2.5-flash-lite` (alive to 2026-10-16).
5. The deprecations page is the authoritative source but is HTML, not an API.
   Rotation logic must key off a **curated, versioned mapping** we maintain,
   refreshed from this page — not scraped at runtime.

## Relevant prior art in this workspace

- `/var/home/bruno/repos/gemini-api-key-rotator-proxy-server` — existing
  Node.js key-rotation proxy (Context7 lib `/p32929/rotato`). Useful reference
  for rotation patterns, but it rotates *keys*, not *models*.
- `nexus-gateway` `upstream_service.py` `call_static` already has a
  `rotated_model_on_this_key` flag and a 503 "blacklist model" path, but the
  model-rotation-on-429/404 logic is currently a stub (references a
  non-existent `_other_model`). This planner designs the real version.
