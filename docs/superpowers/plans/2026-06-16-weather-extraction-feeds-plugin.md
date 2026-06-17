# Weather → `feeds.weather` Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract the `weather` widget from led-ticker core into the existing `led-ticker-feeds` plugin as `feeds.weather`, and expose the rich color-spec coercion publicly so the plugin owns coercion of its plugin-unique `font_color_temp` field.

**Architecture:** Three merge-gated phases. **Phase 0** (core, additive) adds a public `coerce_color_provider` to `led_ticker.plugin` and merges first — the plugin's CI sibling-checks-out core `main`, so the symbol must exist there before Phase A. **Phase A** ports `weather.py` into the plugin (inlining `_match_condition`, repointing all imports to `led_ticker.plugin`, coercing `font_color_temp` via the new primitive) and registers `feeds.weather`. **Phase B** removes the core widget, removes `font_color_temp` from core's coercion registry, adds the migration entry, and does the reverse-coupling + docs sweep. Gates: 0 → A → B; explicit per-PR merge consent at each.

**Tech Stack:** Python 3.14, attrs, aiohttp, uv, hatchling, pytest (+pytest-asyncio, pytest-cov), ruff, pyright. rgbmatrix stub on the pytest path via `pythonpath = ["../led-ticker/tests/stubs"]`.

**Repos / branches:**
- Core: `/Users/james/projects/github/jamesawesome/led-ticker`. Phase 0 + Phase B each run in a **fresh worktree off latest `origin/main`** (own branch). Core tests: `PYTHONPATH=tests/stubs uv run --extra dev pytest …` (the exact CI invocation; `make test` for the full local run).
- Plugin: `/Users/james/projects/github/jamesawesome/led-ticker-feeds`, branch `feat/weather-widget` (already created; this plan lives there). Plugin tests: `uv run pytest` (stubs are on the path via pyproject).
- Reference: the just-merged rss extraction (core PR #226, feeds PR #1) is the working template for every step here.

**MERGE GATE & CONSENT:** Phase 0 merges before Phase A; Phase A merges before Phase B. NEVER merge any PR without James's explicit, per-PR go-ahead. Subagents verify `pwd` + branch before any git op and never touch `main` directly.

---

## Phase 0 — public `coerce_color_provider` (core, additive, merges FIRST)

### Task 0.1: Expose `coerce_color_provider` on the plugin surface

**Files:**
- Modify: `src/led_ticker/plugin.py` (add the function + a `__all__` entry near `as_color_provider` at line ~109)
- Modify: `docs/site/src/content/docs/plugins/api-reference.mdx` (add a row inside the `api-exports` marked region, ~line 165 next to `as_color_provider`)
- Create: `tests/test_plugins/test_coerce_color_provider.py`

Context: core's private `app/coercion._coerce_color_provider(value, context="font_color") -> ColorProvider | None` parses `[r,g,b]`, `"rainbow"`/`"color_cycle"`/`"shimmer"`/`"random"`, and `{style=…}` tables, and passes through an already-built provider. The public `as_color_provider` only wraps a constant `Color`. This task makes the rich parser public so plugins can coerce their own color fields. The drift test `tests/test_docs_plugin_api_drift.py` asserts `plugin.__all__` matches the api-reference table, so both must change together.

- [ ] **Step 1: Write the failing test**

Create `tests/test_plugins/test_coerce_color_provider.py`:

```python
"""The public coerce_color_provider primitive for plugin color fields."""

from led_ticker.plugin import ColorProvider, coerce_color_provider


def test_constant_rgb_list_becomes_a_provider():
    p = coerce_color_provider([255, 200, 0], "font_color_temp")
    assert isinstance(p, ColorProvider)
    assert p.frame_invariant is True  # constant color is frame-invariant


def test_rainbow_string_becomes_a_per_frame_provider():
    p = coerce_color_provider("rainbow", "font_color_temp")
    assert isinstance(p, ColorProvider)
    assert p.frame_invariant is False  # rainbow animates per frame


def test_style_table_becomes_a_provider():
    p = coerce_color_provider(
        {"style": "gradient", "from": [255, 0, 0], "to": [0, 0, 255]},
        "font_color_temp",
    )
    assert isinstance(p, ColorProvider)


def test_already_a_provider_passes_through():
    first = coerce_color_provider("rainbow")
    assert coerce_color_provider(first) is first


def test_none_returns_none():
    assert coerce_color_provider(None) is None
```

- [ ] **Step 2: Run it to verify it fails**

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_plugins/test_coerce_color_provider.py -q`
Expected: FAIL — `ImportError: cannot import name 'coerce_color_provider' from 'led_ticker.plugin'`.

- [ ] **Step 3: Add the function to `plugin.py`**

Add this near the other public color helpers (e.g. just after `make_color`):

```python
def coerce_color_provider(value: Any, context: str = "font_color") -> "ColorProvider | None":
    """Parse a TOML color spec into a ColorProvider — the public primitive for
    plugins with custom color fields.

    Accepts a constant ``[r, g, b]`` / ``(r, g, b)``; the string shorthands
    ``"random"`` / ``"rainbow"`` / ``"color_cycle"`` / ``"shimmer"``; an inline
    ``{style = "...", ...}`` table; an already-built ``ColorProvider`` (returned
    unchanged); or ``None`` (returns ``None``, caller supplies the default).
    ``context`` is used in error messages. Mirrors how core coerces
    ``font_color`` at config-load, so a plugin's plugin-unique color field gets
    the same behaviour without depending on core's internal field-name registry.
    """
    from led_ticker.app.coercion import _coerce_color_provider

    return _coerce_color_provider(value, context)
```

Confirm `Any` is imported at the top of `plugin.py` (it is — used by other signatures). Then add `"coerce_color_provider",` to `__all__` (keep the list's existing ordering convention — place it adjacent to `"as_color_provider",`).

- [ ] **Step 4: Run the unit test to verify it passes**

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_plugins/test_coerce_color_provider.py -q`
Expected: PASS (5 passed).

- [ ] **Step 5: Add the api-reference row, then verify the drift test**

In `docs/site/src/content/docs/plugins/api-reference.mdx`, inside the `{/* <!-- api-exports:start --> */}` … `{/* <!-- api-exports:end --> */}` region, add a row next to `as_color_provider(color)`:

```
| `coerce_color_provider(value, context="font_color")`                       | Parse a TOML color spec (constant `[r,g,b]`; `"rainbow"`/`"color_cycle"`/`"shimmer"`/`"random"`; or `{style=…}` table) into a `ColorProvider`. Use for a plugin's own color fields. |
```

Match the column count / pipe alignment of the surrounding rows exactly (prettier will realign on docs-lint).

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_docs_plugin_api_drift.py -q`
Expected: PASS (the new symbol is now documented).

- [ ] **Step 6: Full verify + commit**

Run: `make test` (expect green), `make lint` (clean), `make docs-lint` (clean — sources nvm if pnpm/node not on PATH).

```bash
git add src/led_ticker/plugin.py docs/site/src/content/docs/plugins/api-reference.mdx tests/test_plugins/test_coerce_color_provider.py
git commit -m "feat(plugins): expose coerce_color_provider for plugin color fields"
```

### Task 0.2: Open the Phase 0 PR

- [ ] Push the branch; open a PR to core `main` titled `feat(plugins): expose coerce_color_provider for plugin color fields`. Body: explains it's an additive public primitive (no removals) enabling plugins to coerce their own color fields; the first consumer is `feeds.weather` (in-flight). Note CI may hit the known transient uv-cache flake on the self-hosted runner — re-run the `test` job if so.
- [ ] **Do NOT merge.** Report green CI + mergeable, and ask for explicit consent. Phase A cannot start its PR-CI until this is merged to core `main`.

---

## Phase A — `feeds.weather` in `led-ticker-feeds`

Runs on branch `feat/weather-widget`. **Prerequisite:** Phase 0 merged to core `main` (Phase A's CI sibling-checks-out core `main` and imports `coerce_color_provider`). Local dev works against the editable `../led-ticker` checkout — make sure that checkout is on a commit that has Task 0.1 (e.g. rebased onto merged `main`) before running Phase A tests locally.

### Task A1: Port `weather.py` + register `feeds.weather`

**Files:**
- Create: `src/led_ticker_feeds/weather.py`
- Modify: `src/led_ticker_feeds/__init__.py`
- Modify: `pyproject.toml` (version bump only)

- [ ] **Step 1: Create `src/led_ticker_feeds/weather.py`**

Copy core's `src/led_ticker/widgets/weather.py` **verbatim** except for the edits below. The class body (`start`, `update`, `draw`, `_draw_segment`, all fields, the `__attrs_post_init__` location/units logic) is unchanged.

Replace the core import block (core lines 10–18) and drop `@register("weather")` with this header:

```python
"""Weather widget using WeatherAPI.com (feeds.weather)."""

import logging
import os
from typing import Any, Self

import aiohttp
import attrs
from led_ticker.plugin import (
    FONT_DEFAULT,
    Canvas,
    Color,
    ColorProvider,
    DrawResult,
    Font,
    FrameAwareBase,
    coerce_color_provider,
    compute_baseline,
    compute_cursor,
    draw_emoji_at,
    draw_text,
    draw_text_per_char,
    get_text_width,
    measure_emoji_at,
    run_monitor_loop,
    spawn_tracked,
)
from led_ticker.plugin import (
    colors as _colors,
)

WEATHERAPI_URL: str = "https://api.weatherapi.com/v1/current.json"


def _match_condition(condition: str) -> str:
    """Map a WeatherAPI condition string to an emoji slug."""
    c = condition.lower()
    if "thunder" in c:
        return "thunder"
    if "snow" in c or "blizzard" in c or "ice" in c or "sleet" in c:
        return "snow"
    if "rain" in c or "drizzle" in c or "shower" in c:
        return "rain"
    if "fog" in c or "mist" in c:
        return "fog"
    if "partly" in c:
        return "partly_cloudy"
    if "cloud" in c or "overcast" in c:
        return "cloud"
    # Sunny, Clear, or anything else
    return "sun"


@attrs.define
class WeatherWidget(FrameAwareBase):
    """Current weather display widget."""
```

Then, in the field defaults (core lines 33 & 42), replace the bare color constants with the namespaced module access (the `colors`-as-`_colors` alias avoids the attrs field/module name shadow, same fix rss used):

```python
    font_color: Color | ColorProvider = attrs.Factory(lambda: _colors.DEFAULT_COLOR)
    # ... (keep the two-color explanatory comment verbatim) ...
    font_color_temp: Color | ColorProvider = attrs.Factory(lambda: _colors.RGB_WHITE)
```

In `__attrs_post_init__`, replace the two constant-only coercion lines (core lines 57–60) with rich coercion via the public primitive:

```python
        # Coerce raw TOML color specs into ColorProvider instances. font_color is
        # also pre-coerced by core (shared field name) — coerce_color_provider is
        # idempotent, so this is a safe pass-through there. font_color_temp is
        # plugin-unique and NOT coerced by core, so this is where its rich forms
        # ("rainbow", {style=...}, etc.) are parsed.
        self.font_color = coerce_color_provider(self.font_color, "font_color")
        self.font_color_temp = coerce_color_provider(
            self.font_color_temp, "font_color_temp"
        )
```

In `draw()`, remove BOTH lazy import lines `from led_ticker.widgets.weather_icons import _match_condition` (core lines 138 & 175) — `_match_condition` is now a module-level function in this file, so the existing `_match_condition(self.weather)` call sites resolve directly. Also remove the lazy `from led_ticker.pixel_emoji import measure_emoji_at` / `draw_emoji_at` lines (core lines 137 & 174): `measure_emoji_at` and `draw_emoji_at` are now imported at the top from `led_ticker.plugin`, so the call sites resolve directly. (Both repoints are REQUIRED — `test_import_purity` rejects any `led_ticker.*` import that isn't from `led_ticker.plugin`.)

- [ ] **Step 2: Wire registration in `__init__.py`**

Edit `src/led_ticker_feeds/__init__.py` to add one widget line and refresh the module docstring (drop the "planned" phrasing):

```python
"""led-ticker-feeds: data-feed widgets contributed via the
``led_ticker.plugins`` entry point.

The entry-point name ``feeds`` is the plugin namespace, so widgets are
``type = "feeds.rss"`` and ``type = "feeds.weather"`` in config.toml.
"""

from led_ticker_feeds.rss import RSSFeedMonitor
from led_ticker_feeds.weather import WeatherWidget


def register(api):
    api.widget("rss")(RSSFeedMonitor)
    api.widget("weather")(WeatherWidget)
```

- [ ] **Step 3: Bump the version**

In `pyproject.toml`, change `version = "0.1.0"` → `version = "0.2.0"` (a second widget ships). No dependency change — `aiohttp` is already a dep; weather needs no `feedparser`.

- [ ] **Step 4: Sanity-check import + registration**

Run: `uv run python -c "from led_ticker_feeds.weather import WeatherWidget, _match_condition; print(_match_condition('Light rain'))"`
Expected: prints `rain`, no ImportError.

- [ ] **Step 5: Commit**

```bash
git add src/led_ticker_feeds/weather.py src/led_ticker_feeds/__init__.py pyproject.toml uv.lock
git commit -m "feat: add feeds.weather widget (ported from core)"
```
(Include `uv.lock` if the version bump changed it.)

### Task A2: Port the tests

**Files:**
- Create: `tests/test_weather.py`
- Create: `tests/test_weather_icons.py`
- Modify: `tests/test_smoke.py`

- [ ] **Step 1: Port `test_weather.py`**

Copy core's `tests/test_widgets/test_weather.py` (639 lines) into `tests/test_weather.py`, repointing every `from led_ticker.widgets.weather import WeatherWidget` to `from led_ticker_feeds.weather import WeatherWidget` (top-level import + the many in-test local imports). Leave everything else verbatim: the `_set_weather_api_key` autouse fixture (`monkeypatch.setenv("WEATHERAPI_KEY", "test-key-12345")`), the per-test `monkeypatch.setenv` calls, and all imports from `rgbmatrix` / `led_ticker.color_providers` / `led_ticker.widget` (those resolve via the stub + the editable core install — they are not subject to import-purity, which only scans `src/led_ticker_feeds/`).

- [ ] **Step 2: Port the `_match_condition` tests**

Copy core's `tests/test_weather_icons.py` (62 lines — the `TestMatchCondition` class only; core's file has no other content) into `tests/test_weather_icons.py`, changing the import to `from led_ticker_feeds.weather import _match_condition`.

- [ ] **Step 3: Extend the smoke test**

In `tests/test_smoke.py`, add a `feeds.weather` assertion alongside the existing `feeds.rss` one (inside the same `load_plugins` block):

```python
        assert get_widget_class("feeds.rss") is not None
        assert get_widget_class("feeds.weather") is not None
```

- [ ] **Step 4: Run the plugin suite**

Run: `uv run pytest -q`
Expected: PASS — the ported weather tests + `_match_condition` tests + smoke (`feeds.weather` resolves) + the existing rss/import-purity tests all green. `test_import_purity` MUST pass (proves `weather.py` imports only from `led_ticker.plugin`). Coverage ≥ 90%.

- [ ] **Step 5: Commit**

```bash
git add tests/test_weather.py tests/test_weather_icons.py tests/test_smoke.py
git commit -m "test: port weather + _match_condition tests; smoke covers feeds.weather"
```

### Task A3: README weather section + demo GIF + CLAUDE.md invariant

**Files:**
- Modify: `README.md`
- Create: `docs/weather.gif`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Copy the demo GIF**

```bash
cp ../led-ticker/docs/site/public/demos-long/widget-weather.gif docs/weather.gif
```

- [ ] **Step 2: Add a Weather section to `README.md`**

After the RSS section, add a **Weather** section mirroring the RSS section's shape:
- The GIF: `![feeds.weather demo](docs/weather.gif)`.
- Intro: current conditions from WeatherAPI.com; shows `Label: <icon> <temp>°`.
- Prerequisite: a free WeatherAPI.com key in `WEATHERAPI_KEY` (env / `.env`).
- Config example:
  ```toml
  [[playlist.section.widget]]
  type = "feeds.weather"
  location = "Brooklyn"
  text = "Brooklyn"
  units = "imperial"
  font_color = "rainbow"
  font_color_temp = [255, 255, 255]
  ```
- Field table ported from core's weather docs (`location`, `text`, `units`, `font_color`, `font_color_temp`, `show_icon`, `bg_color`, `update_interval`).

Update the intro / "what it provides" list to name both `feeds.rss` and `feeds.weather`, and remove the "weather is planned" note.

- [ ] **Step 3: Add a weather invariant to `CLAUDE.md`**

Add a short bullet: `feeds.weather` keeps the two-color design (`font_color` label + `font_color_temp` temp); `font_color_temp` is coerced in-widget via the public `coerce_color_provider` (core does not coerce this plugin-unique field), while `font_color` relies on core's shared coercion; `_match_condition` is an inlined plugin-internal helper returning a core emoji slug; the condition icons themselves live in core's emoji registry — the widget only uses public slugs via `draw_emoji_at`.

- [ ] **Step 4: Commit**

```bash
git add README.md docs/weather.gif CLAUDE.md
git commit -m "docs: weather README section + demo gif + CLAUDE.md invariant"
```

### Task A4: Open the Phase A PR

- [ ] Push `feat/weather-widget`; open a PR to feeds `main` titled `feat: add feeds.weather widget`. Body summarizes the port (clean public-surface imports, inlined `_match_condition`, plugin-side `coerce_color_provider` for `font_color_temp`), notes the **Phase 0 dependency** (needs `coerce_color_provider` on core `main` — link the Phase 0 PR), and the merge gate (merges after Phase 0, before Phase B).
- [ ] Confirm CI green (sibling-checkout resolves `coerce_color_provider`) + mergeable. **Do NOT merge** — report and ask for explicit consent.

---

## Phase B — core removal PR

Runs in a **fresh worktree off latest `origin/main`** (which must already include the merged Phase 0 + the merged rss removal). **Prerequisite:** Phase A merged. The reverse-coupling sweep is a discovery step — the steps below list every known site, but the implementer MUST grep for stragglers and fix any that break `make test` (the readiness audit only checks forward imports).

### Task B1: Remove the widget, the coercion key, and wire the migration

**Files:**
- Delete: `src/led_ticker/widgets/weather.py`
- Modify: `src/led_ticker/widgets/weather_icons.py` (remove `_match_condition`; keep icon data)
- Modify: `src/led_ticker/widgets/__init__.py` (drop `weather,` from auto-imports)
- Modify: `src/led_ticker/app/factories.py` (add `_EXTRACTED_TYPES` entry; remove `font_color_temp`/`show_icon` FieldHints; drop `weather` from the message→text migration tuple)
- Modify: `src/led_ticker/app/coercion.py` (remove `font_color_temp` from `_PROVIDER_COLOR_KEYS`)
- Modify: `tests/test_widgets/test_crypto_migration.py` (add the weather migration test)
- Modify: `tests/test_plugin_extraction_readiness.py` (remove both weather `_ALLOWED` entries)

- [ ] **Step 1: Add the migration test (write it first)**

In `tests/test_widgets/test_crypto_migration.py`, add (mirrors `test_bare_calendar_type_raises_migration_to_plugin`):

```python
def test_bare_weather_type_raises_migration_to_plugin():
    result = build_widget_cfg_error_for_type("weather")
    assert result is not None
    message, fix = result
    assert "led-ticker-feeds" in message
    assert "feeds.weather" in fix
```

- [ ] **Step 2: Run it to verify it fails**

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_widgets/test_crypto_migration.py::test_bare_weather_type_raises_migration_to_plugin -q`
Expected: FAIL (no `_EXTRACTED_TYPES["weather"]` yet — `result is None`).

- [ ] **Step 3: Add the `_EXTRACTED_TYPES` entry**

In `src/led_ticker/app/factories.py`, add to the `_EXTRACTED_TYPES` dict (next to the `rss_feed` entry):

```python
    "weather": (
        "Widget type 'weather' was extracted from led-ticker core; it now ships "
        "in the led-ticker-feeds plugin as 'feeds.weather'.",
        "Install led-ticker-feeds (add it to config/requirements-plugins.txt) "
        'and use type = "feeds.weather".',
    ),
```

- [ ] **Step 4: Delete the widget + clean weather_icons + registry**

```bash
git rm src/led_ticker/widgets/weather.py
```

In `src/led_ticker/widgets/weather_icons.py`, delete ONLY the `_match_condition` function (the last ~18 lines). KEEP everything above it (the `PixelData` import + `SUN`/`CLOUD`/`PARTLY_CLOUDY`/`RAIN`/`SNOW`/`THUNDER`/`FOG` icon data) — `pixel_emoji._build_emoji_registry` imports those for the core `:sun:`/`:cloud:`/etc. emoji.

In `src/led_ticker/widgets/__init__.py`, remove `weather,` (and any `weather` reference) from the auto-import list.

- [ ] **Step 5: Remove the weather-specific factory FieldHints + migration entry**

In `src/led_ticker/app/factories.py`: delete the `"font_color_temp": FieldHint(...)` and `"show_icon": FieldHint(...)` entries (the `--list-fields` surface moves to the plugin). Remove `"weather"` from the `message`→`text` migration-check tuple (the one that raises `MigrationError` for a stale `message =` key on `("message", "countdown", "weather")` → becomes `("message", "countdown")`).

- [ ] **Step 6: Remove `font_color_temp` from core coercion**

In `src/led_ticker/app/coercion.py`, remove `"font_color_temp",` from `_PROVIDER_COLOR_KEYS`. Update the nearby comment to note plugin-unique color fields (e.g. `feeds.weather`'s `font_color_temp`) are coerced plugin-side via the public `coerce_color_provider`.

- [ ] **Step 7: Trim the readiness allowlist**

In `tests/test_plugin_extraction_readiness.py`, remove both the `"widgets/weather.py": {…}` and `"widgets/weather_icons.py": {}` entries from `_ALLOWED` (weather.py is gone; weather_icons.py is no longer an extraction candidate).

- [ ] **Step 8: Run the targeted tests**

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_widgets/test_crypto_migration.py tests/test_plugin_extraction_readiness.py -q`
Expected: PASS (migration test green; readiness tripwire green with weather removed).

- [ ] **Step 9: Commit** (after the Task B1b sweep below passes `make test` — keep the removal + sweep in one green commit)

### Task B1b: Reverse-coupling test sweep (same commit as B1)

**Files (audit + fix each):** `tests/test_widgets/test_weather.py` (delete), `tests/test_weather_icons.py` (split), `tests/golden/list_fields/weather.txt` (delete) + `tests/test_list_fields_golden.py`, `tests/test_widgets/test_registry.py`, `tests/test_app.py`, `tests/test_widgets/test_widget_updates.py`, `tests/test_validate.py`, `tests/fixtures/broken-bigsign-config.toml`, plus a grep for stragglers (`test_dispatch_drift.py`, `test_plugins/test_loader_core.py`, `test_plugin_hint.py`, `test_webui_redact.py`, `test_drawing.py`, `test_hires_font_loader.py`, `test_pixel_emoji.py`, `test_widgets/test_image_text_wrap.py`).

- [ ] **Step 1: Delete weather-widget-specific tests that moved to the plugin**

```bash
git rm tests/test_widgets/test_weather.py
git rm tests/golden/list_fields/weather.txt
```
In `tests/test_weather_icons.py`: this file is ONLY `TestMatchCondition` (62 lines) — its subject (`_match_condition`) moved to the plugin. `git rm tests/test_weather_icons.py`. (The icon pixel-data has no dedicated core test file; coverage of the icons comes via `tests/test_pixel_emoji.py` through the registry — confirm that file does NOT import `_match_condition`; if it does, drop that import/case.)

- [ ] **Step 2: Update the golden list-fields test**

In `tests/test_list_fields_golden.py`, remove the `weather` entry from whatever parametrize list / golden-file map drives it (so it no longer expects `tests/golden/list_fields/weather.txt`). Run `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_list_fields_golden.py -q` → PASS.

- [ ] **Step 3: Registry count**

In `tests/test_widgets/test_registry.py`: remove the `weather`/`WeatherWidget` import + the `"weather": WeatherWidget` expected-registry entry; decrement the built-in count assertion (rss already took it from 8→7; weather makes it 6 — rename the test to `test_registry_has_six_widgets` and assert `6`; confirm by listing the registry at runtime: `clock, gif, message, countdown, image, two_row`).

- [ ] **Step 4: test_app.py**

Remove `"weather"` from the network-widget skip list; delete any weather-specific config / list-fields tests (e.g. a `test_weather_*` that builds `type = "weather"` or calls `_list_widget_fields("weather")`). Keep generic prose comments. Run the file → PASS.

- [ ] **Step 5: test_widget_updates.py**

Remove the weather update test class + the `from led_ticker.widgets.weather import WeatherWidget` import + the `WEATHERAPI_KEY` monkeypatch the weather class used (leave other widgets' update tests intact).

- [ ] **Step 6: test_validate.py + fixtures**

Remove any weather validation test that builds `type = "weather"`. In `tests/fixtures/broken-bigsign-config.toml`, if it uses `type = "weather"` as a (non-weather-purpose) sample, swap to a remaining core widget (`message`); if the fixture's brokenness IS the weather type, repoint to whichever field it was actually exercising. Inspect before editing.

- [ ] **Step 7: Grep for stragglers, fix by principle**

Run: `grep -rn 'WeatherWidget\|widgets.weather\b\|type = "weather"\|"weather"' tests/` — for each remaining hit, apply the rss principle: weather-widget-specific test → delete; generic-but-named (a string label, a comment) → keep; a sample config USING the widget → swap to a core widget or stub. The `_match_condition`/`weather_icons` import in `tests/test_pixel_emoji.py` (if any) should reference only the icon DATA, which stays — keep those.

- [ ] **Step 8: Full suite green, then commit B1 + B1b together**

Run: `make test` → expect green (the full suite; note the count drops vs the rss baseline). Run the exact CI command too: `PYTHONPATH=tests/stubs uv run --extra dev pytest --cov=src/led_ticker --cov-report=term-missing "--ignore-glob=*/gif_plan/*" -q` → exit 0. `make lint` clean.

```bash
git add -A
git commit -m "feat: remove weather from core (extracted to led-ticker-feeds)"
```

### Task B2: Catalog `provides` + CLAUDE.md ecosystem

**Files:**
- Modify: `src/led_ticker/plugins_catalog.json`
- Modify: `CLAUDE.md`

The feeds plugin is ALREADY registered (rss extraction). No new requirements line — the existing `git+…/led-ticker-feeds.git@main` installs the whole plugin.

- [ ] **Step 1: Update the catalog `feeds` entry**

In `src/led_ticker/plugins_catalog.json`, change the `feeds` entry: `provides` → `["feeds.rss", "feeds.weather"]`, and broaden `summary` to e.g. `"RSS/Atom feed headlines and current weather (feeds.rss, feeds.weather)."`.

- [ ] **Step 2: Update CLAUDE.md ecosystem line**

In `CLAUDE.md`, update the `led-ticker-feeds` bullet (and the plugin intro paragraph) to mention both `feeds.rss` and `feeds.weather`.

- [ ] **Step 3: Run catalog tests + commit**

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_plugin_requirements.py tests/test_plugins/test_catalog.py tests/test_plugins/test_plugin_cli.py -q` → PASS.

```bash
git add src/led_ticker/plugins_catalog.json CLAUDE.md
git commit -m "feat: catalog feeds.weather; CLAUDE.md ecosystem update"
```

### Task B3: Docs reframe — page stub, demo-TOML, stray literals

**Files:**
- Modify: `docs/site/src/content/docs/widgets/weather.mdx` (→ plugin stub)
- Delete: `docs/site/demos-long/widget-weather.toml`
- Modify: `docs/site/src/content/docs/widgets/index.mdx`, `docs/site/astro.config.mjs`, `docs/site/src/content/docs/plugins/available.mdx`
- Modify: stray-literal sweep across `concepts/animations.mdx`, `concepts/borders.mdx`, and any page with `type = "weather"`

Mirror the rss reframe (core PR #226 commit `57b6c485`) and `crypto-coingecko.mdx`.

- [ ] **Step 1: Reframe `weather.mdx` to a plugin stub**

Keep the slug. Keep `import DemoGif` + `<DemoGif src="/demos-long/widget-weather.gif" … />` (committed gif stays). Drop `TomlExample`/`OptionsTable`/`RelatedPages`. Frontmatter `title: feeds.weather widget` + a description naming the plugin. Body: one paragraph ("provided by the **[led-ticker-feeds](https://github.com/JamesAwesome/led-ticker-feeds)** plugin, `type = "feeds.weather"`"), the two standard install bullets (`git+…/led-ticker-feeds.git@main` + README link), and the plugin-system closing line.

- [ ] **Step 2: Remove the demo TOML from the render pipeline**

```bash
git rm docs/site/demos-long/widget-weather.toml
```
Do NOT delete `docs/site/public/demos-long/widget-weather.gif` (the stub renders it).

- [ ] **Step 3: index / sidebar / available**

`widgets/index.mdx`: move `weather` from the core table to the plugin grouping as `[`feeds.weather`](/widgets/weather/) _(plugin)_`; update the "Live data" prose. `astro.config.mjs`: move the weather sidebar entry into the `(plugin)` group as `feeds.weather (plugin)`. `plugins/available.mdx`: add `feeds.weather` to the existing feeds entry.

- [ ] **Step 4: Stray-literal sweep**

`grep -rn 'type = "weather"\|`weather`\|"weather"' docs/site/src --include='*.mdx'`. Config literals `type = "weather"` → `feeds.weather`. In `concepts/animations.mdx` + `concepts/borders.mdx` tables (which now read `feeds.rss`, `weather` after the rss change), rename the `weather` token to `feeds.weather`. Prose mentions → `feeds.weather` for accuracy. Verify zero copy-pasteable `type = "weather"` literals remain.

- [ ] **Step 5: Verify + commit**

Run: `PYTHONPATH=tests/stubs uv run --extra dev pytest tests/test_docs_config_options_drift.py tests/ -k docs -q` → PASS. `make docs-lint` → clean.

```bash
git add -A
git commit -m "docs: reframe weather as the feeds.weather plugin widget"
```

### Task B4: Full verify + open the Phase B PR

- [ ] **Step 1: Full local verification**

Run: `make test` (green), `make lint` (clean), `make docs-lint` (clean). Spot-check the migration: a `type = "weather"` config raises the migration error pointing at `feeds.weather` (covered by the unit test). Confirm core's `:sun:`/`:cloud:` emoji still resolve (`grep -n "from led_ticker.widgets.weather_icons import" src/led_ticker/pixel_emoji.py` still imports the icon data, icons retained).

- [ ] **Step 2: Push + open PR**

Push the branch; open a PR to core `main` titled `feat: remove weather from core (extracted to led-ticker-feeds)`. Body: 3-commit summary (removal+sweep, catalog, docs), the merge-gate note (after Phase A; feeds plugin `main` already ships `feeds.weather`), and the post-merge consequence (core rebuilds need led-ticker-feeds installed for `feeds.weather` — already in the enabled-by-default requirements). Flag the known transient uv-cache CI flake → re-run `test` if hit.

- [ ] **Step 3: Report, do NOT merge**

Confirm all CI checks green + mergeable. **Do NOT merge** — report and ask for explicit per-PR consent. Phase B must merge only after Phase A is merged.

---

## Final review

After all phases merge: run the full `make test` on core `main`; confirm `feeds.weather` resolves from a config; confirm the three PRs merged in order (0 → A → B). Use superpowers:finishing-a-development-branch semantics for each branch (delete merged branches, remove worktrees).
