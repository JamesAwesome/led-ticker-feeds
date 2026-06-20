# led-ticker-feeds — moved & split

The RSS and weather widgets now live in the **[led-ticker-plugins](https://github.com/JamesAwesome/led-ticker-plugins)** monorepo as two separate packages, with new type names:

| was | now | install |
|---|---|---|
| `feeds.rss` | **`rss.feed`** — [`plugins/rss/`](https://github.com/JamesAwesome/led-ticker-plugins/tree/main/plugins/rss) | `git+https://github.com/JamesAwesome/led-ticker-plugins.git@rss-v0.2.0#subdirectory=plugins/rss` |
| `feeds.weather` | **`weather.current`** — [`plugins/weather/`](https://github.com/JamesAwesome/led-ticker-plugins/tree/main/plugins/weather) | `git+https://github.com/JamesAwesome/led-ticker-plugins.git@weather-v0.2.0#subdirectory=plugins/weather` |

This repository is archived (read-only); its history is preserved in the monorepo.
