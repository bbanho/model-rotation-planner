# Decisions (2026-07-15)

Locked with user before implementation.

1. **Rotation trigger**
   - Switch model on HTTP 404 / 410 (model shut down).
   - On HTTP 429: rotate the KEY first; only switch model if the key was
     already rotated and still 429s (model is the *second* lever, not the
     first). This prevents masking a real per-key quota issue (we just hit
     5 RPM/key — a model swap would have hidden it).

2. **Map + algorithm location**
   - Live directly in `nexus-gateway` (`src/nexus_gateway/...`), NOT as a
     separate importable package. This `model-rotation-planner` repo is
     research + design reference only.

3. **Default mode**
   - Transparent fallback ON by default. A dead requested model falls back to
     its replacement instead of 502. (No `MODEL_ROTATION` kill-switch planned
     unless needed later.)

## Resulting behavior matrix

| Trigger | Lever 1 | Lever 2 | Terminal? |
|---|---|---|---|
| 404 / 410 | model -> next fallback | (fresh key) | yes, if fallbacks exhausted |
| 429 (first) | key rotate | — | no |
| 429 (after key rotated) | model -> next fallback | fresh key | yes, if fallbacks exhausted |
| 503 | backoff (existing) | model fallback before final | yes, if fallbacks exhausted |
