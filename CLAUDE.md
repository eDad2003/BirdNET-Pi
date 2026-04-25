# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BirdNET-Pi is a realtime acoustic bird classification system for Raspberry Pi. It runs as a set of Linux systemd services — this repo is developed on Windows but deployed to aarch64/x86_64 Linux. The ML pipeline uses TensorFlow Lite to identify bird species from continuous audio recordings.

## Commands

**Testing:**
```bash
pytest                                        # run all tests
pytest tests/test_analysis.py                 # run a single test file
pytest tests/test_analysis.py::test_function  # run a single test
```

**Linting:**
```bash
flake8                          # lint all Python (max line 160, max complexity 15)
```

**Streamlit app (interactive charts):**
```bash
streamlit run scripts/plotly_streamlit.py
```

## Architecture

The system has three runtime layers:

### 1. Analysis daemon (`scripts/birdnet_analysis.py`)
Watches the `StreamData/` directory via inotify. For each new WAV file, it splits the audio signal, runs TFLite inference, and queues results for reporting. Loads the model once at startup via `utils/models.py`.

### 2. Chart daemon (`scripts/daily_plot.py`)
Runs as a separate systemd service. Generates static daily charts (Matplotlib/Seaborn) written to disk for the web UI to serve.

### 3. Web interface (`homepage/`)
PHP + JavaScript served by Caddy. `api.php` is the REST endpoint consumed by the JS frontend. Configuration is read/written through the web UI which modifies `/etc/birdnet/birdnet.conf`.

### Core utility modules (`scripts/utils/`)
| Module | Responsibility |
|---|---|
| `analysis.py` | TFLite model inference, audio splitting (`splitSignal`), librosa I/O |
| `reporting.py` | Write detections to DB, clip audio snippets, send notifications, BirdWeather integration |
| `db.py` | Read-only SQLite queries (detection counts, aggregations) |
| `helpers.py` | Load `/etc/birdnet/birdnet.conf`, font selection for i18n charts, path constants |
| `models.py` | Model loading and label file management |
| `classes.py` | `Detection` and `ParseFileName` dataclasses |

### Data flow
```
microphone → recorder → StreamData/*.wav
                                ↓
                   birdnet_analysis.py (inotify)
                                ↓
                   utils/analysis.py (TFLite)
                                ↓
                   utils/reporting.py
                    ├── scripts/birds.db (SQLite)
                    ├── clipped audio snippets
                    ├── Apprise notifications
                    └── BirdWeather API
```

## Key Configuration

- **Main config:** `/etc/birdnet/birdnet.conf` (on deployed device)
- **Database:** `scripts/birds.db` (SQLite; replaced MariaDB in earlier versions)
- **Models:** `model/` directory — primary model is `BirdNET_6K_GLOBAL_MODEL.tflite`
- **Linting rules:** `.flake8` — max-line-length=160, max-complexity=15

## Deployment Notes

This codebase targets Linux (Raspberry Pi aarch64 or Debian x86_64). Shell scripts in `scripts/install_*.sh` provision the full environment including systemd services, Caddy web server, GoTTY terminal, and FTP. The Python virtualenv is created during installation. There is no local dev server setup — web UI testing requires a deployed device or VM.

## CI

GitHub Actions runs `flake8` against Python 3.9, 3.11, and 3.13 on PRs (`python-ci.yml`) and runs `pytest` on push to `main` (`python-app.yml`). Both must pass before merging.