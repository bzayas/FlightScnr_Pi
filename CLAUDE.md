# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Claude / agent notes — FlightScnr_Pi

FlightScnr Pi is a flight and marine radar for a Raspberry Pi with a round 4″ 720×720 touch display (rendered at 1080×1080). It has a pygame kiosk UI and a Flask web portal for setup.

## License rules (mandatory)

This repository is **CC BY-NC-SA 4.0**. Full terms: `LICENSE`. Attribution text: `NOTICE`. Same rules also live in `AGENTS.md`, `.cursor/rules/`, and `.github/copilot-instructions.md`.

1. Do not remove or rewrite top-of-file copyright / SPDX / `[AI-DIRECTIVE]` headers.
2. Include that exact header on new or refactored first-party source files (Python `#` form or `/* */` form).
3. Credit Yash Mulgaonkar and https://github.com/yashmulgaonkar/FlightScnr_Pi.
4. Remind the user that commercial use is prohibited without separate permission.

## Boot disclaimer (mandatory)

The boot safety disclaimer is mandatory — never skip, remove, or bypass `SCREEN_DISCLAIMER` on startup; "Don't show again" only arms the 8s auto-continue. See `.cursor/rules/boot-safety-disclaimer.mdc`. (Upstream policy; PR #140 was declined for this reason.)

## FROZEN file (mandatory)

`scripts/release.sh` is FROZEN — never edit, rename, delete, chmod, or reformat it. Old fleet devices have that path mode-dirty; any upstream change breaks their `git pull --ff-only` OTA and strands them. Use `scripts/dev-release.sh` instead. If a task seems to require changing `scripts/release.sh`, stop and ask the user.

## Commands

```bash
# Run tests (pytest; run from flightscnr/, tests add the parent dir to sys.path)
cd flightscnr && python3 -m pytest tests/ -q
cd flightscnr && python3 -m pytest tests/test_cache.py -q          # one file
cd flightscnr && python3 -m pytest tests/test_cache.py::TestTTLCache -q  # one class

# Run the app locally (starts the web portal as a child process, then the pygame UI)
cd flightscnr && python3 flightscnr.py

# Release (bumps VERSION to year.month.day.iteration, commits, tags)
./scripts/dev-release.sh --dry-run
./scripts/dev-release.sh --push --message "Short release note"
```

Notes:

- There is no pytest config file and no lint config. pytest and the runtime deps (`requirements.txt`) must be installed in your Python; the dev Mac has no project venv.
- Pushing a version tag triggers `.github/workflows/release.yml`, which verifies `VERSION` matches the tag and publishes a GitHub Release. The web-portal updater on devices consumes these releases.
- Deployment target is a Raspberry Pi: `install-pi.sh` sets up a venv, X11 kiosk, and the systemd unit `flightscnr/setup/flightscnr.service` (runs `flightscnr.py` as root).

## Architecture

Two processes, one shared data directory:

1. `flightscnr/flightscnr.py` — entry point. It validates config, spawns `web/app.py` (Flask portal, port from `WEB_PORT`) as a subprocess, then runs the pygame display loop until exit.
2. The display process and the portal do not talk over HTTP or IPC. They share JSON/text files in `FLIGHTSCNR_DATA_DIR` (default `/var/lib/flightscnr`): `round_touch_settings.json` (+ `.reload` request files), `tracked_flight.json`, `location.json`, `secrets.json`, `maps/`, etc. The display polls file mtimes to pick up portal changes.

### Configuration

`flightscnr/config.py` reads **only environment variables** — no user defaults in source. Priority (highest wins): `/etc/flightscnr.env` (systemd) → web portal `secrets.json` → `config.h` (both merged into env by `secrets_store.bootstrap_secrets()`) → `.env` in repo root for local dev. `.env.example` documents every variable; `config.h.example` is the user-friendly subset. When you add a config value, document it in `.env.example`.

### Data flow

- `utilities/overhead.py` — background grab thread. It polls aircraft sources, resolves airline branding/logos, and feeds the radar.
- Aircraft sources: `fr24_client.py` (paid FR24 gRPC feed — routes, details, tracked flights), `adsb_client.py` (adsb.fi, free), `dump1090_client.py` (local receiver), `opensky_*`, `adsbexchange_client.py`. `position_source.py` tries sources in portal-configured order until one returns a position; it also computes the Follow-map radius from ground speed.
- Other feeds in `utilities/`: `ais_client.py` (aisstream.io vessels), `atc_audio.py`/`liveatc_client.py`, `temperature.py`/`air_quality.py` (Tomorrow.io weather), earthquake/wildfire overlays live under `display/round_touch/`.
- `utilities/cache.py` — TTL caches and FR24 rate limiting. Respect its polling intervals; FR24 credits are metered.

### Display UI

`display/round_touch/app.py` is the main loop; `screens/` has one module per screen (radar, flight_detail, tracked, clock variants, forecast, info, wifi_setup…). Navigation and gestures: `nav.py`, `gesture_handler.py`, `input_handler.py`. Rendering helpers (map tiles, overlays, tags, theming) are sibling modules in `round_touch/`. `settings.py` owns persisted UI settings and their option lists — the portal writes the same file, so keep option lists in sync when adding a setting (portal UI lives in `web/templates/` + `web/app.py`).

### Versioning

`VERSION` at repo root holds `year.month.day.iteration` (e.g. `2026.8.26.2`). Devices self-update by `git pull --ff-only` via the portal (`setup/portal-update.sh`) — this is why file modes and history on `main` must stay pull-friendly.

## FlightScnr CYD (`cyd/`)

This fork's front page is FlightScnr CYD, a port to the ESP32 "Cheap Yellow Display" (2.8″ ESP32-2432S028R, its two-USB ST7789 twin, and the 4.0″ E32R40T). The Pi app above is unchanged; its README is `FLIGHTSCNR_PI.md`. The user guide is `docs/cyd/` (developer notes: `docs/cyd/development.md`).

```bash
cd cyd/firmware
pio run -e cyd-e32r40t                     # firmware (one image for every board)
make -C sim -j && sim/build/fs_sim [--small] [--landscape] --out shots   # simulator; exits non-zero if swipe/settings checks fail
make -C sim check                          # installer schema (cyd/installer/js/schema.js) matches src/core/config.cpp
make -C test/fetch check                   # HTTP/HTTPS client tests
python3 tools/build_portal.py [--check]    # re-embed cyd/installer as the device portal after editing it
python3 tools/make_gallery.py shots        # regenerate installer + guide screenshots from all four sim sizes
sim/build/fs_sim --gallery --out gallery   # every widget at every slot size + every glyph, for design review
```

- ESP32 without PSRAM: memory is the main constraint. HTTPS is BearSSL in a static block (`src/net/fetch.cpp`); the net task runs on core 0 at priority 1; Traffic and Settings are drawn lists built on demand.
- Settings JSON lives in the `fscfg` partition (`0x3D0000`). Never move partition offsets, and keep stored JSON keys stable (layout keys `infograph`/`modular`/`focus`/`full` are shown as Instruments/Panels/Focus/Full screen).
- The boot safety disclaimer rule applies to the CYD too (`src/ui/screens/boot.cpp`).
- Version in `cyd/VERSION`, tags `cyd-v<version>`. CYD CI (`.github/workflows/cyd.yml`) must never create GitHub Releases, and CYD builds must never use `scripts/release.sh`.
- Update `docs/cyd/` and `cyd/CHANGELOG.md` with user-visible changes.
