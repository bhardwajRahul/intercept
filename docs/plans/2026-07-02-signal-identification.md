# Signal Identification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a bundled JSON signal database (~500 entries) with a scored match API route and a standalone modal overlay accessible from the waterfall and global nav.

**Architecture:** `utils/signal_db.py` loads `data/signals.json` once at startup (lazy, cached). `routes/signalid.py` gains `POST /signalid/match` which calls the pure match function. `static/js/components/signal-id-modal.js` is a standalone IIFE modal called from both `waterfall.js` and a new nav button in `templates/partials/nav.html`.

**Tech Stack:** Python 3.11+, Flask blueprints, vanilla JS (IIFE), JSON data file, existing CSS variables.

## Global Constraints

- All modulation tokens in `signals.json` must be uppercase strings (`WFM`, `FM`, `AM`, `USB`, `LSB`, `NFM`, `FSK`, `OOK`, `PSK`, etc.)
- All frequencies in `signals.json` stored as integers in Hz
- Match scores are 0–100 integer points
- Route added to existing `signalid_bp` blueprint in `routes/signalid.py`; no new blueprint
- No new Python dependencies — stdlib only
- JS follows IIFE pattern consistent with other components: `window.SignalIdModal = (function() { ... })()`
- Modal styled with existing CSS variables only (`--bg-card`, `--accent-cyan`, `--font-mono`, `--text-primary`, `--text-muted`, `--text-secondary`, `--accent-red`, `--bg-input`)
- `signal-id-modal.js` goes in `static/js/components/` (eagerly loaded, not lazy-loaded per mode)

---

## File Map

| Action | File | Responsibility |
|---|---|---|
| Create | `data/signals.json` | Bundled signal database |
| Create | `utils/signal_db.py` | Load + cache signals.json; pure `match_signals()` function |
| Create | `tests/test_signals_json.py` | Schema validation for every entry in signals.json |
| Create | `tests/test_signalid_match.py` | Unit tests for matching algorithm |
| Create | `static/js/components/signal-id-modal.js` | Signal ID modal IIFE component |
| Modify | `routes/signalid.py` | Add `POST /signalid/match` route |
| Modify | `config.py` | Add `REGION = _get_env("REGION", "GLOBAL")` |
| Modify | `templates/index.html` | Add eager `<script>` tag for `signal-id-modal.js` |
| Modify | `templates/partials/nav.html` | Add "Signal ID" button in Intel group |
| Modify | `templates/partials/modes/waterfall.html` | Replace Signal ID sidebar panel with single "Identify Signal" button |
| Modify | `static/js/modes/waterfall.js` | Wire "Identify Signal" button and `handoff('signalid')` to `SignalIdModal.open()` |

---

## Task 1: signals.json seed database + schema validation test

**Files:**
- Create: `data/signals.json`
- Create: `tests/test_signals_json.py`

**Interfaces:**
- Produces: `data/signals.json` — JSON array of signal objects consumed by `utils/signal_db.py` in Task 2

- [ ] **Step 1: Write the schema validation test**

Create `tests/test_signals_json.py`:

```python
"""Schema validation for data/signals.json."""

from __future__ import annotations

import json
from pathlib import Path

SIGNALS_PATH = Path(__file__).resolve().parent.parent / "data" / "signals.json"

VALID_REGIONS = {"GLOBAL", "EU", "US", "UK", "AU"}


def _load() -> list[dict]:
    assert SIGNALS_PATH.exists(), f"signals.json not found at {SIGNALS_PATH}"
    with open(SIGNALS_PATH) as f:
        data = json.load(f)
    assert isinstance(data, list), "signals.json must be a JSON array"
    return data


class TestSignalsJsonSchema:
    def test_file_loads_as_list(self):
        data = _load()
        assert len(data) > 0

    def test_all_ids_unique(self):
        data = _load()
        ids = [s["id"] for s in data]
        assert len(ids) == len(set(ids)), f"Duplicate ids: {[x for x in ids if ids.count(x) > 1]}"

    def test_required_string_fields(self):
        data = _load()
        for s in data:
            for field in ("id", "name", "description"):
                assert field in s, f"Missing '{field}' in signal {s.get('id', '?')}"
                assert isinstance(s[field], str), f"'{field}' must be str in {s['id']}"
                assert s[field].strip(), f"'{field}' must not be empty in {s['id']}"

    def test_frequency_ranges_valid(self):
        data = _load()
        for s in data:
            sid = s.get("id", "?")
            assert "frequency_ranges" in s, f"Missing frequency_ranges in {sid}"
            assert isinstance(s["frequency_ranges"], list), f"frequency_ranges must be list in {sid}"
            assert len(s["frequency_ranges"]) > 0, f"frequency_ranges must not be empty in {sid}"
            for r in s["frequency_ranges"]:
                assert isinstance(r.get("min_hz"), int), f"min_hz must be int in {sid}"
                assert isinstance(r.get("max_hz"), int), f"max_hz must be int in {sid}"
                assert r["min_hz"] > 0, f"min_hz must be > 0 in {sid}"
                assert r["min_hz"] < r["max_hz"], f"min_hz must be < max_hz in {sid}"

    def test_bandwidth_range_valid_or_null(self):
        data = _load()
        for s in data:
            sid = s.get("id", "?")
            assert "bandwidth_range" in s, f"Missing bandwidth_range in {sid}"
            bw = s["bandwidth_range"]
            if bw is None:
                continue
            assert isinstance(bw.get("min_hz"), int), f"bandwidth_range.min_hz must be int in {sid}"
            assert isinstance(bw.get("max_hz"), int), f"bandwidth_range.max_hz must be int in {sid}"
            assert bw["min_hz"] > 0, f"bandwidth_range.min_hz must be > 0 in {sid}"
            assert bw["min_hz"] < bw["max_hz"], f"bandwidth_range.min_hz must be < max_hz in {sid}"

    def test_modulations_uppercase_strings(self):
        data = _load()
        for s in data:
            sid = s.get("id", "?")
            assert "modulations" in s, f"Missing modulations in {sid}"
            assert isinstance(s["modulations"], list), f"modulations must be list in {sid}"
            for m in s["modulations"]:
                assert isinstance(m, str), f"modulation token must be str in {sid}"
                assert m == m.upper(), f"modulation '{m}' must be uppercase in {sid}"
                assert m.strip(), f"modulation token must not be empty in {sid}"

    def test_categories_list_of_strings(self):
        data = _load()
        for s in data:
            sid = s.get("id", "?")
            assert "categories" in s, f"Missing categories in {sid}"
            assert isinstance(s["categories"], list), f"categories must be list in {sid}"
            for c in s["categories"]:
                assert isinstance(c, str) and c.strip(), f"category must be non-empty str in {sid}"

    def test_regions_valid_values(self):
        data = _load()
        for s in data:
            sid = s.get("id", "?")
            assert "regions" in s, f"Missing regions in {sid}"
            assert isinstance(s["regions"], list), f"regions must be list in {sid}"
            assert len(s["regions"]) > 0, f"regions must not be empty in {sid}"
            for r in s["regions"]:
                assert r in VALID_REGIONS, f"Invalid region '{r}' in {sid}. Valid: {VALID_REGIONS}"

    def test_sigidwiki_url_string_or_null(self):
        data = _load()
        for s in data:
            sid = s.get("id", "?")
            assert "sigidwiki_url" in s, f"Missing sigidwiki_url in {sid}"
            url = s["sigidwiki_url"]
            if url is not None:
                assert isinstance(url, str), f"sigidwiki_url must be str or null in {sid}"
                assert url.startswith("https://www.sigidwiki.com/"), f"sigidwiki_url must be sigidwiki.com URL in {sid}"
```

- [ ] **Step 2: Run test — expect file-not-found failure**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signals_json.py -v 2>&1 | head -20
```

Expected: `AssertionError: signals.json not found`

- [ ] **Step 3: Create data/signals.json with initial seed**

Create `data/signals.json` with the 20 signals migrated from `utils/signal_guess.py` (frequency ranges sourced from that file). The full initial seed to write:

```json
[
  {
    "id": "fm-broadcast",
    "name": "FM Broadcast Radio",
    "description": "Commercial FM radio stations transmitting wideband stereo audio. Used worldwide for public broadcasting.",
    "categories": ["broadcast", "commercial", "audio"],
    "frequency_ranges": [{"min_hz": 87500000, "max_hz": 108000000}],
    "bandwidth_range": {"min_hz": 150000, "max_hz": 250000},
    "modulations": ["WFM", "FM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/FM_Broadcast"
  },
  {
    "id": "airband-civil",
    "name": "Airband (Civil Aviation Voice)",
    "description": "Civil aviation voice communications between pilots and air traffic control. AM modulated.",
    "categories": ["aviation", "voice", "aeronautical"],
    "frequency_ranges": [{"min_hz": 118000000, "max_hz": 137000000}],
    "bandwidth_range": {"min_hz": 6000, "max_hz": 10000},
    "modulations": ["AM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/AM_Aviation"
  },
  {
    "id": "ism-433-eu",
    "name": "ISM Device (433 MHz EU)",
    "description": "Industrial, Scientific, and Medical band short-range devices — weather stations, remote controls, car key fobs, doorbells.",
    "categories": ["ism", "telemetry", "consumer", "short-range"],
    "frequency_ranges": [{"min_hz": 433050000, "max_hz": 434790000}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 50000},
    "modulations": ["OOK", "ASK", "FSK", "NFM", "FM"],
    "regions": ["EU", "UK", "AU"],
    "sigidwiki_url": null
  },
  {
    "id": "ism-315-us",
    "name": "ISM Device (315 MHz US)",
    "description": "US ISM band short-range devices — garage openers, remote controls, tire pressure monitors.",
    "categories": ["ism", "telemetry", "consumer", "short-range"],
    "frequency_ranges": [{"min_hz": 314000000, "max_hz": 316000000}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 50000},
    "modulations": ["OOK", "ASK", "FSK"],
    "regions": ["US"],
    "sigidwiki_url": null
  },
  {
    "id": "ism-868-eu",
    "name": "ISM Device (868 MHz EU)",
    "description": "EU sub-GHz ISM band used for LoRa, Z-Wave, smart metering, and IoT sensors.",
    "categories": ["ism", "iot", "lora", "telemetry"],
    "frequency_ranges": [{"min_hz": 863000000, "max_hz": 870000000}],
    "bandwidth_range": {"min_hz": 200, "max_hz": 500000},
    "modulations": ["FSK", "GFSK", "LORA", "OOK"],
    "regions": ["EU", "UK"],
    "sigidwiki_url": null
  },
  {
    "id": "ism-915-us",
    "name": "ISM Device (915 MHz US)",
    "description": "US 915 MHz ISM band used for LoRa, Zigbee, smart meters, and IoT devices.",
    "categories": ["ism", "iot", "lora", "telemetry"],
    "frequency_ranges": [{"min_hz": 902000000, "max_hz": 928000000}],
    "bandwidth_range": {"min_hz": 200, "max_hz": 500000},
    "modulations": ["FSK", "GFSK", "LORA", "OOK"],
    "regions": ["US"],
    "sigidwiki_url": null
  },
  {
    "id": "pmr446",
    "name": "PMR446 (Licence-Free UHF)",
    "description": "European licence-free walkie-talkie band. Used by hikers, event staff, and light commercial users.",
    "categories": ["voice", "pmr", "consumer"],
    "frequency_ranges": [{"min_hz": 446006250, "max_hz": 446193750}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 15000},
    "modulations": ["NFM", "FM"],
    "regions": ["EU", "UK"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/PMR446"
  },
  {
    "id": "frs-gmrs",
    "name": "FRS/GMRS (US Licence-Free UHF)",
    "description": "US Family Radio Service and General Mobile Radio Service. Common consumer walkie-talkie channels.",
    "categories": ["voice", "consumer"],
    "frequency_ranges": [{"min_hz": 462550000, "max_hz": 467725000}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 20000},
    "modulations": ["NFM", "FM"],
    "regions": ["US"],
    "sigidwiki_url": null
  },
  {
    "id": "maritime-vhf",
    "name": "Maritime VHF",
    "description": "Marine VHF radio communications. Channel 16 (156.8 MHz) is the international distress and calling channel.",
    "categories": ["maritime", "voice", "marine"],
    "frequency_ranges": [{"min_hz": 156000000, "max_hz": 174000000}],
    "bandwidth_range": {"min_hz": 12000, "max_hz": 25000},
    "modulations": ["NFM", "FM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/Marine_VHF_Radio"
  },
  {
    "id": "ais",
    "name": "AIS (Automatic Identification System)",
    "description": "Vessel tracking system transmitting position, speed, and identity data. Operates on VHF channels 87B (161.975 MHz) and 88B (162.025 MHz).",
    "categories": ["maritime", "data", "tracking", "navigation"],
    "frequency_ranges": [
      {"min_hz": 161975000, "max_hz": 161975000},
      {"min_hz": 162025000, "max_hz": 162025000}
    ],
    "bandwidth_range": {"min_hz": 12500, "max_hz": 25000},
    "modulations": ["GMSK", "NFM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/Automatic_Identification_System_(AIS)"
  },
  {
    "id": "noaa-weather-radio",
    "name": "NOAA Weather Radio",
    "description": "US National Weather Service continuous weather broadcast on 7 designated VHF channels between 162.400 and 162.550 MHz.",
    "categories": ["broadcast", "weather", "government"],
    "frequency_ranges": [{"min_hz": 162400000, "max_hz": 162550000}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 20000},
    "modulations": ["NFM", "FM"],
    "regions": ["US"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/NOAA_Weather_Radio"
  },
  {
    "id": "dab-digital-radio",
    "name": "DAB Digital Radio",
    "description": "Digital Audio Broadcasting. European digital radio standard replacing FM in many countries. Broad multiplex blocks.",
    "categories": ["broadcast", "digital", "audio"],
    "frequency_ranges": [
      {"min_hz": 174928000, "max_hz": 239200000},
      {"min_hz": 1452960000, "max_hz": 1490624000}
    ],
    "bandwidth_range": {"min_hz": 1536000, "max_hz": 1536000},
    "modulations": ["OFDM"],
    "regions": ["EU", "UK", "AU"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/DAB"
  },
  {
    "id": "amateur-2m",
    "name": "Amateur Radio (2m Band)",
    "description": "2 metre amateur radio band. Used for local voice repeaters, packet radio, satellite communications, and weak signal work.",
    "categories": ["amateur", "ham", "voice"],
    "frequency_ranges": [{"min_hz": 144000000, "max_hz": 146000000}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 200000},
    "modulations": ["NFM", "FM", "USB", "LSB", "AM", "PSK"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": null
  },
  {
    "id": "amateur-70cm",
    "name": "Amateur Radio (70cm Band)",
    "description": "70 centimetre amateur radio band. Common for repeaters, ATV, packet radio, and digital modes.",
    "categories": ["amateur", "ham", "voice"],
    "frequency_ranges": [{"min_hz": 430000000, "max_hz": 440000000}],
    "bandwidth_range": {"min_hz": 10000, "max_hz": 200000},
    "modulations": ["NFM", "FM", "USB", "LSB", "DSTAR", "DMR", "C4FM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": null
  },
  {
    "id": "acars",
    "name": "ACARS (Aircraft Communications)",
    "description": "Aircraft Communications Addressing and Reporting System. Data link for operational airline messages between aircraft and ground stations.",
    "categories": ["aviation", "data", "aeronautical"],
    "frequency_ranges": [
      {"min_hz": 129125000, "max_hz": 129125000},
      {"min_hz": 136900000, "max_hz": 136900000},
      {"min_hz": 131725000, "max_hz": 131725000}
    ],
    "bandwidth_range": {"min_hz": 2400, "max_hz": 6000},
    "modulations": ["AM", "NFM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/ACARS"
  },
  {
    "id": "ads-b",
    "name": "ADS-B (Aircraft Tracking)",
    "description": "Automatic Dependent Surveillance-Broadcast. Aircraft transmit GPS position, altitude, and identity at 1090 MHz for air traffic control.",
    "categories": ["aviation", "data", "tracking", "navigation"],
    "frequency_ranges": [{"min_hz": 1090000000, "max_hz": 1090000000}],
    "bandwidth_range": {"min_hz": 1000000, "max_hz": 2000000},
    "modulations": ["PPM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/ADS-B"
  },
  {
    "id": "pocsag",
    "name": "POCSAG Pager",
    "description": "Post Office Code Standardisation Advisory Group pager protocol. One-way numeric or text messaging to pagers. Common in hospitals and emergency services.",
    "categories": ["pager", "data", "utility"],
    "frequency_ranges": [
      {"min_hz": 138000000, "max_hz": 175000000},
      {"min_hz": 450000000, "max_hz": 470000000}
    ],
    "bandwidth_range": {"min_hz": 8000, "max_hz": 20000},
    "modulations": ["FSK", "NFM"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/POCSAG"
  },
  {
    "id": "lte-700",
    "name": "LTE / 4G (700 MHz)",
    "description": "Long-Term Evolution mobile data network. 700 MHz band used for rural coverage and in-building penetration.",
    "categories": ["cellular", "data", "mobile"],
    "frequency_ranges": [{"min_hz": 698000000, "max_hz": 806000000}],
    "bandwidth_range": {"min_hz": 1400000, "max_hz": 20000000},
    "modulations": ["OFDM", "LTE"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": null
  },
  {
    "id": "gsm-900",
    "name": "GSM 900 (2G Mobile)",
    "description": "Global System for Mobile Communications on 900 MHz band. Voice calls and SMS. Being phased out but still active in many regions.",
    "categories": ["cellular", "voice", "mobile"],
    "frequency_ranges": [{"min_hz": 880000000, "max_hz": 960000000}],
    "bandwidth_range": {"min_hz": 200000, "max_hz": 200000},
    "modulations": ["GMSK"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": "https://www.sigidwiki.com/wiki/GSM"
  },
  {
    "id": "wifi-24ghz",
    "name": "WiFi / Bluetooth (2.4 GHz ISM)",
    "description": "IEEE 802.11 b/g/n/ax WiFi and Bluetooth Classic/BLE sharing the 2.4 GHz ISM band. 13 channels (EU) or 11 (US).",
    "categories": ["ism", "wifi", "bluetooth", "data"],
    "frequency_ranges": [{"min_hz": 2400000000, "max_hz": 2484000000}],
    "bandwidth_range": {"min_hz": 1000000, "max_hz": 22000000},
    "modulations": ["OFDM", "DSSS", "FHSS"],
    "regions": ["GLOBAL"],
    "sigidwiki_url": null
  }
]
```

- [ ] **Step 4: Run schema test — expect all tests pass**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signals_json.py -v
```

Expected output (all pass):
```
tests/test_signals_json.py::TestSignalsJsonSchema::test_file_loads_as_list PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_all_ids_unique PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_required_string_fields PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_frequency_ranges_valid PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_bandwidth_range_valid_or_null PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_modulations_uppercase_strings PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_categories_list_of_strings PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_regions_valid_values PASSED
tests/test_signals_json.py::TestSignalsJsonSchema::test_sigidwiki_url_string_or_null PASSED
```

Note: The AIS entry has two `frequency_ranges` where `min_hz == max_hz` (point frequencies at 161.975 and 162.025 MHz). The schema test permits `min_hz < max_hz`, so adjust those two entries to use a 1 Hz range: `{"min_hz": 161974999, "max_hz": 161975001}` and `{"min_hz": 162024999, "max_hz": 162025001}`. Same for ADS-B at 1090 MHz and ACARS point frequencies.

- [ ] **Step 5: Fix point-frequency entries to use ±500 Hz ranges**

Edit `data/signals.json` — replace exact point frequencies with 1 kHz windows:
- AIS channel 87B: `{"min_hz": 161974500, "max_hz": 161975500}`
- AIS channel 88B: `{"min_hz": 162024500, "max_hz": 162025500}`
- ADS-B: `{"min_hz": 1089500000, "max_hz": 1090500000}`
- ACARS 129.125: `{"min_hz": 129124500, "max_hz": 129125500}`
- ACARS 136.900: `{"min_hz": 136899500, "max_hz": 136900500}`
- ACARS 131.725: `{"min_hz": 131724500, "max_hz": 131725500}`

Re-run tests: `pytest tests/test_signals_json.py -v` — all should pass.

- [ ] **Step 6: Commit**

```bash
git add data/signals.json tests/test_signals_json.py
git commit -m "feat: add signal database seed and schema validation test"
```

---

## Task 2: Database loader + match function

**Files:**
- Create: `utils/signal_db.py`
- Create: `tests/test_signalid_match.py`

**Interfaces:**
- Consumes: `data/signals.json` (Task 1)
- Produces:
  - `load_signals() -> list[dict]` — returns cached signal list
  - `match_signals(*, frequency_mhz, bandwidth_hz=None, modulation=None, region="GLOBAL", limit=8) -> list[dict]` — returns ranked matches with `score: int` and `match_reasons: list[str]` added to each result

- [ ] **Step 1: Write failing tests**

Create `tests/test_signalid_match.py`:

```python
"""Unit tests for the signal database loader and match function."""

from __future__ import annotations

import pytest


class TestLoadSignals:
    def test_returns_list(self):
        from utils.signal_db import load_signals
        signals = load_signals()
        assert isinstance(signals, list)
        assert len(signals) > 0

    def test_cached_on_second_call(self):
        from utils.signal_db import load_signals
        first = load_signals()
        second = load_signals()
        assert first is second  # same object — cached


class TestMatchSignals:
    def test_fm_broadcast_matched_at_center(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5)
        names = [r["name"] for r in results]
        assert "FM Broadcast Radio" in names

    def test_frequency_at_exact_range_boundary_included(self):
        from utils.signal_db import match_signals
        # 87.5 MHz is the lower bound of FM broadcast
        results = match_signals(frequency_mhz=87.5)
        names = [r["name"] for r in results]
        assert "FM Broadcast Radio" in names

    def test_frequency_just_outside_range_excluded(self):
        from utils.signal_db import match_signals
        # 87.499 MHz is just below FM broadcast lower bound
        results = match_signals(frequency_mhz=87.499)
        names = [r["name"] for r in results]
        assert "FM Broadcast Radio" not in names

    def test_no_matches_returns_empty_list(self):
        from utils.signal_db import match_signals
        # 5000 MHz has no signals in our database
        results = match_signals(frequency_mhz=5000.0)
        assert results == []

    def test_results_have_score_field(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5)
        assert len(results) > 0
        for r in results:
            assert "score" in r
            assert isinstance(r["score"], int)
            assert 0 <= r["score"] <= 100

    def test_results_have_match_reasons(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5)
        assert len(results) > 0
        for r in results:
            assert "match_reasons" in r
            assert isinstance(r["match_reasons"], list)

    def test_results_sorted_by_score_descending(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5)
        scores = [r["score"] for r in results]
        assert scores == sorted(scores, reverse=True)

    def test_limit_respected(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5, limit=2)
        assert len(results) <= 2

    def test_limit_clamped_to_minimum_1(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5, limit=0)
        assert len(results) >= 1

    def test_bandwidth_within_range_scores_higher(self):
        from utils.signal_db import match_signals
        # FM broadcast bandwidth_range is 150k–250k Hz; 200k is within
        with_bw = match_signals(frequency_mhz=98.5, bandwidth_hz=200_000)
        without_bw = match_signals(frequency_mhz=98.5)
        fm_with = next(r for r in with_bw if r["name"] == "FM Broadcast Radio")
        fm_without = next(r for r in without_bw if r["name"] == "FM Broadcast Radio")
        assert fm_with["score"] >= fm_without["score"]

    def test_bandwidth_outside_2x_scores_zero_for_bw_criterion(self):
        from utils.signal_db import match_signals
        # FM broadcast max_bw is 250k Hz; 600k is > 2× that
        results = match_signals(frequency_mhz=98.5, bandwidth_hz=600_000)
        fm = next((r for r in results if r["name"] == "FM Broadcast Radio"), None)
        # FM may still appear due to frequency score, but BW reason should not be "within typical"
        if fm:
            assert "bandwidth: within typical" not in fm["match_reasons"]

    def test_modulation_exact_match_scores_higher(self):
        from utils.signal_db import match_signals
        with_mod = match_signals(frequency_mhz=98.5, modulation="WFM")
        without_mod = match_signals(frequency_mhz=98.5)
        fm_with = next(r for r in with_mod if r["name"] == "FM Broadcast Radio")
        fm_without = next(r for r in without_mod if r["name"] == "FM Broadcast Radio")
        assert fm_with["score"] >= fm_without["score"]

    def test_modulation_mismatch_no_mod_reason(self):
        from utils.signal_db import match_signals
        results = match_signals(frequency_mhz=98.5, modulation="LSB")
        fm = next((r for r in results if r["name"] == "FM Broadcast Radio"), None)
        if fm:
            assert "modulation: exact match" not in fm["match_reasons"]

    def test_multi_range_signal_matched_by_any_range(self):
        from utils.signal_db import match_signals
        # POCSAG has ranges in 138-175 MHz and 450-470 MHz
        # 162 MHz is in the first range (maritime VHF area, but also POCSAG territory)
        results_vhf = match_signals(frequency_mhz=155.0)
        results_uhf = match_signals(frequency_mhz=455.0)
        vhf_names = [r["name"] for r in results_vhf]
        uhf_names = [r["name"] for r in results_uhf]
        assert "POCSAG Pager" in vhf_names
        assert "POCSAG Pager" in uhf_names

    def test_region_mismatch_does_not_exclude_signal(self):
        from utils.signal_db import match_signals
        # PMR446 is EU/UK only; should still appear with US region but may score lower
        results = match_signals(frequency_mhz=446.09375, region="US")
        names = [r["name"] for r in results]
        assert "PMR446 (Licence-Free UHF)" in names

    def test_original_signal_dict_not_mutated(self):
        from utils.signal_db import load_signals, match_signals
        original = load_signals()
        first_id_before = original[0]["id"]
        match_signals(frequency_mhz=98.5, modulation="WFM")
        assert original[0]["id"] == first_id_before  # not mutated
        assert "score" not in original[0]  # score not added in-place
```

- [ ] **Step 2: Run tests — expect ImportError or AttributeError**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signalid_match.py -v 2>&1 | head -20
```

Expected: `ImportError: cannot import name 'load_signals' from 'utils.signal_db'`

- [ ] **Step 3: Create utils/signal_db.py**

```python
"""Signal database loader and match engine.

Loads data/signals.json once at startup (lazy, cached). Provides a pure
match_signals() function that scores candidates by frequency, bandwidth,
modulation, and region.
"""

from __future__ import annotations

import json
from pathlib import Path
from typing import Any

from utils.logging import get_logger

logger = get_logger('intercept.signal_db')

_DB_PATH = Path(__file__).resolve().parent.parent / "data" / "signals.json"
_cache: list[dict[str, Any]] | None = None


def load_signals() -> list[dict[str, Any]]:
    """Return cached signal list, loading from JSON on first call."""
    global _cache
    if _cache is not None:
        return _cache

    if not _DB_PATH.exists():
        logger.warning("signals.json not found at %s — signal matching will return no results", _DB_PATH)
        _cache = []
        return _cache

    try:
        with open(_DB_PATH) as f:
            data = json.load(f)
        if not isinstance(data, list):
            raise ValueError("signals.json must be a JSON array")
        _cache = data
        logger.info("Loaded %d signals from %s", len(_cache), _DB_PATH)
    except Exception as exc:
        logger.error("Failed to load signals.json: %s", exc)
        _cache = []

    return _cache


def match_signals(
    *,
    frequency_mhz: float,
    bandwidth_hz: int | None = None,
    modulation: str | None = None,
    region: str = "GLOBAL",
    limit: int = 8,
) -> list[dict[str, Any]]:
    """Return signals ranked by how well they match the given parameters.

    Args:
        frequency_mhz: Centre frequency to match (required).
        bandwidth_hz: Observed signal bandwidth in Hz (optional — improves scoring).
        modulation: Observed modulation token e.g. 'WFM', 'AM' (optional).
        region: User's region code e.g. 'EU', 'US', 'GLOBAL'.
        limit: Maximum number of results to return (clamped to 1–20).

    Returns:
        List of signal dicts (copies) sorted by score descending, each with
        added fields: score (int 0–100), match_reasons (list[str]).
    """
    limit = max(1, min(limit, 20))
    target_hz = frequency_mhz * 1_000_000
    mod_upper = modulation.strip().upper() if modulation else None
    region_upper = region.strip().upper() if region else "GLOBAL"

    candidates: list[dict[str, Any]] = []
    for sig in load_signals():
        ranges = sig.get("frequency_ranges", [])
        if not any(r["min_hz"] <= target_hz <= r["max_hz"] for r in ranges):
            continue
        candidates.append(sig)

    scored: list[dict[str, Any]] = []
    for sig in candidates:
        result = dict(sig)  # shallow copy — do not mutate original
        score = 0
        reasons: list[str] = []

        # --- Frequency centrality (10–40 pts) ---
        ranges = sig.get("frequency_ranges", [])
        best = min(
            ranges,
            key=lambda r: abs(target_hz - (r["min_hz"] + r["max_hz"]) / 2),
        )
        centre = (best["min_hz"] + best["max_hz"]) / 2
        half_span = (best["max_hz"] - best["min_hz"]) / 2 or 1
        centrality = 1.0 - min(abs(target_hz - centre) / half_span, 1.0)
        freq_pts = int(10 + 30 * centrality)
        score += freq_pts
        if centrality >= 0.8:
            reasons.append("frequency: centre of range")
        elif centrality >= 0.4:
            reasons.append("frequency: within range")
        else:
            reasons.append("frequency: edge of range")

        # --- Bandwidth match (0–30 pts) ---
        bw_range = sig.get("bandwidth_range")
        if bandwidth_hz is not None:
            if bw_range is None:
                score += 10
            elif bw_range["min_hz"] <= bandwidth_hz <= bw_range["max_hz"]:
                score += 30
                reasons.append("bandwidth: within typical")
            elif (bandwidth_hz <= bw_range["max_hz"] * 2
                  and bandwidth_hz >= bw_range["min_hz"] // 2):
                score += 15
                reasons.append("bandwidth: near typical")
            # else: 0 pts, no reason added
        else:
            score += 15  # neutral — no bandwidth provided

        # --- Modulation match (0–20 pts) ---
        sig_mods = [m.upper() for m in sig.get("modulations", [])]
        if mod_upper:
            if mod_upper in sig_mods:
                score += 20
                reasons.append("modulation: exact match")
            # else: 0 pts
        else:
            score += 10  # neutral — no modulation provided

        # --- Region match (5–10 pts) ---
        sig_regions = [r.upper() for r in sig.get("regions", [])]
        if "GLOBAL" in sig_regions or region_upper in sig_regions:
            score += 10
        else:
            score += 5

        result["score"] = min(score, 100)
        result["match_reasons"] = reasons
        scored.append(result)

    scored.sort(key=lambda s: s["score"], reverse=True)
    return scored[:limit]
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signalid_match.py -v
```

Expected: all 16 tests pass.

- [ ] **Step 5: Commit**

```bash
git add utils/signal_db.py tests/test_signalid_match.py
git commit -m "feat: add signal_db loader and match_signals scoring function"
```

---

## Task 3: POST /signalid/match route + REGION config

**Files:**
- Modify: `routes/signalid.py` (add route at bottom)
- Modify: `config.py` (add REGION)

**Interfaces:**
- Consumes: `match_signals()` from `utils/signal_db` (Task 2)
- Produces: `POST /signalid/match` returning `{status, frequency_mhz, bandwidth_hz, modulation, matches, match_count, cached}`

- [ ] **Step 1: Add REGION to config.py**

Open `config.py`. Find the section with other `_get_env(...)` calls (around line 410+). Add after the existing env var block — search for `HOST = _get_env(` and add below the existing settings:

```python
# Signal identification region (affects match ranking; does not filter results)
# Valid values: GLOBAL, EU, US, UK, AU
REGION = _get_env("REGION", "GLOBAL")
```

- [ ] **Step 2: Write a failing route test**

Add to `tests/test_signal_guess_api.py` (already exists — append this class at the bottom) or create `tests/test_signalid_match_route.py`:

```python
"""Integration tests for POST /signalid/match route."""

from __future__ import annotations

import json
import pytest


@pytest.fixture
def client():
    from app import create_app
    app = create_app()
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c


class TestSignalidMatchRoute:
    def test_missing_frequency_returns_400(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({}),
            content_type="application/json",
        )
        assert resp.status_code == 400

    def test_invalid_frequency_returns_400(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({"frequency_mhz": -1}),
            content_type="application/json",
        )
        assert resp.status_code == 400

    def test_valid_request_returns_200(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({"frequency_mhz": 98.5}),
            content_type="application/json",
        )
        assert resp.status_code == 200
        data = resp.get_json()
        assert data["status"] == "ok"
        assert isinstance(data["matches"], list)
        assert isinstance(data["match_count"], int)

    def test_fm_broadcast_in_results_at_98mhz(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({"frequency_mhz": 98.5, "modulation": "WFM"}),
            content_type="application/json",
        )
        data = resp.get_json()
        names = [m["name"] for m in data["matches"]]
        assert "FM Broadcast Radio" in names

    def test_no_matches_returns_empty_list(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({"frequency_mhz": 5000.0}),
            content_type="application/json",
        )
        data = resp.get_json()
        assert data["status"] == "ok"
        assert data["matches"] == []
        assert data["match_count"] == 0

    def test_limit_param_respected(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({"frequency_mhz": 98.5, "limit": 2}),
            content_type="application/json",
        )
        data = resp.get_json()
        assert len(data["matches"]) <= 2

    def test_bandwidth_hz_accepted(self, client):
        resp = client.post(
            "/signalid/match",
            data=json.dumps({"frequency_mhz": 98.5, "bandwidth_hz": 200000}),
            content_type="application/json",
        )
        assert resp.status_code == 200

    def test_cached_true_on_second_identical_request(self, client):
        payload = json.dumps({"frequency_mhz": 98.5})
        client.post("/signalid/match", data=payload, content_type="application/json")
        resp2 = client.post("/signalid/match", data=payload, content_type="application/json")
        data = resp2.get_json()
        assert data["cached"] is True
```

- [ ] **Step 3: Run tests — expect 404 (route not yet defined)**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signalid_match_route.py -v 2>&1 | head -30
```

Expected: tests fail with 404 responses.

- [ ] **Step 4: Add POST /signalid/match to routes/signalid.py**

Open `routes/signalid.py`. Add import at top of file alongside existing imports:

```python
import config
from utils.signal_db import match_signals
```

Then append the new route at the bottom of the file (after the existing `sigidwiki_lookup` function):

```python
_match_cache: dict[str, dict[str, Any]] = {}


@signalid_bp.route('/match', methods=['POST'])
def signalid_match() -> Response:
    """Match a signal by frequency, bandwidth, and modulation against the local database."""
    payload = request.get_json(silent=True) or {}

    freq_raw = payload.get('frequency_mhz')
    if freq_raw is None:
        return api_error('frequency_mhz is required', 400)
    try:
        frequency_mhz = float(freq_raw)
    except (TypeError, ValueError):
        return api_error('Invalid frequency_mhz', 400)
    if frequency_mhz <= 0:
        return api_error('frequency_mhz must be positive', 400)

    bw_raw = payload.get('bandwidth_hz')
    bandwidth_hz: int | None = None
    if bw_raw is not None:
        try:
            bandwidth_hz = int(float(bw_raw))
        except (TypeError, ValueError):
            return api_error('Invalid bandwidth_hz', 400)
        if bandwidth_hz <= 0:
            return api_error('bandwidth_hz must be positive', 400)

    modulation = str(payload.get('modulation') or '').strip().upper()[:16] or None

    limit_raw = payload.get('limit', 8)
    try:
        limit = max(1, min(int(limit_raw), 20))
    except (TypeError, ValueError):
        limit = 8

    region = getattr(config, 'REGION', 'GLOBAL')

    cache_key = f'{round(frequency_mhz, 6)}|{bandwidth_hz}|{modulation}|{limit}|{region}'
    cached = _cache_get_match(cache_key)
    if cached is not None:
        return jsonify({
            'status': 'ok',
            'frequency_mhz': round(frequency_mhz, 6),
            'bandwidth_hz': bandwidth_hz,
            'modulation': modulation,
            'cached': True,
            **cached,
        })

    try:
        matches = match_signals(
            frequency_mhz=frequency_mhz,
            bandwidth_hz=bandwidth_hz,
            modulation=modulation,
            region=region,
            limit=limit,
        )
    except Exception as exc:
        logger.error('Signal match failed: %s', exc)
        return api_error('Signal match failed', 502)

    response_data = {
        'matches': matches,
        'match_count': len(matches),
    }
    _cache_set_match(cache_key, response_data)

    return jsonify({
        'status': 'ok',
        'frequency_mhz': round(frequency_mhz, 6),
        'bandwidth_hz': bandwidth_hz,
        'modulation': modulation,
        'cached': False,
        **response_data,
    })


def _cache_get_match(key: str) -> Any | None:
    entry = _match_cache.get(key)
    if not entry:
        return None
    if time.time() >= entry['expires']:
        _match_cache.pop(key, None)
        return None
    return entry['data']


def _cache_set_match(key: str, data: Any, ttl: int = 60) -> None:
    _match_cache[key] = {'data': data, 'expires': time.time() + ttl}
```

- [ ] **Step 5: Run route tests — expect all pass**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signalid_match_route.py -v
```

Expected: all 8 tests pass.

- [ ] **Step 6: Run full test suite to check for regressions**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signals_json.py tests/test_signalid_match.py tests/test_signalid_match_route.py -v
```

Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git add routes/signalid.py config.py tests/test_signalid_match_route.py
git commit -m "feat: add POST /signalid/match route with scoring and caching"
```

---

## Task 4: Signal ID modal component

**Files:**
- Create: `static/js/components/signal-id-modal.js`
- Modify: `templates/index.html` (add eager script tag)

**Interfaces:**
- Consumes: `POST /signalid/match` (Task 3)
- Produces: `window.SignalIdModal` with `open(opts)` and `close()` methods
  - `opts.frequency_mhz` (number, optional) — pre-fills frequency field
  - `opts.modulation` (string, optional) — pre-selects modulation

- [ ] **Step 1: Create static/js/components/signal-id-modal.js**

```javascript
/* Signal identification modal — standalone IIFE component.
 * Usage: SignalIdModal.open({ frequency_mhz: 98.5, modulation: 'WFM' })
 *        SignalIdModal.open({})   // blank fields
 *        SignalIdModal.close()
 */
window.SignalIdModal = (function () {
    'use strict';

    var _modal = null;
    var _backdrop = null;
    var _lastFreq = null;
    var _lastMod = null;

    function _esc(s) {
        return String(s == null ? '' : s)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#39;');
    }

    function _safeSigIdUrl(url) {
        try {
            var p = new URL(String(url || ''));
            if (p.protocol === 'https:' && p.hostname.endsWith('sigidwiki.com')) return p.toString();
        } catch (_) {}
        return null;
    }

    function _build() {
        if (_modal) return;

        _backdrop = document.createElement('div');
        _backdrop.id = 'sigIdBackdrop';
        _backdrop.style.cssText = [
            'position:fixed', 'inset:0', 'z-index:9998',
            'background:rgba(0,0,0,0.65)', 'display:none',
        ].join(';');
        _backdrop.addEventListener('click', close);

        _modal = document.createElement('div');
        _modal.id = 'sigIdModal';
        _modal.style.cssText = [
            'position:fixed', 'top:50%', 'left:50%',
            'transform:translate(-50%,-50%)',
            'z-index:9999',
            'width:min(500px,95vw)', 'max-height:85vh',
            'overflow-y:auto',
            'background:var(--bg-card,#1a1a2e)',
            'border:1px solid rgba(255,255,255,0.12)',
            'border-radius:8px', 'padding:16px',
            'display:none',
            'font-family:var(--font-mono,monospace)',
            'color:var(--text-primary,#e0e0e0)',
            'box-sizing:border-box',
        ].join(';');

        _modal.innerHTML = [
            '<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:14px;">',
            '  <div style="font-size:13px;font-weight:700;">Signal Identification</div>',
            '  <button id="sigIdClose" style="background:none;border:none;color:var(--text-muted,#888);',
            '    cursor:pointer;font-size:20px;line-height:1;padding:0 4px;" aria-label="Close">&times;</button>',
            '</div>',
            '<div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:8px;">',
            '  <div>',
            '    <label for="sigIdFreq" style="font-size:10px;color:var(--text-muted,#888);display:block;margin-bottom:3px;">',
            '      Frequency (MHz)</label>',
            '    <input id="sigIdFreq" type="number" step="0.0001" min="0.001" max="6000"',
            '      style="width:100%;background:var(--bg-input,rgba(255,255,255,0.07));',
            '      border:1px solid rgba(255,255,255,0.15);border-radius:4px;',
            '      color:var(--text-primary,#e0e0e0);padding:5px 8px;',
            '      font-family:var(--font-mono,monospace);font-size:11px;box-sizing:border-box;">',
            '  </div>',
            '  <div>',
            '    <label for="sigIdBw" style="font-size:10px;color:var(--text-muted,#888);display:block;margin-bottom:3px;">',
            '      Bandwidth (kHz)</label>',
            '    <input id="sigIdBw" type="number" step="0.1" min="0.1" placeholder="optional"',
            '      style="width:100%;background:var(--bg-input,rgba(255,255,255,0.07));',
            '      border:1px solid rgba(255,255,255,0.15);border-radius:4px;',
            '      color:var(--text-primary,#e0e0e0);padding:5px 8px;',
            '      font-family:var(--font-mono,monospace);font-size:11px;box-sizing:border-box;">',
            '  </div>',
            '</div>',
            '<div style="display:flex;gap:8px;margin-bottom:12px;align-items:flex-end;">',
            '  <div style="flex:1;">',
            '    <label for="sigIdMod" style="font-size:10px;color:var(--text-muted,#888);display:block;margin-bottom:3px;">',
            '      Modulation</label>',
            '    <select id="sigIdMod" style="width:100%;background:var(--bg-input,rgba(255,255,255,0.07));',
            '      border:1px solid rgba(255,255,255,0.15);border-radius:4px;',
            '      color:var(--text-primary,#e0e0e0);padding:5px 8px;',
            '      font-family:var(--font-mono,monospace);font-size:11px;box-sizing:border-box;">',
            '      <option value="">Auto</option>',
            '      <option value="WFM">WFM</option>',
            '      <option value="NFM">NFM</option>',
            '      <option value="FM">FM</option>',
            '      <option value="AM">AM</option>',
            '      <option value="USB">USB</option>',
            '      <option value="LSB">LSB</option>',
            '      <option value="FSK">FSK</option>',
            '      <option value="OOK">OOK</option>',
            '      <option value="PSK">PSK</option>',
            '    </select>',
            '  </div>',
            '  <button id="sigIdSearch" style="background:var(--accent-cyan,#00c8ff);color:#000;',
            '    border:none;border-radius:4px;padding:6px 18px;font-size:11px;font-weight:700;',
            '    cursor:pointer;font-family:var(--font-mono,monospace);white-space:nowrap;">',
            '    Search</button>',
            '</div>',
            '<div id="sigIdStatus" style="font-size:10px;color:var(--text-muted,#888);min-height:14px;margin-bottom:8px;"></div>',
            '<div id="sigIdResults"></div>',
        ].join('');

        document.body.appendChild(_backdrop);
        document.body.appendChild(_modal);

        document.getElementById('sigIdClose').addEventListener('click', close);
        document.getElementById('sigIdSearch').addEventListener('click', search);
        document.getElementById('sigIdFreq').addEventListener('input', _validateFreq);
    }

    function _validateFreq() {
        var freq = document.getElementById('sigIdFreq');
        var btn = document.getElementById('sigIdSearch');
        if (!freq || !btn) return;
        var val = parseFloat(freq.value);
        var valid = isFinite(val) && val > 0;
        btn.disabled = !valid;
        freq.style.borderColor = (freq.value === '' || valid)
            ? 'rgba(255,255,255,0.15)'
            : 'var(--accent-red,#ff4444)';
    }

    function _setStatus(text, isError) {
        var el = document.getElementById('sigIdStatus');
        if (!el) return;
        el.textContent = text || '';
        el.style.color = isError ? 'var(--accent-red,#ff4444)' : 'var(--text-muted,#888)';
    }

    function _renderResults(matches, freqMhz) {
        var el = document.getElementById('sigIdResults');
        if (!el) return;
        if (!matches.length) {
            el.innerHTML = '<div style="font-size:10px;color:var(--text-muted,#888);">'
                + 'No signals match ' + freqMhz.toFixed(4) + ' MHz'
                + ' — try adjusting the frequency or leaving bandwidth blank.</div>';
            return;
        }
        el.innerHTML = matches.map(function (m, i) {
            var dot = i === 0 ? '●' : '○';
            var score = Number(m.score) || 0;
            var name = _esc(m.name || 'Unknown');
            var desc = _esc(m.description || '');
            var cats = Array.isArray(m.categories) ? m.categories : [];
            var reasons = Array.isArray(m.match_reasons) ? m.match_reasons : [];
            var safeUrl = _safeSigIdUrl(m.sigidwiki_url);
            var ranges = Array.isArray(m.frequency_ranges) ? m.frequency_ranges : [];
            var mods = Array.isArray(m.modulations) ? m.modulations : [];
            var bw = m.bandwidth_range;

            var freqStr = ranges.length
                ? (ranges[0].min_hz / 1e6).toFixed(3) + '–' + (ranges[0].max_hz / 1e6).toFixed(3) + ' MHz'
                : '';
            var bwStr = bw
                ? (bw.min_hz >= 1000 ? (bw.min_hz / 1000).toFixed(0) + 'k' : bw.min_hz)
                  + '–'
                  + (bw.max_hz >= 1000 ? (bw.max_hz / 1000).toFixed(0) + 'k' : bw.max_hz)
                  + ' Hz'
                : '';
            var modStr = mods.join(', ');
            var meta = [freqStr, modStr, bwStr].filter(Boolean).join(' · ');

            var catHtml = cats.slice(0, 5).map(function (c) {
                return '<span style="background:rgba(0,200,255,0.1);color:var(--accent-cyan,#00c8ff);'
                    + 'padding:1px 5px;border-radius:3px;font-size:9px;margin-right:3px;">'
                    + _esc(c) + '</span>';
            }).join('');

            return '<div style="border:1px solid rgba(255,255,255,0.08);border-radius:5px;'
                + 'padding:8px;margin-bottom:6px;">'
                + '<div style="display:flex;justify-content:space-between;align-items:flex-start;'
                + 'gap:8px;margin-bottom:4px;">'
                + '<div style="font-size:11px;font-weight:700;">' + dot + ' ' + name + '</div>'
                + '<div style="display:flex;align-items:center;gap:4px;flex-shrink:0;">'
                + '<div style="width:56px;height:5px;background:rgba(255,255,255,0.1);border-radius:3px;overflow:hidden;">'
                + '<div style="width:' + score + '%;height:100%;background:var(--accent-cyan,#00c8ff);"></div>'
                + '</div><div style="font-size:9px;color:var(--text-muted,#888);">' + score + '</div>'
                + '</div></div>'
                + (meta ? '<div style="font-size:9px;color:var(--text-muted,#888);margin-bottom:4px;">' + _esc(meta) + '</div>' : '')
                + (desc ? '<div style="font-size:10px;color:var(--text-secondary,#b0b0b0);line-height:1.4;margin-bottom:5px;">' + desc + '</div>' : '')
                + (catHtml ? '<div style="margin-bottom:5px;">' + catHtml + '</div>' : '')
                + (reasons.length ? '<div style="font-size:9px;color:var(--text-dim,#666);">'
                    + reasons.map(_esc).join(' · ') + '</div>' : '')
                + (safeUrl ? '<div style="margin-top:5px;"><a href="' + _esc(safeUrl) + '"'
                    + ' target="_blank" rel="noopener noreferrer"'
                    + ' style="font-size:9px;color:var(--accent-cyan,#00c8ff);text-decoration:none;">'
                    + '↗ View on SigID Wiki</a></div>' : '')
                + '</div>';
        }).join('');
    }

    function open(opts) {
        _build();
        opts = opts || {};

        var freqEl = document.getElementById('sigIdFreq');
        var modEl = document.getElementById('sigIdMod');
        var bwEl = document.getElementById('sigIdBw');
        var resultsEl = document.getElementById('sigIdResults');

        if (opts.frequency_mhz != null) {
            freqEl.value = Number(opts.frequency_mhz).toFixed(4);
            _lastFreq = Number(opts.frequency_mhz);
        } else {
            freqEl.value = '';
            _lastFreq = null;
        }

        var modVal = (opts.modulation || '').toUpperCase();
        modEl.value = modVal || '';

        bwEl.value = '';
        resultsEl.innerHTML = '';
        _setStatus('');
        _validateFreq();

        _backdrop.style.display = 'block';
        _modal.style.display = 'block';

        if (!opts.frequency_mhz) {
            setTimeout(function () { freqEl.focus(); }, 50);
        }
    }

    function close() {
        if (!_modal) return;
        _modal.style.display = 'none';
        _backdrop.style.display = 'none';
    }

    function search() {
        var freqEl = document.getElementById('sigIdFreq');
        var modEl = document.getElementById('sigIdMod');
        var bwEl = document.getElementById('sigIdBw');
        var searchBtn = document.getElementById('sigIdSearch');

        var freq = parseFloat(freqEl.value);
        if (!isFinite(freq) || freq <= 0) return;

        var bwKhz = parseFloat(bwEl.value);
        var bwHz = (isFinite(bwKhz) && bwKhz > 0) ? Math.round(bwKhz * 1000) : null;
        var mod = (modEl.value || '').trim().toUpperCase() || null;

        _setStatus('Searching ' + freq.toFixed(4) + ' MHz…');
        document.getElementById('sigIdResults').innerHTML = '';
        searchBtn.disabled = true;

        fetch('/signalid/match', {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({
                frequency_mhz: freq,
                bandwidth_hz: bwHz,
                modulation: mod,
                limit: 8,
            }),
        })
        .then(function (r) { return r.json(); })
        .then(function (data) {
            searchBtn.disabled = false;
            if (!data || data.status !== 'ok') {
                _setStatus('Search failed', true);
                return;
            }
            var matches = data.matches || [];
            if (matches.length) {
                _setStatus(matches.length + ' match' + (matches.length !== 1 ? 'es' : '')
                    + ' for ' + freq.toFixed(4) + ' MHz');
            }
            _renderResults(matches, freq);
        })
        .catch(function () {
            searchBtn.disabled = false;
            _setStatus('Search failed — check connection', true);
            document.getElementById('sigIdResults').innerHTML =
                '<div style="font-size:10px;color:var(--accent-red,#ff4444);">'
                + 'Network error — <button onclick="window.SignalIdModal.search()" '
                + 'style="background:none;border:none;color:var(--accent-cyan,#00c8ff);'
                + 'cursor:pointer;font-size:10px;padding:0;">Retry</button></div>';
        });
    }

    return {open: open, close: close, search: search};
})();
```

- [ ] **Step 2: Add eager script tag to templates/index.html**

In `templates/index.html`, find the block of eagerly-loaded component scripts near the bottom (around line 3663 — the `<script src=".../js/components/signal-guess.js">` line). Add immediately after the last component script tag before the mode-scripts comment:

```html
    <script src="{{ url_for('static', filename='js/components/signal-id-modal.js') }}?v={{ version }}"></script>
```

- [ ] **Step 3: Manually verify the modal renders correctly**

Start the dev server: `sudo -E venv/bin/python intercept.py`

Open the browser console and run:
```javascript
SignalIdModal.open({ frequency_mhz: 98.5, modulation: 'WFM' })
```

Verify:
- Modal and backdrop appear
- Frequency field shows `98.5000`
- Modulation shows `WFM`
- Clicking "Search" fires a network request to `/signalid/match`
- Results render with score bars and match reasons
- FM Broadcast Radio appears as top result
- Clicking × or backdrop closes the modal

Then run: `SignalIdModal.open({})` and verify the frequency field is blank and focused.

- [ ] **Step 4: Commit**

```bash
git add static/js/components/signal-id-modal.js templates/index.html
git commit -m "feat: add SignalIdModal IIFE component with scored results"
```

---

## Task 5: Nav button + waterfall integration

**Files:**
- Modify: `templates/partials/nav.html`
- Modify: `templates/partials/modes/waterfall.html`
- Modify: `static/js/modes/waterfall.js`

**Interfaces:**
- Consumes: `window.SignalIdModal.open(opts)` (Task 4)

- [ ] **Step 1: Add "Signal ID" button to nav.html**

In `templates/partials/nav.html`, find the Intel group `<div class="mode-nav-dropdown-menu">` (the one containing `tscm`, `spystations`, `websdr`, `drone` items — around line 150). Add a Signal ID button as the first item inside that dropdown menu:

```html
            <button type="button" class="mode-nav-btn"
                    onclick="if(window.SignalIdModal) SignalIdModal.open({})"
                    aria-label="Signal ID">
                <span class="nav-icon icon"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="6"/><path d="M17 17l4 4"/><path d="M8 11h6"/><path d="M11 8v6"/></svg></span>
                <span class="nav-label">Signal ID</span>
            </button>
```

The SVG is a magnifying glass with a + symbol — representing "find/identify signal".

- [ ] **Step 2: Replace Signal ID sidebar panel in waterfall.html**

In `templates/partials/modes/waterfall.html`, find and replace the entire Signal Identification `<div class="section">` block (lines 127–154, from `<div class="section">` through `</div>` containing `wfSigIdExternal`):

Replace this entire block:
```html
    <div class="section">
        <h3>Signal Identification</h3>
        <div class="wf-side-help">
            Identify current frequency using local catalog and SigID Wiki matches.
        </div>
        <div class="form-group">
            <label for="wfSigIdFreq">Frequency (MHz)</label>
            <input type="number" id="wfSigIdFreq" value="100.0000" step="0.0001" min="0.001" max="6000">
        </div>
        <div class="form-group">
            <label for="wfSigIdMode">Mode Hint</label>
            <select id="wfSigIdMode">
                <option value="auto" selected>Auto (Current Mode)</option>
                <option value="wfm">WFM</option>
                <option value="fm">NFM</option>
                <option value="am">AM</option>
                <option value="usb">USB</option>
                <option value="lsb">LSB</option>
            </select>
        </div>
        <div class="wf-side-grid-2">
            <button class="preset-btn" onclick="Waterfall.useTuneForSignalId()">Use Tuned</button>
            <button class="preset-btn" onclick="Waterfall.identifySignal()">Identify</button>
        </div>
        <div id="wfSigIdStatus" class="wf-side-status-line">Ready</div>
        <div id="wfSigIdResult" class="wf-side-box" style="display:none;"></div>
        <div id="wfSigIdExternal" class="wf-side-box wf-side-box-muted" style="display:none;"></div>
    </div>
```

With this compact replacement:
```html
    <div class="section">
        <h3>Signal Identification</h3>
        <div class="wf-side-help">
            Identify the current frequency against the local signal database.
        </div>
        <button class="preset-btn" style="width:100%;" onclick="Waterfall.openSignalId()">
            Identify Signal
        </button>
    </div>
```

- [ ] **Step 3: Update waterfall.js — replace signalid handoff and add openSignalId()**

In `static/js/modes/waterfall.js`, find the `handoff('signalid')` branch (around line 1438):

```javascript
            } else if (target === 'signalid') {
                useTuneForSignalId();
                _setHandoffStatus(`Running Signal ID at ${currentFreq.toFixed(4)} MHz`);
                identifySignal().catch((err) => {
                    _setSignalIdStatus(`Signal ID failed: ${err && err.message ? err.message : 'unknown error'}`, true);
                });
```

Replace that branch with:

```javascript
            } else if (target === 'signalid') {
                if (window.SignalIdModal) {
                    SignalIdModal.open({ frequency_mhz: currentFreq, modulation: _getMonitorMode() });
                }
                _setHandoffStatus(`Opening Signal ID for ${currentFreq.toFixed(4)} MHz`);
```

Then find the public API block at the bottom of the IIFE (search for `return {` near the end of waterfall.js). Add `openSignalId` to the returned public API:

Find the existing `return {` block and add `openSignalId` alongside the other exported functions:

```javascript
        openSignalId: function() {
            if (window.SignalIdModal) {
                SignalIdModal.open({
                    frequency_mhz: Number.isFinite(_monitorFreqMhz) ? _monitorFreqMhz : _currentCenter(),
                    modulation: _getMonitorMode(),
                });
            }
        },
```

- [ ] **Step 4: Remove dead Signal ID functions from waterfall.js**

Search `waterfall.js` for these now-unused functions and remove each one entirely:
- `function _setSignalIdStatus(` (around line 250)
- `function _signalIdFreqInput(` (around line 257)
- `function _syncSignalIdFreq(` (around line 261)
- `function _clearSignalIdPanels(` (around line 268)
- `function _signalIdModeHint(` (around line 281)
- `function _renderLocalSignalGuess(` (around line 288)
- `function _renderExternalSignalMatches(` (around line 332)
- `function useTuneForSignalId(` (around line 381)
- `async function identifySignal(` (around line 386)

Also remove `useTuneForSignalId` and `identifySignal` from the public `return {}` block if they are listed there.

Also remove `_safeSigIdUrl` if it is only used by the above functions (check for any other references first with grep).

- [ ] **Step 5: Verify the waterfall still works**

Start the dev server and open the Waterfall mode. Verify:
- "Identify Signal" button appears in the sidebar
- Clicking it opens the Signal ID modal pre-populated with the current tuned frequency and modulation
- The "Signal ID" handoff button (in the Handoff section) also opens the modal
- Waterfall FFT continues running behind the modal
- The rest of the waterfall (bookmarks, scan, monitor, etc.) is unaffected

Also open the nav and verify the "Signal ID" button appears in the Intel dropdown and opens the modal with blank fields.

- [ ] **Step 6: Run full test suite**

```bash
cd /Users/admin/Dev/intercept && pytest tests/test_signals_json.py tests/test_signalid_match.py tests/test_signalid_match_route.py -v
```

Expected: all tests pass with no regressions.

- [ ] **Step 7: Commit**

```bash
git add templates/partials/nav.html templates/partials/modes/waterfall.html static/js/modes/waterfall.js
git commit -m "feat: wire SignalIdModal to waterfall and global nav"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ Bundled JSON database (`data/signals.json`) — Task 1
- ✅ Schema validation test (`tests/test_signals_json.py`) — Task 1
- ✅ Database loader (`utils/signal_db.py`) — Task 2
- ✅ 4-criterion scoring (frequency, bandwidth, modulation, region) — Task 2
- ✅ match_reasons in response — Task 2
- ✅ `POST /signalid/match` route — Task 3
- ✅ REGION config (`INTERCEPT_REGION`) — Task 3
- ✅ 60-second result cache — Task 3
- ✅ Modal IIFE component (`signal-id-modal.js`) — Task 4
- ✅ Pre-populated from waterfall (frequency + modulation) — Task 5
- ✅ Blank entry from global nav — Task 5
- ✅ Optional bandwidth field in modal — Task 4
- ✅ Score bar + match reasons in modal — Task 4
- ✅ SigID Wiki link when url present — Task 4
- ✅ Error states (no matches, network failure, invalid freq) — Task 4
- ✅ Waterfall continues running behind modal — by design (modal is overlay)
- ✅ Existing `/signalid/sigidwiki` route untouched — nothing removed from it
- ✅ `signal_guess.py` untouched — Task 3 imports `match_signals`, not replacing the old module

**Type consistency:**
- `match_signals()` is defined in Task 2 and imported in Task 3 — same signature used throughout
- `SignalIdModal.open(opts)` defined in Task 4, called in Task 5 — consistent
- `bandwidth_hz` is always integer Hz in Python, converted from kHz in JS before the API call
