# CLAUDE.md

Guidance for Claude Code when working in **led-ticker-feeds**, an external plugin for
[led-ticker](https://github.com/JamesAwesome/led-ticker).

`README.md` is the source of truth for the user-facing surface (widget options, install).
This file keeps the **load-bearing invariants** a contributor must respect, plus navigation aids.
When a fact here and the README disagree about *how a feature works*, the README wins; this file
is the source of truth for *how to keep it working*.

## Overview

This plugin contributes, via the `led_ticker.plugins` entry point, data-feed widgets:

- `feeds.rss` — headlines from any RSS feed; a Container that polls in the background and
  populates `feed_stories` with `TickerMessage` items; the display loop re-reads the list on
  every pass, so updates surface within one cycle without restarting.

The entry-point name `feeds` is the plugin namespace, so the config type is `feeds.rss`
(see `register()` in `__init__.py`).

## Commands

led-ticker is **not on PyPI**; it resolves from a sibling checkout via
`[tool.uv.sources] led-ticker = { path = "../led-ticker", editable = true }`. CI checks out
`led-ticker` next to this repo using a read-only deploy key (`LED_TICKER_DEPLOY_KEY`). The
sibling checkout matters at test time too: `pyproject.toml` puts `../led-ticker/tests/stubs`
on the pytest path so the rgbmatrix stub is importable headless.

```bash
uv sync --extra dev          # install deps (needs ../led-ticker checked out)
uv run pytest -q             # full suite (asyncio_mode = "auto")
uv run ruff check src tests  # lint — run before pushing
uv run ruff format src tests # format
uv run pyright src           # type-check
```

Python **3.14+** only.

## Package layout

```
src/led_ticker_feeds/
  __init__.py   # register(api) entry point — the only place names are registered
  rss.py        # feeds.rss widget (RSSFeedMonitor)
```

`register()` in `__init__.py`:

```python
def register(api):
    api.widget("rss")(RSSFeedMonitor)
```

## Load-bearing invariants

Each rule must hold when modifying the named area.

**Import only the public surface** — every `led_ticker` import MUST come from `led_ticker.plugin`,
never `led_ticker.<internal>`. Enforced by `tests/test_import_purity.py`, which AST-walks every
source file (catches `from`-imports *and* `import led_ticker.x` forms, not just a text grep).
Intra-package imports (`from led_ticker_feeds.rss import …`) are fine. If you need a core symbol
that isn't on `led_ticker.plugin.__all__`, that's a core API change — raise it upstream, don't
reach around the surface.

**`_colors` import alias** — `rss.py` imports `led_ticker.plugin.colors as _colors` (with the
leading underscore). This is intentional: `RSSFeedMonitor` has an attrs field named `colors`, and
using the same name for the module import would shadow it inside the class body. Do NOT rename the
import alias back to `colors` — it reintroduces the shadow.

**Plugin shape** — entry point `feeds = "led_ticker_feeds:register"` (in `pyproject.toml`);
namespace `feeds`; `register(api)` calls `api.widget("rss")(RSSFeedMonitor)` which makes the
config type `feeds.rss`.

**Container invariant** — `RSSFeedMonitor.feed_stories: list[TickerMessage]` (and `feed_title`)
are rebuilt by the background `update()` task. The engine pushes the widget AS ITSELF into
`Ticker.monitors` (not pre-expanded) and re-reads `feed_stories` via `_expand_sources` on every
pass. Never snapshot or cache `feed_stories` at section-build time — that was the longboi
stale-display bug.

**`feedparser` off the event loop** — `update()` MUST call `feedparser.parse` via
`asyncio.to_thread`. The feedparser XML parse is CPU-bound; running it on the event loop directly
blocks all widgets for the duration of the parse. Tripwire: `TestFeedparserOffEventLoop` in
`tests/test_rss.py`.

**`font_color` semantics** — when `font_color` is unset (`None`), each story draws from the
legacy 3-color cycle (`DEFAULT_COLOR` / `RED` / `GREEN`) via `_story_color()`. When `font_color`
is explicitly set (e.g. `font_color = "rainbow"`), ALL stories share it. Both paths run through
`_story_color()` — don't bypass it.

**Python 3.14 / PEP 649** — no `from __future__ import annotations` anywhere (same rule as core).
Bare `tuple[int, int, int]` annotations are fine.

**Weather-ready** — adding a planned `feeds.weather` widget means creating a new module and adding
one `api.widget("weather")(WeatherWidget)` line in `register()`. The namespace stays `feeds`.
No structural change to `__init__.py` or `rss.py` is needed; import-purity and smoke tests cover
the new module automatically.

## Tests / CI

`uv run pytest -q` runs the suite (`tests/`):

- `test_import_purity.py` — AST tripwire (public-surface-only). Treat a failure as a contract
  violation, not a test to relax.
- `test_smoke.py` — loads the plugin through led-ticker's real plugin loader and asserts
  `feeds.rss` registers under the `feeds.*` namespace (entry-point wiring guard).
- `test_rss.py` — behavior and validation coverage for `RSSFeedMonitor`.

CI (`.github/workflows/ci.yml`): checks out this repo + led-ticker as siblings (deploy key),
Python 3.14, `uv sync --extra dev`, then `ruff check src tests`, `ruff format --check src tests`,
`pyright src`, and `pytest --cov=src --cov-report=term-missing`.

## Adding to the plugin

Register the class in `register()` in `__init__.py` (`api.widget`); it becomes `feeds.<name>`.
Import any core dependency from `led_ticker.plugin` only, and keep the import-purity test green.
