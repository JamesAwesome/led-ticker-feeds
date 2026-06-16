# RSS → led-ticker-feeds Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract the `rss_feed` widget from led-ticker core into a new `led-ticker-feeds` plugin (registered as `feeds.rss`), then remove it from core — modeled exactly on the shipped `led-ticker-calendar` plugin.

**Architecture:** Plugin-first, two repos. **Phase A** builds `led-ticker-feeds` (a near-verbatim port of `rss_feed.py` with imports repointed to the public `led_ticker.plugin` surface, plus ported tests, CI, README/GIF) and merges it. **Phase B** is a core PR that removes `rss_feed`, adds the `_EXTRACTED_TYPES` migration entry, and updates the requirements file / catalog / docs / tripwire.

**Tech Stack:** Python 3.14, uv, pytest (+pytest-asyncio), hatchling, ruff, pyright, feedparser, aiohttp.

**Spec:** `docs/superpowers/specs/2026-06-16-rss-feed-extraction-feeds-plugin-design.md` (read first).

**Reference:** the `led-ticker-calendar` repo at `../led-ticker-calendar` (on main) is the template for every Phase-A file. **Core** is at `../led-ticker`.

**Conventions:**
- Phase A runs in the `led-ticker-feeds` repo (default branch `main`, remote `git@github.com:jamesAwesome/led-ticker-feeds.git`, currently one commit: the spec). Do Phase-A feature work on a branch `feat/rss-widget` → PR #1.
- Phase B runs in the **core** repo on a branch (use a worktree). NEVER commit to core's main.
- **MERGE GATE:** Phase A must be merged before Phase B is merged (plugin-first — core must never reference a non-installable plugin). Per the owner's rule, NEVER merge any PR without explicit per-PR consent.
- Tests run via `uv run pytest` (the plugin's pyproject puts `../led-ticker/tests/stubs` on the pythonpath for the rgbmatrix stub).

---

## PHASE A — build the `led-ticker-feeds` plugin (repo: led-ticker-feeds)

### Task A1: Repo scaffold + widget + register

**Files (create):**
- `pyproject.toml`, `.gitignore`, `.pre-commit-config.yaml`, `Makefile`
- `src/led_ticker_feeds/__init__.py`, `src/led_ticker_feeds/rss.py`

- [ ] **Step 1: branch**

```bash
cd /Users/james/projects/github/jamesawesome/led-ticker-feeds
pwd && git branch --show-current   # expect: main
git checkout -b feat/rss-widget
```

- [ ] **Step 2: `pyproject.toml`** (adapt calendar's — change name/desc/deps/entry point):

```toml
[project]
name = "led-ticker-feeds"
version = "0.1.0"
description = "Data-feed widgets for led-ticker — RSS/Atom headlines (feeds.rss), with weather planned."
readme = "README.md"
requires-python = ">=3.14"
authors = [{ name = "James Awesome", email = "james@morelli.nyc" }]
dependencies = [
    "led-ticker",
    "aiohttp",
    "feedparser>=6.0",
]

# The entry-point NAME ("feeds") becomes the plugin namespace, so the widget is
# referenced in TOML as `type = "feeds.rss"`.
[project.entry-points."led_ticker.plugins"]
feeds = "led_ticker_feeds:register"

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=5.0",
    "pre-commit>=4.0",
    "ruff>=0.4",
    "pyright>=1.1",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/led_ticker_feeds"]

# led-ticker is not on PyPI yet; resolve it editable from the sibling checkout.
[tool.uv.sources]
led-ticker = { path = "../led-ticker", editable = true }

[tool.pytest.ini_options]
asyncio_mode = "auto"
pythonpath = ["../led-ticker/tests/stubs"]

[tool.ruff]
target-version = "py314"
src = ["src"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.pyright]
pythonPlatform = "All"
pythonVersion = "3.14"
extraPaths = ["../led-ticker/tests/stubs"]

[tool.coverage.report]
fail_under = 90
```

- [ ] **Step 3: `.gitignore`** (copy calendar's verbatim):

```
.venv/
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/
.pytest_cache/
.ruff_cache/
.env

# Coverage artifacts
.coverage
.coverage.*
```

- [ ] **Step 4: `.pre-commit-config.yaml`** — copy calendar's BUT pin ruff at **v0.15.7** (calendar's v0.4.0 predates `py314` and fails; core already bumped to 0.15.7):

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.7
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: uv run pyright src
        language: system
        pass_filenames: false
        always_run: true
        stages: [pre-push]
```

- [ ] **Step 5: `Makefile`** (copy calendar's verbatim):

```makefile
.PHONY: dev test lint format typecheck

dev:  ## Install dev deps + pre-commit hooks
	uv sync --extra dev
	uv run pre-commit install
	uv run pre-commit install --hook-type pre-push

test:  ## Run tests with coverage
	uv run pytest --cov=src --cov-report=term-missing

lint:  ## Ruff lint
	uv run ruff check src tests

format:  ## Ruff format
	uv run ruff format src tests

typecheck:  ## Pyright
	uv run pyright src
```

- [ ] **Step 6: `src/led_ticker_feeds/rss.py`** — port core's `widgets/rss_feed.py` with imports repointed. The ONLY changes vs core: drop `from led_ticker.widgets import register` + the `@register("rss_feed")` decorator; repoint the rest to `led_ticker.plugin`; use `colors.DEFAULT_COLOR/RED/GREEN`:

```python
"""RSS feed monitor widget."""

import asyncio
import itertools
import logging
from typing import Any, Self

import aiohttp
import attrs
import feedparser

from led_ticker.plugin import (
    FONT_DEFAULT,
    Color,
    Font,
    TickerMessage,
    colors,
    run_monitor_loop,
    spawn_tracked,
)

logger: logging.Logger = logging.getLogger(__name__)


@attrs.define
class RSSFeedMonitor:
    """Fetches and displays headlines from an RSS feed."""

    session: aiohttp.ClientSession
    feed_url: str
    padding: int = 6
    colors: itertools.cycle[Color] = attrs.Factory(
        lambda: itertools.cycle([colors.DEFAULT_COLOR, colors.RED, colors.GREEN])
    )
    max_stories: int = 5
    # When set, every story TickerMessage gets this color/provider
    # (e.g. `font_color = "rainbow"` paints all stories rainbow).
    # When unset (None), fall back to the legacy 3-color rotation
    # (DEFAULT_COLOR / RED / GREEN) so existing configs keep working.
    font_color: Any = attrs.field(default=None, kw_only=True)
    bg_color: Color | None = attrs.field(default=None, kw_only=True)
    font: Font = attrs.field(default=attrs.Factory(lambda: FONT_DEFAULT), kw_only=True)
    feed_title: TickerMessage | None = attrs.field(init=False, default=None)
    feed_stories: list[TickerMessage] = attrs.field(init=False, factory=list)

    def _story_color(self) -> Any:
        """Per-story color: `font_color` if set, else next from the
        legacy cycle. Called once per story in `update()`."""
        if self.font_color is not None:
            return self.font_color
        return next(self.colors)

    @classmethod
    async def start(
        cls,
        session: aiohttp.ClientSession,
        feed_url: str,
        update_interval: int = 1800,
        **kwargs: Any,
    ) -> Self:
        widget = cls(session=session, feed_url=feed_url, **kwargs)
        await widget.update()
        spawn_tracked(run_monitor_loop(widget, update_interval))
        return widget

    async def update(self) -> None:
        logger.info("Updating RSS Feed from: %s", self.feed_url)
        async with self.session.get(self.feed_url) as response:
            feed_data = await response.text()
            feed = await asyncio.to_thread(feedparser.parse, feed_data)
            self.feed_title = TickerMessage(
                feed["channel"]["title"],  # type: ignore[index]
                font=self.font,
                font_color=self._story_color(),
                bg_color=self.bg_color,
            )
            self.feed_stories = [
                TickerMessage(
                    item["title"],  # type: ignore[index]
                    font=self.font,
                    font_color=self._story_color(),
                    bg_color=self.bg_color,
                )
                for item in itertools.islice(feed["items"], self.max_stories)  # type: ignore[index]
            ]
        logger.info(
            "RSS %s updated: %d stories",
            self.feed_url,
            len(self.feed_stories),
        )
```

NOTE: there's a name shadow — the `colors` attrs field vs the imported `colors` module. The `lambda` in the field default closes over the *module* `colors` (resolved at call time from module globals, not the instance), so `colors.DEFAULT_COLOR` inside the lambda correctly references the imported module. This is safe but subtle; if pyright/ruff complains or a test fails on it, rename the imported module reference: `from led_ticker.plugin import colors as _colors` and use `_colors.DEFAULT_COLOR/RED/GREEN` in the lambda. Decide based on what the tests show in Task A2.

- [ ] **Step 7: `src/led_ticker_feeds/__init__.py`** (register):

```python
"""led-ticker-feeds: data-feed widgets contributed via the
``led_ticker.plugins`` entry point.

The entry-point name ``feeds`` is the plugin namespace, so the RSS widget is
``type = "feeds.rss"`` in config.toml. Weather is planned for this repo and
will register as ``feeds.weather`` with one more ``api.widget`` line below.
"""

from led_ticker_feeds.rss import RSSFeedMonitor


def register(api):
    api.widget("rss")(RSSFeedMonitor)
```

- [ ] **Step 8: install + import sanity**

```bash
uv sync --extra dev
uv run python -c "import led_ticker_feeds; from led_ticker_feeds.rss import RSSFeedMonitor; print('import ok')"
uv run ruff check src
```
Expected: sync succeeds (resolves led-ticker via path), import prints `import ok`, ruff clean. If the `colors` field/module shadow breaks the import or lint, apply the `as _colors` fix from Step 6's note.

- [ ] **Step 9: Commit**

```bash
git add pyproject.toml .gitignore .pre-commit-config.yaml Makefile src/
git commit -m "feat: scaffold led-ticker-feeds + port rss widget (feeds.rss)"
```

---

### Task A2: Port the tests

**Files (create):** `tests/conftest.py`, `tests/test_rss.py`, `tests/test_import_purity.py`, `tests/test_smoke.py`

- [ ] **Step 1: `tests/conftest.py`** (copy calendar's — `canvas` + `make_widget` mocks):

```python
"""Shared test fixtures for the led-ticker-feeds plugin test suite.

The rgbmatrix stub is on the pytest path via ``pythonpath`` in
``pyproject.toml`` (``../led-ticker/tests/stubs``). The plugin doesn't ship
core's conftest, so re-provide the small fixtures the ported tests use.
"""

import unittest.mock as mock

import pytest


@pytest.fixture
def canvas():
    """Mock LED canvas with standard width and height."""
    c = mock.Mock()
    c.width = 160
    c.height = 16
    return c


@pytest.fixture
def make_widget():
    """Factory for mock widgets with configurable draw width."""

    def _factory(content_width=40):
        widget = mock.Mock()
        widget.hold_time = 0.0
        widget.draw.side_effect = lambda c, cursor_pos=0, **kw: (
            c,
            cursor_pos + content_width,
        )
        return widget

    return _factory
```

- [ ] **Step 2: `tests/test_rss.py`** — port core's `tests/test_widgets/test_rss_feed.py` VERBATIM with exactly two edits:
  1. `from led_ticker.widgets.rss_feed import RSSFeedMonitor` → `from led_ticker_feeds.rss import RSSFeedMonitor` (both occurrences — top import + the one inside `TestRSSFeedUpdateLogging`).
  2. The monkeypatch target in `TestFeedparserOffEventLoop`: `"led_ticker.widgets.rss_feed.asyncio.to_thread"` → `"led_ticker_feeds.rss.asyncio.to_thread"`.

  Everything else copies verbatim, INCLUDING the test-only internal imports (`from led_ticker.color_providers import Rainbow`, `from rgbmatrix.graphics import Color`) and the `c._color.red/green/blue` access — tests may use core internals; only `src/` is purity-gated. The full file to write (with the two edits applied):

  Copy `../led-ticker/tests/test_widgets/test_rss_feed.py`, then apply the two replacements above. (The file is 217 lines; reproduce it exactly with only those substitutions — do not otherwise alter assertions, the `SAMPLE_RSS` fixture, or the five test classes `TestRSSFeedMonitor`, `TestRssBgColor`, `TestRssFontColor`, `TestFeedparserOffEventLoop`, `TestRSSFeedUpdateLogging`.)

- [ ] **Step 3: `tests/test_import_purity.py`** (copy calendar's, change the package dir name):

```python
import ast
import pathlib

SRC = pathlib.Path(__file__).resolve().parents[1] / "src" / "led_ticker_feeds"


def _led_ticker_imports(path):
    tree = ast.parse(path.read_text(), filename=str(path))
    names = []
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module:
            if node.module.split(".")[0] == "led_ticker":
                names.append(node.module)
        elif isinstance(node, ast.Import):
            for alias in node.names:
                if alias.name.split(".")[0] == "led_ticker":
                    names.append(alias.name)
    return names


def test_plugin_imports_only_public_surface():
    offenders = {}
    for py in SRC.rglob("*.py"):
        bad = [m for m in _led_ticker_imports(py) if m != "led_ticker.plugin"]
        if bad:
            offenders[py.name] = bad
    assert not offenders, (
        f"modules import led_ticker internals instead of led_ticker.plugin: {offenders}"
    )
```

- [ ] **Step 4: `tests/test_smoke.py`** (copy calendar's, change namespace + widget type):

```python
from led_ticker import _plugin_loader as L


def test_entry_point_registers_feeds_namespace():
    L.reset_plugins()
    try:
        result = L.load_plugins(None, entry_points_enabled=True)
        loaded = {info.namespace for info in result.loaded}
        assert "feeds" in loaded, f"feeds plugin not discovered: {result}"

        from led_ticker.widgets import get_widget_class

        assert get_widget_class("feeds.rss") is not None
    finally:
        L.reset_plugins()
```

- [ ] **Step 5: Run the suite**

```bash
uv run pytest --cov=src --cov-report=term-missing
```
Expected: all ported rss tests pass (5 classes), `test_import_purity` passes (only `led_ticker.plugin` imports in src), `test_smoke` passes (`feeds.rss` resolves via the entry point), coverage ≥ 90%. If `test_import_purity` fails, the widget still imports a core internal — fix it in `rss.py`. If the `colors` shadow surfaced, apply the `as _colors` fix and re-run.

- [ ] **Step 6: Commit**

```bash
git add tests/
git commit -m "test: port rss tests + import-purity + smoke for feeds plugin"
```

---

### Task A3: CI workflow + CLAUDE.md

**Files (create):** `.github/workflows/ci.yml`, `.github/dependabot.yml`, `CLAUDE.md`

- [ ] **Step 1: `.github/workflows/ci.yml`** — copy `../led-ticker-calendar/.github/workflows/ci.yml` VERBATIM, then replace every `led-ticker-calendar` path/checkout-path with `led-ticker-feeds`. The three jobs (lint / typecheck / test) + `ci-passed` gate, sibling-checkout of core via `secrets.LED_TICKER_DEPLOY_KEY`, `uv sync --extra dev`, `uv run pytest --cov=src`. (Read calendar's file and reproduce with the name swap — it's the exact CI contract core's other plugins use.)

- [ ] **Step 2: `.github/dependabot.yml`** — copy calendar's verbatim (no name-specific content; if it references the package, adjust).

- [ ] **Step 3: `CLAUDE.md`** — write a contributor-invariants file modeled on calendar's. Cover:
  - Public-surface-only imports (enforced by `tests/test_import_purity.py`) — `src/` imports ONLY from `led_ticker.plugin`.
  - The `feed_stories: list[TickerMessage]` container contract: the engine re-reads `feed_stories` each cycle; `update()` rebuilds it. Don't snapshot.
  - `asyncio.to_thread(feedparser.parse, …)`: feedparser is CPU-bound XML parsing; it MUST stay off the event loop (tripwire: `TestFeedparserOffEventLoop`).
  - `font_color` semantics: unset → legacy `DEFAULT_COLOR`/`RED`/`GREEN` cycle; set → all stories share it.
  - Weather-ready: adding weather is a new module + one `api.widget("weather")` line in `register()`; the namespace stays `feeds`.
  - Dev: sibling checkout of `../led-ticker`; `make dev/test/lint/typecheck`.

- [ ] **Step 4: Lint/typecheck locally**

```bash
uv run ruff check src tests && uv run ruff format --check src tests && uv run pyright src
```
Expected: clean. (`ruff format --check` may flag the ported test file's formatting — run `uv run ruff format src tests` and re-stage if so.)

- [ ] **Step 5: Commit**

```bash
git add .github/ CLAUDE.md
git commit -m "ci: add CI workflow + CLAUDE.md for led-ticker-feeds"
```

---

### Task A4: README + demo GIF + demo TOML

**Files (create):** `README.md`, `docs/rss.gif`, `docs/demos/widget-rss.toml`

- [ ] **Step 1: Copy the GIF**

```bash
mkdir -p docs/demos
cp ../led-ticker/docs/site/public/demos-long/widget-rss_feed.gif docs/rss.gif
cp ../led-ticker/docs/site/demos-long/widget-rss_feed.toml docs/demos/widget-rss.toml
```
Then edit `docs/demos/widget-rss.toml`: change `type = "rss_feed"` → `type = "feeds.rss"` (so it's the plugin-era source for regenerating `docs/rss.gif` later).

- [ ] **Step 2: `README.md`** — model on calendar's README. Sections:
  - Title + one-line intro + the GIF: `![feeds.rss — RSS headlines scrolling across the panel](docs/rss.gif)`
  - **Prerequisites**: led-ticker + a feed URL.
  - **Install**: Docker via `config/requirements-plugins.txt` (`git+https://github.com/JamesAwesome/led-ticker-feeds.git@main`) + standalone `pip install` from git.
  - **What it provides**: `feeds.rss` — RSS/Atom headlines as rotating `TickerMessage`s. One line: "Weather (`feeds.weather`) is planned for this repo."
  - **Config**: an example `[[playlist.section.widget]]` block with `type = "feeds.rss"` + `feed_url`, and a field table — port the option rows from core's `docs/site/src/content/docs/widgets/rss_feed.mdx` (`feed_url`, `max_stories`, `font`, `font_color`, `bg_color`, `padding`, `update_interval`). Use `type = "feeds.rss"` throughout.
  - **Development**: sibling-checkout setup (`git clone` core next to this; `make dev`; `make test`).
  - **Links**: core repo + `https://docs.ledticker.dev/widgets/rss_feed/`.

  Read `../led-ticker/docs/site/src/content/docs/widgets/rss_feed.mdx` for the exact option descriptions to port into the table.

- [ ] **Step 3: README sanity** — confirm the GIF path resolves (`ls docs/rss.gif`) and the markdown has no broken relative links.

- [ ] **Step 4: Commit**

```bash
git add README.md docs/
git commit -m "docs: README + demo gif for feeds.rss"
```

---

### Task A5: PR for led-ticker-feeds

- [ ] **Step 1: Full green locally**

```bash
make test && make lint && uv run ruff format --check src tests && uv run pyright src
```
Expected: tests pass (coverage ≥90%), lint/format/typecheck clean.

- [ ] **Step 2: Push + PR**

```bash
git push -u origin feat/rss-widget
gh pr create --title "feat: extract rss_feed into led-ticker-feeds (feeds.rss)" --body "$(cat <<'EOF'
## Summary
First widget in the led-ticker-feeds plugin: the RSS/Atom headlines widget, extracted from led-ticker core as `feeds.rss` (modeled on led-ticker-calendar). Repo is shaped so weather can join as `feeds.weather` later.

- `RSSFeedMonitor` ported near-verbatim; imports repointed to the public `led_ticker.plugin` surface (enforced by test_import_purity).
- Ported rss tests (5 classes) + smoke test (`feeds.rss` registers via the entry point) + import-purity tripwire. Coverage ≥90%.
- README with the demo GIF; CI mirrors the other plugins (sibling-checkout of core).

Core removal of `rss_feed` is a follow-up PR in led-ticker (plugin-first sequencing). Spec + plan: docs/superpowers/ in this repo.

## Test plan
- [ ] CI green (lint / typecheck / test)
- [ ] `feeds.rss` resolves via the entry point (test_smoke)
- [ ] src imports only led_ticker.plugin (test_import_purity)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 3: Watch CI; on green, request merge consent.** Do NOT merge without the owner's explicit go-ahead for this PR. After merge, proceed to Phase B.

---

## MERGE GATE

**Phase B must not merge until Phase A (led-ticker-feeds PR) is merged.** Phase B can be *implemented* (branch + PR opened) while A is in review, but core must not drop `rss_feed` on `main` before the plugin is real and installable.

---

## PHASE B — remove rss_feed from core (repo: led-ticker, in a worktree)

### Task B1: Remove the widget + wire the migration

**Files (core):**
- Delete: `src/led_ticker/widgets/rss_feed.py`, `tests/test_widgets/test_rss_feed.py`
- Modify: `src/led_ticker/widgets/__init__.py`, `src/led_ticker/app/factories.py`, `tests/test_widgets/test_crypto_migration.py`, `tests/test_plugin_extraction_readiness.py`

- [ ] **Step 1: worktree** — create a core worktree off latest origin/main (use the using-git-worktrees flow / EnterWorktree), confirm `pwd && git branch --show-current`, fetch + align to origin/main, `uv sync --extra dev`.

- [ ] **Step 2: Delete the widget + its core test**

```bash
git rm src/led_ticker/widgets/rss_feed.py tests/test_widgets/test_rss_feed.py
```

- [ ] **Step 3: `src/led_ticker/widgets/__init__.py`** — remove `rss_feed,` from the auto-import block:

```python
from led_ticker.widgets import (  # noqa: E402, F401
    clock,
    gif,
    message,
    still,
    two_row,
    weather,
)
```

- [ ] **Step 4: `src/led_ticker/app/factories.py`** — add to `_EXTRACTED_TYPES` (after the `"calendar"` entry):

```python
    "rss_feed": (
        "Widget type 'rss_feed' was extracted from led-ticker core; it now ships "
        "in the led-ticker-feeds plugin as 'feeds.rss'.",
        "Install led-ticker-feeds (add it to config/requirements-plugins.txt) "
        'and use type = "feeds.rss".',
    ),
```

- [ ] **Step 5: `tests/test_widgets/test_crypto_migration.py`** — add (mirror the calendar test):

```python
def test_bare_rss_feed_type_raises_migration_to_plugin():
    result = build_widget_cfg_error_for_type("rss_feed")
    assert result is not None
    message, fix = result
    assert "led-ticker-feeds" in message
    assert "feeds.rss" in fix
```

- [ ] **Step 6: `tests/test_plugin_extraction_readiness.py`** — remove the `"widgets/rss_feed.py"` entry from the `_ALLOWED` dict (the file no longer exists; the tripwire would `assert path.exists()`-fail otherwise).

- [ ] **Step 7: Run the affected tests**

```bash
PYTHONPATH=tests/stubs uv run pytest tests/test_widgets/test_crypto_migration.py tests/test_plugin_extraction_readiness.py -q
```
Expected: pass (migration test green; readiness tripwire green with rss removed).

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: remove rss_feed from core (extracted to led-ticker-feeds)"
```

---

### Task B2: requirements + catalog + CLAUDE.md

**Files (core):** `config/requirements-plugins.example.txt`, `src/led_ticker/plugins_catalog.json`, `CLAUDE.md`

- [ ] **Step 1: `config/requirements-plugins.example.txt`** — add before the Calendar line:

```
# RSS/Atom feed headlines (type = "feeds.rss"):
git+https://github.com/JamesAwesome/led-ticker-feeds.git@main
```

- [ ] **Step 2: `src/led_ticker/plugins_catalog.json`** — add a `feeds` entry to the `plugins` array (match the existing calendar entry's shape):

```json
    {
      "name": "feeds",
      "namespace": "feeds",
      "summary": "RSS/Atom feed headlines widget.",
      "homepage": "https://github.com/JamesAwesome/led-ticker-feeds",
      "provides": ["feeds.rss"],
      "sources": [
        { "type": "git", "url": "https://github.com/JamesAwesome/led-ticker-feeds", "ref": "main" }
      ]
    }
```
(Read the existing calendar entry first and match its exact field set/order; if there's a drift test on the catalog, make the entry conform.)

- [ ] **Step 3: `CLAUDE.md`** — add after the crypto line in the Plugin ecosystem section:

```markdown
- [`led-ticker-feeds`](https://github.com/JamesAwesome/led-ticker-feeds) — `feeds.rss`: RSS/Atom feed headlines widget.
```

- [ ] **Step 4: Commit**

```bash
git add config/requirements-plugins.example.txt src/led_ticker/plugins_catalog.json CLAUDE.md
git commit -m "chore: register led-ticker-feeds in requirements, catalog, CLAUDE.md"
```

---

### Task B3: Docs page + demo-TOML handling

**Files (core):** `docs/site/src/content/docs/widgets/rss_feed.mdx`, `docs/site/demos-long/widget-rss_feed.toml` (+ scan other docs for `type = "rss_feed"`)

- [ ] **Step 1: Rewrite `widgets/rss_feed.mdx` to the plugin-note pattern** — mirror `calendar.mdx`: frontmatter description noting it's provided by the led-ticker-feeds plugin (`type = "feeds.rss"`); a lead paragraph; the "to add it to your sign: add `git+https://github.com/JamesAwesome/led-ticker-feeds.git@main` to `config/requirements-plugins.txt` and rebuild" bullet; a "full documentation lives in the led-ticker-feeds README" bullet; keep the demo GIF embed (`/demos-long/widget-rss_feed.gif`) and update any example `type` to `feeds.rss`. Read `calendar.mdx` and follow its exact shape.

- [ ] **Step 2: Demo-TOML — remove from the render pipeline.** The committed GIF stays (the page renders it), but `docs/site/demos-long/widget-rss_feed.toml` uses the now-removed `type` and would fail `make render-long-demos`. Delete it:

```bash
git rm docs/site/demos-long/widget-rss_feed.toml
```
(The GIF `docs/site/public/demos-long/widget-rss_feed.gif` stays — do NOT delete it.) Verify nothing else in the render tooling hard-references the toml: `grep -rn "widget-rss_feed" docs/site Makefile scripts tools 2>/dev/null` — if a manifest lists it, remove that reference too.

- [ ] **Step 3: Update stray `type = "rss_feed"` config literals in other docs** — `grep -rn 'rss_feed' docs/site/src/content/docs/` ; for any page showing a copy-pasteable `type = "rss_feed"` config snippet (e.g. tutorial/showcase/getting-started), update it to `feeds.rss`. Prose mentions of "RSS feed" the feature can stay. Be surgical — only change config-literal `type =` values.

- [ ] **Step 4: docs-lint**

```bash
node --version >/dev/null 2>&1 || source ~/.nvm/nvm.sh
make docs-lint
```
Expected: clean (run `make docs-format` + re-stage if prettier complains).

- [ ] **Step 5: Commit**

```bash
git add docs/
git commit -m "docs: rss_feed page points at led-ticker-feeds plugin; drop demo toml from render pipeline"
```

---

### Task B4: Full core verification + PR

- [ ] **Step 1: Full suite + lint**

```bash
make test
make lint
```
Expected: all pass. Watch specifically for: any test asserting `rss_feed` is a built-in widget type; any config-options drift test; any demo-render test. Fix fallout (e.g. an example `config/*.toml` in core that uses `type = "rss_feed"` → update to `feeds.rss` or remove). Report exact failures verbatim if any.

- [ ] **Step 2: Validate the migration end-to-end**

```bash
printf '[display]\nrows=32\ncols=64\nchain_length=8\ndefault_scale=1\n\n[[playlist.section]]\nmode="swap"\n\n[[playlist.section.widget]]\ntype="rss_feed"\nfeed_url="http://x"\n' > /tmp/rss_mig.toml
uv run led-ticker validate /tmp/rss_mig.toml --json
```
Expected: invalid, with the migration error naming `led-ticker-feeds` + `feeds.rss`.

- [ ] **Step 3: Push + PR (do NOT merge — needs owner consent AND Phase A merged first)**

```bash
git push -u origin <branch>
gh pr create --title "feat: remove rss_feed from core (now led-ticker-feeds: feeds.rss)" --body "$(cat <<'EOF'
## Summary
Removes the rss_feed widget from core now that it ships in the led-ticker-feeds plugin as `feeds.rss` (plugin-first; the plugin PR merged first).

- Deletes `widgets/rss_feed.py` + its core test; adds the `_EXTRACTED_TYPES` migration entry (`rss_feed` → install led-ticker-feeds, use `feeds.rss`) + a migration test.
- Removes the rss entry from the P3 extraction-readiness tripwire.
- Enables led-ticker-feeds by default in `config/requirements-plugins.example.txt`; adds it to `plugins_catalog.json` + the CLAUDE.md ecosystem list.
- Core docs page kept (now points at the plugin, mirroring calendar); demo TOML dropped from the render pipeline (GIF kept).

Depends on led-ticker-feeds being merged. Spec + plan live in the led-ticker-feeds repo.

## Test plan
- [ ] `make test` green; `make lint` / `make docs-lint` clean
- [ ] `led-ticker validate` on a `type = "rss_feed"` config raises the migration error → `feeds.rss`
- [ ] readiness tripwire green with rss removed

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 4: After CI green + Phase A merged, request explicit merge consent.** Merge only on the owner's go-ahead; then clean up the worktree.

---

## Self-review notes
- **Spec coverage:** plugin scaffold/pyproject/register (A1) · widget port w/ public-surface imports (A1) · ported tests + import-purity + smoke (A2) · CI + CLAUDE.md (A3) · README + GIF + demo toml (A4) · plugin PR (A5) · core widget removal + migration entry + migration test + tripwire entry (B1) · requirements-enabled-by-default + catalog + CLAUDE.md ecosystem (B2) · core docs page note + demo-toml removal + stray type literals (B3) · full verify + validate + core PR (B4). All spec sections mapped.
- **Type/name consistency:** package `led_ticker_feeds`; widget `RSSFeedMonitor` in `rss.py`; entry-point name `feeds`; registered type `feeds.rss`; class registered via `api.widget("rss")(RSSFeedMonitor)`. Used identically across A1/A2/A5/B1.
- **Sequencing guard:** explicit MERGE GATE (A before B); both PRs require explicit per-owner merge consent.
- **The `colors` field/module shadow** is the one real code subtlety — flagged in A1 Step 6 with the `as _colors` fallback if it bites.
