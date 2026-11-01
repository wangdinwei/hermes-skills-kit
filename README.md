# Hermes Skills Collection (curated & sanitized)

A curated set of reusable [Hermes Agent](https://hermes-agent.nousresearch.com/docs) skills,
covering two themes:

## frontend-pitfalls
Battle-tested fixes for common web frontend traps (Flask/Jinja2, Bootstrap/Tabler, CSS, JS):
- `bfcache-fixed-panel-bug` — fixed panels/modals persisting across back/forward (bfcache)
- `bootstrap-display-important-override-inline` — JS `style.display` loses to framework `!important`
- `css-grid-flex-card-equal-height` — equal-height cards in CSS Grid when children are flex
- `aspect-ratio-height-percentage-not-working` — `height:100%` not resolving under `aspect-ratio`
- `ajax-inline-js-event-binding` — inline `onclick`/`data-action` lost after `innerHTML` injection
- `backend-time-format-avoid-js-timezone` — render Beijing time server-side, skip JS timezone bugs
- `chinese-char-redirect-corruption` — non-URL-encoded Chinese in redirect() corrupts names
- `consolidate-duplicate-event-handlers` — find & merge duplicate DOM handlers

## ai-pipeline-robustness
Making LLM / AI pipelines resilient:
- `ai-json-response-resilience` — multi-layer JSON parsing (standard → regex → repair → self-heal)
- `ai-pipeline-json-parse-retry-fallback` — three-tier prompt fallback for structured output
- `ai-pipeline-subprocess-debug-checklist` — silent subprocess failures (ffmpeg etc.)
- `async-step-api-conversion` — sync step API → async background thread (avoid proxy 524 timeouts)

## Usage
Copy any skill folder into your Hermes skills directory (`~/.hermes/profiles/<profile>/skills/`).
Each folder contains a `SKILL.md` (YAML frontmatter + instructions) and optional `scripts/`/`references/`.

## License
MIT — see LICENSE.
