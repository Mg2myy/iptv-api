# CLAUDE.md — AI Assistant Guide for iptv-api

This document provides the essential context for AI assistants working in this repository.

---

## Project Overview

**iptv-api** is a Python-based IPTV channel aggregation platform that:
- Collects live TV streams from multiple sources (subscriptions, hotel networks, multicast, online search)
- Tests each stream's speed/quality
- Outputs ranked M3U/TXT playlists filtered by resolution and bandwidth
- Serves results via a Flask REST API with optional RTMP/HLS streaming

Current version: **1.7.3** (see `version.json`)

---

## Repository Structure

```
iptv-api/
├── main.py                    # CLI entry point — UpdateSource orchestrator
├── config/
│   ├── config.ini             # Master configuration (141 parameters)
│   └── rtp/                   # Regional multicast config files (20+ regions)
├── updates/                   # Data collection modules
│   ├── subscribe/             # M3U/TXT subscription fetching
│   ├── hotel/                 # Hotel IPTV scraping (Foodie + FOFA APIs)
│   ├── multicast/             # Multicast stream detection
│   ├── online_search/         # Keyword-based search
│   ├── epg/                   # Electronic Program Guide (XML/GZ)
│   ├── proxy/                 # Proxy support
│   └── fofa/                  # FOFA security search integration
├── utils/
│   ├── config.py              # Config management (property-based access)
│   ├── channel.py             # Channel filtering, sorting, speed testing
│   ├── speed.py               # Network speed measurement
│   ├── tools.py               # Helpers: URL parsing, M3U/TXT conversion, logging
│   ├── types.py               # TypedDict definitions (ChannelData, TestResult, …)
│   ├── constants.py           # Path constants and regex patterns
│   ├── alias.py               # Channel name aliasing for fuzzy matching
│   ├── db.py                  # SQLite connection management
│   └── requests/tools.py      # HTTP client with retry logic
├── service/
│   ├── app.py                 # Flask REST API
│   └── rtmp.py                # RTMP/HLS streaming via FFmpeg + Nginx
├── tkinter_ui/                # Desktop GUI (tkinter)
│   ├── tkinter_ui.py          # Main window
│   ├── default.py             # Settings tab
│   └── *.py                   # Per-source and feature tabs
├── output/
│   ├── result.txt / result.m3u   # Generated playlists
│   ├── data/cache.pkl.gz         # Compressed speed-test cache
│   ├── log/                      # speed_test.log, result.log, statistic.log
│   ├── ipv4/ ipv6/               # Protocol-specific output
│   └── epg/                      # EPG data
├── Dockerfile
├── entrypoint.sh
├── nginx.conf.template
├── source.json                # Default source URLs (11 entries)
├── Pipfile / Pipfile.lock     # Python 3.13 dependencies
└── .github/workflows/         # CI/CD (main.yml, docker-build.yml, release.yml)
```

---

## Key Architecture Patterns

### Configuration-Driven Design
Almost all features are toggled via `config/config.ini` boolean flags. The `utils/config.py` module exposes these as typed properties. Always check `config.ini` before assuming a feature is active.

```python
# Access pattern
from utils.config import config
if config.open_speed_test:
    ...
```

### Async/Concurrent Processing
Channel fetching and speed testing use `asyncio` with `aiohttp`. Key concurrency limits:
- `speed_test_limit`: 10 concurrent speed tests (default)
- `rtmp_max_streams`: 10 concurrent RTMP streams

### Type Safety
All major data structures use `TypedDict` from `utils/types.py`. Always import and use these rather than raw dicts:
- `ChannelData` — channel record
- `TestResult` — speed test result

### Output Formats
The pipeline always produces both `.txt` and `.m3u` variants, plus protocol-split `ipv4/` and `ipv6/` directories.

### Caching
Speed test results are cached at `output/data/cache.pkl.gz` using pickle + gzip. The cache is keyed by host, so re-runs reuse prior measurements.

---

## Development Commands

All scripts are defined in `Pipfile`:

```bash
# Run the update pipeline (main entry point)
pipenv run dev

# Start Flask API service (port 8000)
pipenv run service

# Launch tkinter GUI
pipenv run ui

# Docker
pipenv run docker_build
pipenv run docker_run
```

### Python Version
Requires **Python 3.13**. Use `pipenv` for dependency management — do not use `pip` directly.

---

## Flask API Endpoints

Served on port **8000** (default):

| Route | Description |
|-------|-------------|
| `/txt` | Full result in TXT format |
| `/m3u` | Full result in M3U format |
| `/ipv4` | IPv4-only results |
| `/ipv6` | IPv6-only results |
| `/hls` | RTMP/HLS mode results |
| `/epg` | EPG XML data |
| `/update` | Trigger manual update |

WebSocket support is included for real-time progress updates.

---

## Configuration Reference

Key parameters in `config/config.ini`:

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `open_driver` | False | Enable Selenium browser automation |
| `open_epg` | True | Fetch EPG data |
| `open_rtmp` | True | Enable RTMP streaming |
| `open_speed_test` | True | Run speed tests |
| `open_service` | True | Start Flask API |
| `open_hotel` | True | Hotel IPTV sources |
| `open_multicast` | True | Multicast sources |
| `open_online_search` | False | Keyword-based search |
| `speed_test_timeout` | 10s | Per-stream timeout |
| `speed_test_limit` | 10 | Concurrent test slots |
| `min_speed` | 0.5 M/s | Minimum acceptable speed |
| `min_resolution` | 1920x1080 | Minimum resolution filter |
| `update_interval` | 12 hours | Auto-update schedule |
| `app_port` | 8000 | Flask API port |
| `nginx_http_port` | 8080 | Nginx HLS port |
| `nginx_rtmp_port` | 1935 | Nginx RTMP port |

Full parameter documentation: `docs/config_en.md`

---

## Docker Deployment

```bash
docker build -t iptv-api .
docker run -p 8000:8000 -p 8080:8080 -p 1935:1935 iptv-api
```

The `entrypoint.sh` starts three background processes: Nginx, `main.py`, and Gunicorn.

Environment variable overrides: `APP_PORT`, `NGINX_HTTP_PORT`, `NGINX_RTMP_PORT`, `APP_WORKDIR`.

---

## CI/CD Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `main.yml` | Cron: 22:00 & 10:00 UTC + manual | Run update pipeline, commit results to `gd` branch |
| `docker-build.yml` | Manual (master) | Build & push multi-arch Docker image to Docker Hub |
| `release.yml` | Manual (master) | Build Windows GUI executable, create GitHub Release |

The `gd` branch is auto-updated by GitHub Actions bot with force push — do not manually edit it.

---

## Git Branches

| Branch | Purpose |
|--------|---------|
| `master` | Production code |
| `dev` | Development work |
| `gd` | Auto-generated results (GitHub Actions only) |
| `claude/*` | AI assistant feature branches |

**Always develop on the designated `claude/` branch** and push with:
```bash
git push -u origin <branch-name>
```

---

## Testing

There is no automated test suite. Validation happens through:
- Log files in `output/log/` after a pipeline run
- `speed_test.log` — per-stream speed test outcomes
- `result.log` — final channel counts and filtering stats
- `statistic.log` — processing statistics
- `nomatch.log` — channel names that failed alias matching

When debugging channel matching issues, check `utils/alias.py` for the name normalization rules.

---

## Key Conventions

1. **Use `TypedDict` types** from `utils/types.py` for all channel/result data — never raw `dict`.
2. **Feature flags first** — before adding logic, check whether a `config.open_*` flag controls it.
3. **Async for I/O** — network fetching and speed tests must use `asyncio`/`aiohttp`, not blocking `requests`.
4. **No test directory** — validate changes by running `pipenv run dev` and inspecting log output.
5. **Channel aliasing** — when adding new channel name patterns, update `utils/alias.py`, not ad-hoc string matching.
6. **Config changes** — add new parameters to both `config/config.ini` and the `Config` class in `utils/config.py` with a typed property.
7. **Output paths** — always use constants from `utils/constants.py` for file paths; never hardcode `output/` paths.
8. **Pipenv only** — do not add to `requirements.txt`; manage all dependencies through `Pipfile`.
