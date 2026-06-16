# Extract `rss_feed` into the `led-ticker-feeds` plugin — Design

**Date:** 2026-06-16
**Status:** Approved (brainstorm with James)

## Context

The plugin-extraction review (in led-ticker core,
`docs/superpowers/reviews/2026-06-15-plugin-extraction-recommendation.html`)
recommended extracting the data-feed widgets into a `led-ticker-feeds` plugin.
The three prerequisites are merged: P1 (migration hints), P2 (hi-res transition
API), and P3 (extraction-readiness audit — `as_color_provider`, the public
surface, the readiness tripwire). This extracts `rss_feed` as the first widget
in `led-ticker-feeds`, modeled on the already-shipped `led-ticker-calendar`.

`rss_feed` is a clean extraction: every symbol it imports is already on the
public `led_ticker.plugin` surface (verified — `Color`, `Font`, `FONT_DEFAULT`,
`TickerMessage`, `run_monitor_loop`, `spawn_tracked`, and the `DEFAULT_COLOR` /
`RED` / `GREEN` constants via the public `colors` module). It builds
`TickerMessage`s and exposes a `feed_stories` container; it needs none of the
rich-text helpers calendar used.

**Decisions (from brainstorm):**
- Namespaced type: **`feeds.rss`** (old `rss_feed`). Weather will later add
  `feeds.weather` in the same repo.
- The core docs-site page **stays** (with a "ships in the plugin" install note),
  mirroring `calendar.mdx` / `pool.mdx` / `crypto-coingecko.mdx`. The plugin
  README is a second, self-contained docs surface with the demo GIF.
- **Sequenced plugin-first**: build + merge `led-ticker-feeds`, then a core
  removal PR — so core never has a gap.

## Phase A — `led-ticker-feeds` plugin repo

Mirrors `led-ticker-calendar`'s structure (its pyproject / register / CI /
conftest / README are the template).

### Package
- `src/led_ticker_feeds/__init__.py` — `register(api)`:
  ```python
  from led_ticker_feeds.rss import RSSFeedMonitor

  def register(api):
      api.widget("rss")(RSSFeedMonitor)
  ```
  Registers `feeds.rss`. Shaped so weather later adds one `api.widget("weather")`
  line — no structural change.
- `src/led_ticker_feeds/rss.py` — near-verbatim copy of core's
  `widgets/rss_feed.py`:
  - Drop the `@register("rss_feed")` decorator (registration is in `register()`).
  - Repoint imports to `from led_ticker.plugin import (Color, Font, FONT_DEFAULT,
    TickerMessage, colors, run_monitor_loop, spawn_tracked)`; use
    `colors.DEFAULT_COLOR` / `colors.RED` / `colors.GREEN`.
  - Keep the `RSSFeedMonitor` class verbatim otherwise: `start()` classmethod,
    `async update()` with `asyncio.to_thread(feedparser.parse, …)` (off the event
    loop), the `feed_stories: list[TickerMessage]` container, `_story_color()`
    (`font_color` override vs the legacy `DEFAULT_COLOR`/`RED`/`GREEN` cycle),
    `bg_color`, `max_stories`, INFO logging.

### pyproject.toml (from calendar's template)
- `name = "led-ticker-feeds"`, `version = "0.1.0"`, `requires-python = ">=3.14"`.
- `dependencies = ["led-ticker", "aiohttp", "feedparser"]` (aiohttp also a core
  dep; `feedparser` is the new one).
- `[project.entry-points."led_ticker.plugins"] feeds = "led_ticker_feeds:register"`.
- `[tool.uv.sources] led-ticker = { path = "../led-ticker", editable = true }`.
- dev extras (pytest, pytest-asyncio, pytest-cov, pre-commit, ruff, pyright);
  `[tool.pytest.ini_options] asyncio_mode = "auto"`, `pythonpath =
  ["../led-ticker/tests/stubs"]`; ruff `target-version = "py314"`, lint select
  `["E","F","I","UP","B","SIM"]`; pyright extraPaths the stubs;
  `[tool.coverage.report] fail_under = 90`; hatchling build of
  `src/led_ticker_feeds`.

### Tests (`tests/`)
- Port core's `tests/test_widgets/test_rss_feed.py` verbatim: the `mock_session`
  fixture + the five classes (`TestRSSFeedMonitor`, `TestRssBgColor`,
  `TestRssFontColor`, `TestFeedparserOffEventLoop`, `TestRSSFeedUpdateLogging`),
  with imports repointed to `led_ticker_feeds.rss`.
- `conftest.py` — `canvas` + `make_widget` mock fixtures (copied from calendar's,
  since the plugin doesn't ship core's conftest). Include only what the rss tests
  use.
- `test_import_purity.py` — AST tripwire: every `led_ticker.*` import in
  `src/led_ticker_feeds/` must be from `led_ticker.plugin` (copy calendar's).
- `test_smoke.py` — load the entry point, assert `feeds.rss` registers and
  resolves to `RSSFeedMonitor` (copy calendar's registration guard).

### README.md
Framed as the multi-widget *feeds* plugin (weather-ready):
1. Intro + the RSS demo GIF (`docs/rss.gif`, copied from core's
   `widget-rss_feed.gif`).
2. Prerequisites (led-ticker + a feed URL).
3. Install — `config/requirements-plugins.txt` (Docker) + standalone `pip`.
4. What it provides — `feeds.rss` now; a one-line "weather is planned for this
   repo" note.
5. Config — example TOML + a field table ported from core's rss docs (with
   `type = "feeds.rss"`).
6. Development — sibling-checkout setup.
7. Links — core repo + `docs.ledticker.dev`.

### Supporting files (copied/adapted from calendar)
`CLAUDE.md` (contributor invariants: public-surface-only imports enforced by
`test_import_purity`, the `feed_stories` container contract, the
`asyncio.to_thread` feedparser constraint, the weather-ready `register()`
structure), `Makefile`, `.github/workflows/ci.yml` (sibling-checkout of core via
`LED_TICKER_DEPLOY_KEY`; lint + typecheck + test + `ci-passed` gate),
`.pre-commit-config.yaml`, `.gitignore`. Optionally a
`config/config.feeds_smoketest.toml`.

## Phase B — core removal PR

Mirrors the calendar removal (the `_EXTRACTED_TYPES` + tripwire infra is already
in place):
- Delete `src/led_ticker/widgets/rss_feed.py`; drop `rss_feed,` from
  `src/led_ticker/widgets/__init__.py` auto-imports.
- Add to `_EXTRACTED_TYPES` (`src/led_ticker/app/factories.py`):
  ```python
  "rss_feed": (
      "Widget type 'rss_feed' was extracted from led-ticker core; it now ships "
      "in the led-ticker-feeds plugin as 'feeds.rss'.",
      "Install led-ticker-feeds (add it to config/requirements-plugins.txt) "
      'and use type = "feeds.rss".',
  ),
  ```
- Delete `tests/test_widgets/test_rss_feed.py`; add
  `test_bare_rss_feed_type_raises_migration_to_plugin` to
  `tests/test_widgets/test_crypto_migration.py` (mirror the calendar test:
  assert `led-ticker-feeds` in message, `feeds.rss` in fix).
- Remove the `"widgets/rss_feed.py"` entry from the P3 readiness tripwire's
  `_ALLOWED` dict (`tests/test_plugin_extraction_readiness.py`).
- `config/requirements-plugins.example.txt`: add the feeds line (enabled by
  default), before the calendar line:
  ```
  # RSS/Atom feed headlines (type = "feeds.rss"):
  git+https://github.com/JamesAwesome/led-ticker-feeds.git@main
  ```
- `src/led_ticker/plugins_catalog.json`: add the `feeds` entry (namespace
  `feeds`, summary, homepage, `provides: ["feeds.rss"]`, git source).
- `CLAUDE.md` ecosystem section: add the `led-ticker-feeds` line after crypto.
- Core docs site: KEEP `docs/site/src/content/docs/widgets/rss_feed.mdx` — add
  the "ships in the led-ticker-feeds plugin — install it" note and update the
  example `type` to `feeds.rss` (mirror `calendar.mdx`). Keep the committed GIF
  (the page renders it) and `content-source/widgets/rss_feed.md` (the options
  table behind the page). Update any `type = "rss_feed"` literals in other docs
  pages (showcase/index/getting-started/tutorial/pitfalls) to `feeds.rss` where
  they appear as copy-pasteable config.
- **Demo TOML**: remove `docs/site/demos-long/widget-rss_feed.toml` from core's
  render pipeline (it uses the now-removed `type` and would fail
  `make render-*demos`). The pre-rendered GIF stays committed. A copy of the
  TOML goes to the plugin repo for regenerating its README GIF.

## Testing

- **Plugin**: ported rss tests green; `test_import_purity` (only-public imports);
  `test_smoke` (`feeds.rss` registers); coverage ≥ 90%; CI green (sibling
  checkout).
- **Core**: `make test` green after removal (the migration test passes; the
  readiness tripwire passes with the rss entry removed; no demo-render failure);
  `make lint` / `make docs-lint` clean; `led-ticker validate` on a `type =
  "rss_feed"` config raises the migration error pointing at `feeds.rss`.

## Out of scope

- Adding the weather widget (separate future PR — repo is only *shaped* for it).
- Publishing `led-ticker-feeds` to PyPI (item #4 plugin-registry project).
- Re-rendering the demo GIF (the committed one carries over as-is).

## Delivery

Plugin-first. Phase A: scaffold + port + tests + README/GIF + CI in
`led-ticker-feeds` → PR → merge. Phase B: core removal PR → (await explicit
merge consent) → merge. The spec + plan live in this repo
(`led-ticker-feeds/docs/superpowers/`).
