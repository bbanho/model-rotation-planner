# Plan: Gemini Model-Rotation Layer

Status: DRAFT for review. Repo: `model-rotation-planner` (separate from
`nexus-gateway`). Research basis: `research-deprecations.md`.

## Goal

Make the gateway resilient to model deprecation/shutdown by transparently
falling back to a recommended replacement model when the requested model is
unavailable — instead of returning a terminal 502.

Two distinct "rotation" concepts must not be conflated:
- **Key rotation** (exists): cycle API keys on 429/401. Done in `key_manager`.
- **Model rotation** (this plan): when a model is dead/blocked, switch the
  *model* for the request, optionally with a fresh key.

## Observed gap (why this plan exists)

In `nexus-gateway/src/nexus_gateway/application/services/upstream_service.py`
`call_static`:
- HTTP 404 currently raises a terminal `UpstreamError(502)` ("Model not found").
  A shut-down model (e.g. `gemini-2.0-flash`) should instead fall back.
- The 429 path references `self._other_model(request.model)` — a method that
  does not exist (was a latent AttributeError). The intended behavior (rotate
  model on first 429 strike, then key) was never implemented.
- `rotated_model_on_this_key` flag exists but the model-swap never happens.

## Design

### 1. Curated replacement map (single source of truth)

A versioned, hand-maintained mapping derived from the deprecations page.
Structure (ordered fallbacks):

```
ROTATION_MAP = {
  "gemini-2.0-flash":      ["gemini-2.5-flash", "gemini-3.5-flash"],
  "gemini-2.0-flash-lite": ["gemini-2.5-flash-lite", "gemini-3.1-flash-lite"],
  "gemini-2.5-flash":      ["gemini-3.5-flash"],
  "gemini-2.5-flash-lite": ["gemini-3.1-flash-lite"],
  "gemini-3-flash-preview":["gemini-3.5-flash"],
  ...
}
ALIVE_MODELS = {  # models safe to serve today (2026-07-15)
  "gemini-3.5-flash", "gemini-2.5-flash", "gemini-3.1-flash-lite",
  "gemini-2.5-flash-lite",
}
```

- Maintained, not scraped at runtime (no machine-readable status flag exists
  in the models API — confirmed).
- Refreshed manually when the deprecations page changes; bump a version field.

### 2. Rotation triggers (when to switch model)

| Trigger | Action |
|---|---|
| HTTP 404 / 410 (model shut down) | Switch to next entry in `ROTATION_MAP[requested]`. Terminal only if map exhausted. |
| HTTP 429 (first strike on a key) | Switch model to next fallback **and** rotate key (matches existing docstring intent). |
| HTTP 503 | Existing backoff; also try model fallback before final failure. |
| `is_model_blocked` (existing) | Skip blocked model, pick next fallback. |

### 3. Rotation loop (sketch for `call_static`)

```
attempted_models = set()
while attempts remain:
    model = current fallback (start = requested)
    if model in attempted_models: advance fallback
    if no fallback left: raise 502
    attempted_models.add(model)
    request.model = model
    current_key = acquire()  # fresh key each model switch
    ... POST ...
    on 404/429(first): continue (loop picks next fallback + key)
    on 200: return
```

Key change vs today: **each model switch also acquires a fresh key**, so a
rate-limited key doesn't poison the fallback model. Loop terminates when
all fallbacks exhausted (prevents infinite loops — the original bug).

### 4. Config surface

- `GEMINI_MODELS` (already in docker-compose) = the *allowed entry models*
  the gateway will accept/serve. Rotation fallbacks are internal.
- Optional `MODEL_ROTATION=on|off` to disable fallback (strict mode) for
  callers that must pin an exact model.

## Decisions (locked 2026-07-15 — see DECISIONS.md)

1. **Trigger:** switch model on 404/410; on 429 rotate KEY first, switch model
   only after the key was already rotated (model is the second lever). This
   avoids masking per-key quota 429s.
2. **Location:** map + algorithm live directly in `nexus-gateway`
   (`src/nexus_gateway/...`), NOT a separate package. This repo = research +
   design reference only.
3. **Default:** transparent fallback ON. No kill-switch unless needed later.

## Proposed deliverables (this repo)

- `rotation_map.py` — `ROTATION_MAP`, `ALIVE_MODELS`, `next_model()` helper.
- `rotation.py` — pure function `resolve_model(requested, attempted, alive)`
  + `class ModelRotationStrategy` (testable, no I/O).
- `tests/test_rotation.py` — coverage for: shutdown fallback, map exhaustion,
  no-infinite-loop, alive-set filtering.
- `docs/integration.md` — how `nexus-gateway.call_static` calls into it.
- Later (separate step, after your sign-off): wire into `nexus-gateway`
  `upstream_service.py`; replace the dead `_other_model` reference.

## Verification

- Unit tests for `rotation.py` pass (`pytest`).
- `rotation_map.py` matches the deprecations page snapshot in
  `research-deprecations.md` (review diff on each refresh).
- Integration verified against `nexus-gateway` only after wiring step, using a
  model known dead (e.g. `gemini-2.0-flash`) to confirm it falls back instead
  of 502.
