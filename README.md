# led-ticker-feeds

Data-feed widgets for [led-ticker](https://github.com/JamesAwesome/led-ticker) — RSS/Atom headlines (`feeds.rss`), with weather planned.

## Installation

```bash
pip install led-ticker-feeds
```

## Usage

```toml
[[sections]]
[[sections.widgets]]
type = "feeds.rss"
feed_url = "https://feeds.bbci.co.uk/news/rss.xml"
```
