# led-ticker-feeds

Data-feed widgets for [led-ticker](https://github.com/JamesAwesome/led-ticker) — RSS/Atom headlines and current weather conditions scrolling across the panel.

![feeds.rss — RSS headlines scrolling across the LED panel](docs/rss.gif)

## Prerequisites

- A working [led-ticker](https://github.com/JamesAwesome/led-ticker) install.
- An RSS or Atom feed URL (for `feeds.rss`) and/or a free [WeatherAPI.com](https://www.weatherapi.com/) API key in `WEATHERAPI_KEY` (for `feeds.weather`).

## Install

This plugin auto-registers via the `led_ticker.plugins` entry point — once the package is installed, no `[plugins]` config change is needed.

**Into a containerized led-ticker (recommended):** add the plugin to `config/requirements-plugins.txt` and rebuild:

```text
git+https://github.com/JamesAwesome/led-ticker-feeds.git@main
```

```bash
docker compose up -d --build
```

For production use, pin to a tag or SHA rather than `@main`:

```text
git+https://github.com/JamesAwesome/led-ticker-feeds.git@v0.1.0
```

**Standalone (a venv that already has led-ticker):**

```bash
pip install "git+https://github.com/JamesAwesome/led-ticker-feeds.git@main"
```

led-ticker isn't on PyPI, so this path only works where led-ticker is already installed. See the led-ticker [Plugins docs](https://docs.ledticker.dev/plugins/) for the constraint-based install the Docker image uses.

Once installed, both `feeds.rss` and `feeds.weather` are available automatically.

## What it provides

Two widgets:

- `type = "feeds.rss"` — fetches an RSS or Atom feed and shows the feed title followed by each headline as its own scrolling `TickerMessage`. A background task refreshes the feed at `update_interval`; the display loop re-reads `feed_stories` on every pass so updates surface within one cycle without restarting.
- `type = "feeds.weather"` — fetches current conditions and temperature from [WeatherAPI.com](https://www.weatherapi.com/) and displays them with a pixel-art condition icon. A two-color design (`font_color` for the label, `font_color_temp` for the temperature) lets you animate one and keep the other steady.

---

## RSS (`feeds.rss`)

New to led-ticker configs? The [first-config tutorial](https://docs.ledticker.dev/tutorial/02-first-config/) walks through the overall structure. The blocks below show only the feeds-specific keys.

### Minimal example

```toml
[[playlist.section]]
mode = "infini_scroll"

[[playlist.section.widget]]
type = "feeds.rss"
feed_url = "https://www.nintendolife.com/feeds/news"
```

### With explicit options (RSS)

```toml
[[playlist.section]]
mode = "infini_scroll"

[[playlist.section.widget]]
type = "feeds.rss"
feed_url = "https://feeds.bbci.co.uk/news/rss.xml"
max_stories = 3
update_interval = 3600
font_color = "rainbow"
```

### Field reference

**`feed_url` is the only required field** — everything below is optional tuning.

| Option | Default | Description |
|--------|---------|-------------|
| `feed_url` | **required** | Full URL of the RSS or Atom feed to fetch (e.g. `"https://www.nintendolife.com/feeds/news"`). |
| `max_stories` | `5` | Maximum number of headlines to pull from the feed per fetch. The feed title is always shown first; stories are capped at this number. |
| `update_interval` | `1800` | Seconds between feed fetches. Default is 30 minutes. Be considerate with public feeds — dropping below 60 seconds may get your IP rate-limited. |
| `font` | `"6x12"` | BDF font (e.g. `"5x8"`, `"6x12"`) or hires font name (e.g. `"Inter-Regular"`). |
| `font_color` | none | Color for all story messages. Constant `[r,g,b]`, `"rainbow"`, `"color_cycle"`, `"random"`, or `{style="gradient", from=[...], to=[...]}`. When unset, the widget cycles through three colors (yellow → red → green) per story. |
| `bg_color` | none | Background fill color applied to every story message. |
| `padding` | `6` | Horizontal padding (logical pixels) added to each story when scrolling. |

---

## Weather (`feeds.weather`)

![feeds.weather — current conditions and temperature on the LED panel](docs/weather.gif)

Current conditions from [WeatherAPI.com](https://www.weatherapi.com/): renders `Label: <condition-icon> <temp>°` on one line and polls in the background. The two-color design lets you color the label (e.g. with `"rainbow"`) and keep the temperature value in a steady high-contrast white — or animate both.

### Prerequisites

A free [WeatherAPI.com](https://www.weatherapi.com/) API key stored in the `WEATHERAPI_KEY` environment variable (or your `.env` file). The widget raises `ValueError` at startup if the variable is unset.

### Minimal example

```toml
[[playlist.section.widget]]
type = "feeds.weather"
location = "Brooklyn"
text = "Brooklyn"
units = "imperial"
```

### With color options

```toml
[[playlist.section.widget]]
type = "feeds.weather"
location = "Brooklyn"
text = "Brooklyn"
units = "imperial"
font_color = "rainbow"
font_color_temp = [255, 255, 255]
```

### Field reference

**`location` and `text` are the only required fields** — everything below is optional tuning.

| Option | Default | Description |
|--------|---------|-------------|
| `location` | **required** | Query string passed to WeatherAPI.com. Accepts a city name (`"Brooklyn"`), ZIP code (`"10001"`), `"lat,lon"` string (`"40.71,-74.01"`), or an inline TOML table (`{lat = 40.71, lon = -74.01}`). Prefer a ZIP or lat/lon when a city name is ambiguous. |
| `text` | **required** | Label shown before the reading (e.g. `"Brooklyn"` → `Brooklyn: ☁ 64F`). |
| `units` | `"imperial"` | `"imperial"` for °F, `"metric"` for °C. |
| `font_color` | `DEFAULT_COLOR` | Color for the label segment. Constant `[r,g,b]`, `"rainbow"`, `"color_cycle"`, `"shimmer"`, `"random"`, or `{style="gradient", from=[...], to=[...]}`. |
| `font_color_temp` | `[255, 255, 255]` | Color for the temperature value. Same accepted forms as `font_color`. Defaults to white so the reading stays high-contrast while the label can use an effect. |
| `show_icon` | `true` | When `true`, draws the pixel-art condition icon (sun, cloud, rain, snow, thunder, fog, partly cloudy) between the label and temperature. Set `false` to show the condition string as text instead. |
| `bg_color` | none | Background fill color applied to the widget. |
| `update_interval` | `10800` | Seconds between fetches (default is 3 hours). The WeatherAPI free tier has monthly request quotas — dropping below 60 seconds risks exhausting the quota. |

**Two-color note:** `font_color` colors the label; `font_color_temp` colors the temperature value. Set both to `"rainbow"` if you want them to match, or set them independently to animate one while keeping the other steady.

---

## Development

led-ticker isn't on PyPI, so this plugin resolves it from a sibling checkout. Clone both side by side:

```
~/projects/.../led-ticker
~/projects/.../led-ticker-feeds
```

```bash
uv sync --extra dev      # resolves led-ticker from ../led-ticker
uv run pytest -q
uv run ruff check src tests
uv run pyright src
```

The test suite needs the rgbmatrix stub on the path — `pyproject.toml` wires this automatically via `pythonpath = ["../led-ticker/tests/stubs"]`. To run tests manually without the project config:

```bash
PYTHONPATH=../led-ticker/tests/stubs uv run pytest -q
```

The plugin imports only the public `led_ticker.plugin` surface — `tests/test_import_purity.py` enforces it.

## Links

- [led-ticker](https://github.com/JamesAwesome/led-ticker) — the core project
- [Docs site](https://docs.ledticker.dev) · [RSS feed widget reference](https://docs.ledticker.dev/widgets/rss_feed/) · [Weather widget reference](https://docs.ledticker.dev/widgets/weather/)
