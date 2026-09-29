# XeWe LED OS frontend — Flask prototype of the LED scheduler, exported to firmware

Personal project (XeWe Labs) · 2026-03-18 → 2026-04-22 · Solo: Max Dokukin · Status: Completed (archived; the UI ships inside XeWe LED OS)

## Overview

This repository holds the web front end of the weekly LED scheduler used by
[XeWe LED OS](https://github.com/xewe-labs/xewe-led-os), and the tool that moves it onto the ESP32. The scheduler was
built and iterated as a small Flask app (`schedule/`), where HTML, CSS and JavaScript can be edited and reloaded in a
browser against a mock backend that speaks the same JSON as the firmware. `export/export.py` then converts the app's
`templates/` and `static/` files into Arduino `PROGMEM` headers plus a sketch that registers one web route per file, so
the same UI can be served by the ESP32's web server. The exported headers (`export/exports/schedule_20260320_143655/`)
are the ones compiled into XeWe LED OS under `src/Modules/Software/SmartHome/WebInterface/{static,templates}/`, with
licence headers and formatting added there.

## Highlights

- Week-view calendar (Monday–Sunday, 24 hours) where each block holds one or more CLI commands such as `$led set_rgb 255 0 0`, a start/end time in minutes from midnight and a display colour (`schedule/static/`, `schedule/app.py`)
- Mock backend with the firmware's API: `GET /schedule`, `GET /schedule/json`, `POST /schedule/set`, `POST /schedule/delete` (`schedule/app.py`)
- One-command export of a Flask UI to ESP32 headers, including a rewrite of Jinja `url_for('static', …)` links to plain `/static/…` paths (`export/export.py`)
- Built in 43 commits, 38 of them between 2026-03-18 and 2026-03-20 (`git log`); used by XeWe LED OS from release 2.2.2 ("Added Scheduler for routine LED control")

## How it works

```
schedule/ (Flask: app.py + templates/ + static/) ──► python export/export.py schedule export/exports
    ──► export/exports/schedule_<timestamp>/{templates,static}/*_html.h / *_js.h / *_css.h + schedule_app.ino + app.py
    ──► headers copied into xewe-led-os/src/Modules/Software/SmartHome/WebInterface/ ──► served by WebServer on port 80
```

- **Flask prototype** (`schedule/app.py`, 100 lines) — in-memory `SCHEDULE_DATA` list of blocks `{id, start_time, end_time, day, displayed_color, commands}`; set creates or updates a block, delete removes one by id.
- **UI** (`schedule/templates/schedule.html` + six files in `schedule/static/`, 831 lines) — `schedule-core.js` (calendar grid, 35 px per hour), `schedule-ui.js` (rendering), `schedule-actions.js` (calls to the backend), `schedule-interactions.js` (mouse/drag handling and the command editor), `schedule-utils.js`, `schedule-style.css` (dark theme); features from the history and code: block colour selection with automatic colour detection from `$led set_rgb` / `$led set_hsv` commands (`detectColorFromCommands`), persistent block copy/paste, a sliding current-time line, day-rollover handling.
- **Exporter** (`export/export.py`, 157 lines) — walks `templates/` and `static/`, keeps only `.html`/`.js`/`.css`, writes each as `static const char NAME[] PROGMEM = R"rawliteral(…)rawliteral";` in `<name>_<ext>.h`, derives the browser route and MIME type, copies `app.py` alongside for porting the backend, and generates `<project>_app.ino` with `server.on(route, HTTP_GET, …)` calls (CORS `*`) and a `TODO` for WiFi setup.
- **LLM porting prompt** (`export/exports/app_py_to_ino_prompt.txt`) — the prompt used to translate the Flask backend in `app.py` into ESP32 C++.

## Results

| Metric | Value | Baseline / note |
|---|---|---|
| Files exported | 7 headers (1 template, 6 static) + 1 sketch | `export/exports/schedule_20260320_143655/` |
| UI size | 831 lines of HTML/CSS/JS | raw `wc -l` |
| In production | XeWe LED OS 2.2.2 and 2.3.0 serve these headers at `/schedule` | firmware copies differ only by licence headers and blank lines |

## Getting started

```bash
cd schedule
python -m venv .venv && source .venv/bin/activate
pip install flask
python app.py                 # Flask debug server; open http://127.0.0.1:5000/schedule

cd ..
python export/export.py schedule export/exports    # writes export/exports/schedule_<YYYYMMDD_HHMMSS>/
```

Requirements: Python 3 and Flask. The generated sketch needs the ESP32 Arduino core (`WiFi.h`, `WebServer.h`) and your
own WiFi setup in `setup()`; in practice the headers are copied into XeWe LED OS, which already provides the web server
and the real scheduler backend.

## Documents

- Firmware that serves this UI: [XeWe LED OS](https://github.com/xewe-labs/xewe-led-os) (`src/Modules/Software/SmartHome/WebInterface/`, `src/Modules/Software/Time/Scheduler/`)
- Project page: https://maxdokukin.com/projects/xewe-led-os
- Exporter: [export/export.py](export/export.py) · latest export: [export/exports/schedule_20260320_143655/](export/exports/schedule_20260320_143655/)
