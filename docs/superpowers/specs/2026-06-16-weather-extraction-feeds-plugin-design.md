# Extract `weather` into the `led-ticker-feeds` plugin — Design

**Date:** 2026-06-16
**Status:** Approved (brainstorm with James)

## Context

The `rss_feed` widget was just extracted into `led-ticker-feeds` as `feeds.rss`
(merged: led-ticker-feeds #1, core #226). This extracts the **weather** widget
into the same plugin as `feeds.weather`, the second widget the repo was shaped
for (its `register()` was written to add exactly one more `api.widget` line).

Like rss, weather is a **clean port**: every symbol it imports is already on the
public `led_ticker.plugin` surface — `as_color_provider`, `FrameAwareBase`,
`PixelData`, `compute_baseline` / `compute_cursor` / `get_text_width`,
`draw_text` / `draw_text_per_char`, `draw_emoji_at` / `measure_emoji_at`,
`FONT_DEFAULT`, the `colors` module, `run_monitor_loop`, `spawn_tracked`. P3
pre-provisioned all of these and pre-whitelisted weather in the readiness
allowlist, so the **import audit needs no API work**. `aiohttp` is already a feeds
dependency; weather adds **no new dependency** (rss brought `feedparser`).

The one deliberate public-API addition is *not* driven by an import the audit
missed — it's an architecture choice (decision below): exposing the rich
color-spec coercion as `coerce_color_provider` so the plugin can own coercion of
its plugin-unique `font_color_temp` field instead of leaning on core's private
field-name registry. That's Phase 0, a small additive core PR that merges first.

**Decisions (from brainstorm):**
- Namespaced type: **`feeds.weather`** (old `weather`).
- **weather_icons split** (the one fork the readiness audit missed, mirroring the
  `Ticker.from_rss_feed` discovery): core's `pixel_emoji._build_emoji_registry`
  imports the icon pixel-art (`SUN` / `CLOUD` / `RAIN` / `SNOW` / `THUNDER` /
  `FOG` / `PARTLY_CLOUDY`) from `widgets/weather_icons.py` to register the
  general-purpose `:sun:` / `:cloud:` / etc. inline emoji. Those icons **stay in
  core**; only `_match_condition` (pure condition-string → slug logic, 17 lines,
  no dependence on the pixel data) **moves into the plugin**, inlined into
  `weather.py` (no separate module — YAGNI). The widget renders icons via the
  public `draw_emoji_at(canvas, _match_condition(cond), …)` using core slugs.
  No user-facing emoji change, no public-API growth.
- **`font_color_temp` coercion moves to the plugin via a new public primitive.**
  Today core's private `app/coercion._coerce_color_provider` parses the rich
  color forms (`[r,g,b]`, `"rainbow"` / `"color_cycle"` / `"shimmer"` /
  `"random"`, `{style = "gradient", …}`), triggered by a field-name registry
  (`_PROVIDER_COLOR_KEYS`). Plugins lean on that registry by coincidence — the
  parser was never public, and the public `as_color_provider` only wraps a
  *constant* `Color` (so `as_color_provider("rainbow")` would break). The
  long-term-correct separation is to **expose the parser** as
  `coerce_color_provider(value, context="font_color") -> ColorProvider | None`
  on `led_ticker.plugin`, then have the plugin coerce its own plugin-unique color
  field (`font_color_temp`). Establishes the rule: **shared/generic color field
  names (`font_color`, `top_color`, …) stay core-coerced; plugin-unique color
  fields are coerced by the plugin via the public primitive.** `font_color`
  remains core-coerced (still used by `message`/etc.), so the plugin keeps
  relying on it for that field exactly as `feeds.rss` does.
- **Three-phase sequencing** (the public primitive forces a prerequisite phase):
  Phase 0 adds `coerce_color_provider` to core's public surface and merges
  first (purely additive, safe); Phase A ports the plugin (its CI
  sibling-checks-out core `main`, so the symbol must already be there); Phase B
  removes the core widget. Two merge gates: 0 → A → B. Explicit per-PR merge
  consent required (never merge without James's go-ahead).

## Phase 0 — public `coerce_color_provider` (core, additive, merges first)

A small, purely additive core PR that ships the missing ecosystem primitive.
Nothing is removed, so it's safe to land independently.
- `src/led_ticker/plugin.py`: add a thin public wrapper and append it to
  `__all__`:
  ```python
  def coerce_color_provider(value, context="font_color"):
      """Parse a TOML color spec — constant [r, g, b]; "rainbow" /
      "color_cycle" / "shimmer" / "random"; or {style = "...", ...} — into a
      ColorProvider (None for None input). The public primitive for plugins
      with custom color fields; mirrors how core coerces font_color at
      config-load. Already-a-provider values pass through unchanged."""
      from led_ticker.app.coercion import _coerce_color_provider
      return _coerce_color_provider(value, context)
  ```
- `docs/site/src/content/docs/plugins/api-reference.mdx`: add the
  `coerce_color_provider` row inside the marked region (the drift guard
  `tests/test_docs_plugin_api_drift.py` compares `plugin.__all__` against this
  page; both must match).
- Test: assert `coerce_color_provider("rainbow", "font_color_temp")` returns a
  per-frame provider, `[r,g,b]` returns a constant provider, an already-provider
  passes through, and the drift test stays green.

## Phase A — `feeds.weather` in `led-ticker-feeds`

### Package
- `src/led_ticker_feeds/weather.py` — near-verbatim copy of core's
  `widgets/weather.py`:
  - Drop the `@register("weather")` decorator + `from led_ticker.widgets import
    register` (registration moves to `register()`).
  - Repoint imports to `from led_ticker.plugin import (...)`. Replace
    `from led_ticker.colors import DEFAULT_COLOR, RGB_WHITE` with
    `from led_ticker.plugin import colors as _colors` and use
    `_colors.DEFAULT_COLOR` / `_colors.RGB_WHITE` (the same attrs-field-shadow
    avoidance rss used — `colors` would otherwise collide with field defaults).
  - Inline `_match_condition` as a module-level function (lifted verbatim from
    core's `weather_icons.py`); drop the
    `from led_ticker.widgets.weather_icons import _match_condition` lazy import.
  - **Color coercion** (the B-proper change): import `coerce_color_provider`
    from `led_ticker.plugin`. In `__attrs_post_init__`, coerce `font_color_temp`
    via `coerce_color_provider(self.font_color_temp, "font_color_temp")` instead
    of the constant-only `as_color_provider` (core no longer pre-coerces this
    plugin-unique field; raw `"rainbow"` / `{style=…}` would otherwise reach the
    widget). Coerce `font_color` the same way for symmetry — it's idempotent
    (already-a-provider passes through), and core still pre-coerces `font_color`,
    so this is a safe no-op there. The `if not hasattr(..., "color_for")` guards
    are replaced by direct `coerce_color_provider` calls (pass-through handles the
    already-coerced case).
  - Keep the rest verbatim: `start()` classmethod, `async update()` with the
    aiohttp fetch, the `WEATHERAPI_KEY` env read (raises `ValueError` if unset),
    `FrameAwareBase` inheritance, the two-color `font_color` (label) +
    `font_color_temp` (temperature) `ColorProvider` design, `show_icon`,
    `bg_color`, INFO logging.
- `src/led_ticker_feeds/__init__.py` — add ONE line to the existing `register()`:
  ```python
  from led_ticker_feeds.weather import WeatherWidget

  def register(api):
      api.widget("rss")(RSSFeedMonitor)
      api.widget("weather")(WeatherWidget)
  ```

### pyproject.toml
- No dependency change (`aiohttp` already present; no `feedparser` need). Bump
  version (0.1.0 → 0.2.0) since a second widget ships.

### Tests (`tests/`)
- Port core's `tests/test_widgets/test_weather.py` verbatim, imports repointed to
  `led_ticker_feeds.weather`; the `WEATHERAPI_KEY` monkeypatch fixture moves into
  the plugin's `conftest.py` (or per-test, matching the source).
- Port the `_match_condition` cases from core's `tests/test_weather_icons.py`
  (the condition→slug assertions) into a plugin test against the inlined
  function. The icon-pixel-data tests in that file STAY in core (the icons stay).
- Extend `test_smoke.py`: assert `feeds.weather` registers and resolves to
  `WeatherWidget` (alongside the existing `feeds.rss` assertion).
- `test_import_purity.py` already AST-scans all of `src/led_ticker_feeds/` — it
  covers `weather.py` automatically (every `led_ticker.*` import must be from
  `led_ticker.plugin`).

### README.md
- Add a **Weather** section with the demo GIF (`docs/weather.gif`, copied from
  core's `public/demos-long/widget-weather.gif`): intro, a `WEATHERAPI_KEY`
  prerequisite note, a `type = "feeds.weather"` config example + field table
  (ported from core's weather docs, including `font_color` / `font_color_temp` /
  `show_icon`).
- Update "What it provides" to list both `feeds.rss` and `feeds.weather`; remove
  the "weather is planned for this repo" note (now shipped).

### Supporting files
- `CLAUDE.md`: add a weather invariant (two-color provider design; `_match_condition`
  is an inlined plugin-internal helper; icons live in core's emoji registry, the
  plugin only uses public slugs).

## Phase B — core removal PR

- Delete `src/led_ticker/widgets/weather.py`; drop `weather,` from
  `src/led_ticker/widgets/__init__.py` auto-imports (6 built-ins remain — core
  now ships **no** network/data widgets; all are plugins).
- `src/led_ticker/widgets/weather_icons.py`: remove `_match_condition` (dead in
  core once weather leaves). KEEP the icon pixel-art + `PixelData` import (still
  feeds `pixel_emoji._build_emoji_registry`).
- `src/led_ticker/app/factories.py`:
  - Add to `_EXTRACTED_TYPES`:
    ```python
    "weather": (
        "Widget type 'weather' was extracted from led-ticker core; it now ships "
        "in the led-ticker-feeds plugin as 'feeds.weather'.",
        "Install led-ticker-feeds (add it to config/requirements-plugins.txt) "
        'and use type = "feeds.weather".',
    ),
    ```
  - Remove the `font_color_temp` and `show_icon` `FieldHint`s (weather-specific
    `--list-fields` surface; moves to the plugin).
  - Remove `"weather"` from the `message`→`text` migration-check tuple.
- `src/led_ticker/app/coercion.py`: **remove** `font_color_temp` from
  `_PROVIDER_COLOR_KEYS` (it's no longer a core-known field — the plugin now
  coerces it via the public `coerce_color_provider` from Phase 0). Leave
  `font_color`, `top_color`, etc. (still used by core widgets). Update the
  surrounding comment to note that plugin-unique color fields self-coerce.
- Tests:
  - Delete `tests/test_widgets/test_weather.py`.
  - Delete the golden `tests/golden/list_fields/weather.txt` and its entry in
    `tests/test_list_fields_golden.py` (weather field-listing moved to the plugin).
  - Split `tests/test_weather_icons.py`: keep the icon-registry / pixel-data
    tests; remove the `_match_condition` tests (moved to the plugin).
  - Remove weather from `tests/test_widgets/test_registry.py` (built-in count →
    6), `tests/test_app.py` (network-skip list entry + weather config / list-fields
    tests), `tests/test_widgets/test_widget_updates.py` (the weather update class +
    its `WEATHERAPI_KEY` fixture usage), `tests/test_validate.py` (weather
    validation test).
  - Add `test_bare_weather_type_raises_migration_to_plugin` to the migration test
    file (mirror the rss test: assert `led-ticker-feeds` in message, `feeds.weather`
    in fix).
  - Remove the `"widgets/weather.py"` and `"widgets/weather_icons.py"` entries
    from `tests/test_plugin_extraction_readiness.py`'s `_ALLOWED` (weather.py is
    gone; weather_icons.py is no longer an extraction candidate).
  - Audit `tests/test_dispatch_drift.py`, `tests/test_plugins/test_loader_core.py`,
    `tests/test_plugin_hint.py`, `tests/test_webui_redact.py`,
    `tests/test_drawing.py`, `tests/test_hires_font_loader.py`,
    `tests/test_pixel_emoji.py`, `tests/test_widgets/test_image_text_wrap.py`,
    `tests/test_validate.py`, and `tests/fixtures/broken-bigsign-config.toml` for
    `type = "weather"` / `WeatherWidget` coupling; fix each by the rss principle
    (weather-widget-specific → delete/repoint; generic-but-named → keep; sample
    config using the widget → swap to a core widget or stub). The reverse-coupling
    sweep is the implementation's discovery step (the readiness audit only checks
    forward imports), exactly as it was for rss.
- Core docs site:
  - KEEP `docs/site/src/content/docs/widgets/weather.mdx` — reduce to a plugin
    stub (keep slug + committed `DemoGif`, drop `TomlExample` / `OptionsTable` /
    `RelatedPages`), point at the led-ticker-feeds README. Mirror
    `crypto-coingecko.mdx` / the rss reframe exactly.
  - Remove `docs/site/demos-long/widget-weather.toml` from the render pipeline
    (uses the now-removed type; would fail `make render-long-demos`). The
    committed `public/demos-long/widget-weather.gif` stays.
  - `widgets/index.mdx`, `astro.config.mjs` sidebar, `plugins/available.mdx`:
    move weather to the plugin grouping as `feeds.weather (plugin)`; add it to the
    feeds entry in `available.mdx`.
  - Sweep stray `weather` → `feeds.weather` literals (config literals in
    concepts/tutorial/hardware pages; the `weather` rows in `concepts/animations.mdx`
    and `concepts/borders.mdx` tables — mirror the rss `feeds.rss` rename).
- `config/requirements-plugins.example.txt` / `plugins_catalog.json` / `CLAUDE.md`:
  the feeds plugin is ALREADY registered (from the rss extraction). Update the
  catalog `provides` to `["feeds.rss", "feeds.weather"]`, the catalog `summary`,
  and the CLAUDE.md ecosystem line to mention both widgets. No new requirements
  line (the feeds git line already installs the whole plugin).

## Testing

- **Phase 0 (core)**: `coerce_color_provider` unit test (rich-form parsing +
  pass-through) green; `test_docs_plugin_api_drift` green with the new symbol
  documented; `make test` otherwise unchanged.
- **Plugin**: ported weather tests green; `_match_condition` tests green;
  `test_import_purity` (only-public imports, covers weather.py); `test_smoke`
  (`feeds.weather` registers); coverage ≥ 90%; CI green (sibling checkout).
- **Core**: `make test` green after removal (migration test passes; readiness
  tripwire passes with both weather entries removed; golden list-fields suite
  passes without weather; no demo-render failure); `make lint` / `make docs-lint`
  clean; `led-ticker validate` on a `type = "weather"` config raises the
  migration error pointing at `feeds.weather`; core's `:sun:` / `:cloud:` / etc.
  emoji still render (icons stayed).

## Out of scope

- Changing the WeatherAPI provider or adding new weather data fields.
- Publishing `led-ticker-feeds` to PyPI.
- Re-rendering the demo GIF (the committed one carries over as-is).

## Delivery

Three phases, two merge gates, explicit per-PR consent at each merge:
1. **Phase 0** (core, additive): add public `coerce_color_provider` + its
   api-reference row → PR → merge first.
2. **Phase A** (plugin): port `weather.py` + inline `_match_condition` +
   plugin-side `coerce_color_provider` coercion + tests + README/GIF → PR →
   merge (needs Phase 0 on core `main` for its sibling-checkout CI).
3. **Phase B** (core removal): delete the widget, remove `font_color_temp` from
   coercion, migration entry + reverse-coupling sweep, docs reframe → PR → merge
   last (after Phase A).

The spec + plan live in this repo (`led-ticker-feeds/docs/superpowers/`).
