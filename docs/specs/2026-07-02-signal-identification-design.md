# Signal Identification — Design Spec
**Date:** 2026-07-02
**Status:** Approved

## Overview

Extend Intercept's existing signal identification capability with a bundled local signal database (~500 signals) and a rich modal overlay that works both from the waterfall (pre-populated) and standalone from the global nav (manual entry). The goal is Artemis-like offline signal lookup integrated directly into Intercept's workflow — no manual browsing required.

---

## Context

Intercept already has two signal identification mechanisms:

| Mechanism | Location | Coverage | Offline? |
|---|---|---|---|
| Local heuristic engine | `utils/signal_guess.py` | ~20 signal types | Yes |
| SigID Wiki API proxy | `routes/signalid.py` | ~500+ signals | No |

Both surface results in the waterfall sidebar Signal ID panel. The heuristic engine is limited in coverage; the Wiki proxy requires internet and has latency. This design replaces the heuristic engine's role with a bundled database and adds a richer modal UI.

---

## Architecture

```
data/signals.json              ← bundled signal database (~500 entries)
        ↓ loaded at startup, cached in memory
routes/signalid.py             ← new POST /signalid/match route
        ↓ JSON response with ranked matches + match_reasons
static/js/signal-id-modal.js  ← standalone modal component
        ↑ called from waterfall.js (pre-populated) and nav (blank)
```

### What stays unchanged
- `utils/signal_guess.py` — left in place but no longer called from the new modal. Its role is superseded by the database-driven matcher. The existing `/receiver/signal/guess` route remains registered.
- `routes/signalid.py /signalid/sigidwiki` — untouched. SigID Wiki links in the modal results make it redundant as a parallel lookup; matched results carry a `sigidwiki_url` field instead.
- Waterfall SSE streaming, SDR process management, all other routes

---

## Database Schema

**File:** `data/signals.json`
**Format:** JSON array of signal objects
**Source:** Seeded from SigID Wiki (CC BY-SA), hand-curated, version-controlled in the repo

### Signal object

```json
{
  "id": "fm-broadcast",
  "name": "FM Broadcast Radio",
  "description": "Commercial FM radio stations. Wideband stereo audio, typically 87.5–108 MHz. Used worldwide for public broadcasting.",
  "categories": ["broadcast", "commercial", "audio"],
  "frequency_ranges": [
    { "min_hz": 87500000, "max_hz": 108000000 }
  ],
  "bandwidth_range": { "min_hz": 150000, "max_hz": 250000 },
  "modulations": ["WFM", "FM"],
  "regions": ["GLOBAL"],
  "sigidwiki_url": "https://www.sigidwiki.com/wiki/FM_Broadcast"
}
```

### Field definitions

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Unique kebab-case slug |
| `name` | string | yes | Display name |
| `description` | string | yes | Plain-English, one paragraph max |
| `categories` | string[] | yes | e.g. `broadcast`, `aviation`, `maritime`, `utility`, `amateur`, `military`, `telemetry` |
| `frequency_ranges` | `{min_hz, max_hz}`[] | yes | List — some signals appear on multiple non-contiguous bands |
| `bandwidth_range` | `{min_hz, max_hz}` \| null | yes | Null if unknown or highly variable |
| `modulations` | string[] | yes | Uppercase tokens matching waterfall mode selector: `WFM`, `FM`, `AM`, `USB`, `LSB`, `FSK`, `OOK`, `PSK`, etc. |
| `regions` | string[] | yes | `GLOBAL`, `EU`, `US`, `UK`, `AU` — used to deprioritise region-mismatched results |
| `sigidwiki_url` | string \| null | yes | Direct link shown as "View reference" in modal. Null if no page exists. |

### Initial seed
The ~20 signals already in `utils/signal_guess.py` are migrated to this schema as the starting point. The file grows to ~500 signals seeded from SigID Wiki structured data.

---

## Backend: `/signalid/match`

**Route:** `POST /signalid/match` added to the existing `signalid_bp` blueprint in `routes/signalid.py`

### Request

```json
{
  "frequency_mhz": 100.1,
  "bandwidth_hz": 200000,
  "modulation": "WFM",
  "limit": 8
}
```

| Field | Required | Default | Constraints |
|---|---|---|---|
| `frequency_mhz` | yes | — | > 0 |
| `bandwidth_hz` | no | null | > 0 if provided |
| `modulation` | no | null | Truncated to 16 chars, uppercased |
| `limit` | no | 8 | Clamped to 1–20 |

### Matching algorithm

**Step 1 — Hard filter (frequency overlap)**
Discard any signal where the query frequency does not fall within at least one `frequency_range`. Reduces ~500 candidates to typically 3–15.

**Step 2 — Score each candidate (0–100 points)**

| Criterion | Max pts | Logic |
|---|---|---|
| Frequency centrality | 40 | How centred the query is within the matched range. Dead centre = 40, at the range boundary = 10. |
| Bandwidth match | 30 | Only if `bandwidth_hz` provided. Within `bandwidth_range` = 30, within 2× = 15, outside = 0. Signal has null `bandwidth_range` = 10 (neutral). If `bandwidth_hz` not provided, all signals score 15 (neutral). |
| Modulation match | 20 | Query modulation in signal's `modulations` = 20. No modulation provided = 10 (neutral). Mismatch = 0. |
| Region match | 10 | Signal `regions` includes user's configured region or `GLOBAL` = 10, else 5. User region read from `config.py` (`INTERCEPT_REGION`, default `GLOBAL`). |

**Step 3 — Sort and annotate**
Sort by score descending. Attach `match_reasons` list to each result (e.g. `["frequency: centre of range", "bandwidth: within typical", "modulation: exact match"]`). Return top `limit` results.

### Response

```json
{
  "status": "ok",
  "frequency_mhz": 100.1,
  "bandwidth_hz": 200000,
  "modulation": "WFM",
  "matches": [
    {
      "id": "fm-broadcast",
      "name": "FM Broadcast Radio",
      "description": "Commercial FM radio stations...",
      "categories": ["broadcast", "commercial", "audio"],
      "frequency_ranges": [{ "min_hz": 87500000, "max_hz": 108000000 }],
      "bandwidth_range": { "min_hz": 150000, "max_hz": 250000 },
      "modulations": ["WFM", "FM"],
      "regions": ["GLOBAL"],
      "sigidwiki_url": "https://www.sigidwiki.com/wiki/FM_Broadcast",
      "score": 87,
      "match_reasons": [
        "frequency: centre of range",
        "bandwidth: within typical",
        "modulation: exact match"
      ]
    }
  ],
  "match_count": 1,
  "cached": false
}
```

### Caching
Results cached in-process for 60 seconds keyed by `{frequency_mhz}|{bandwidth_hz}|{modulation}|{limit}`. The database itself is loaded once at startup and never re-read during a session.

### Error cases
- `frequency_mhz` missing or invalid → 400
- `data/signals.json` missing or malformed at startup → route returns 503 with message
- No matches → 200 with `matches: []`

---

## Frontend: Modal Component

### Files
- `static/js/signal-id-modal.js` — standalone IIFE module (`SignalIdModal`)
- Modal HTML injected into DOM on first call (not in any template)
- Styled with existing CSS variables, no new stylesheet required

### Entry points

**From the waterfall** (`waterfall.js`):
```js
SignalIdModal.open({ frequency_mhz: _monitorFreqMhz, modulation: _getMonitorMode() });
```
Replaces the current inline Signal ID sidebar panel. The "Identify Signal" button in the waterfall sidebar triggers this.

**From the global nav** (`templates/partials/nav.html`):
A "Signal ID" nav link calls `SignalIdModal.open({})` — opens with blank fields.

### Modal layout

```
┌─ Signal Identification ──────────────────── [×] ─┐
│                                                    │
│  Frequency   [  100.0000  ] MHz                   │
│  Bandwidth   [  optional  ] kHz  (improves match) │
│  Modulation  [ WFM ▾ ]                            │
│                                          [Search] │
├────────────────────────────────────────────────── │
│  ● FM Broadcast Radio              ████████ 87    │
│    87.5–108 MHz · WFM · Wideband                  │
│    Commercial FM radio. Stereo audio broadcast…   │
│    [broadcast] [commercial] [audio]               │
│    Frequency: centre of range · Modulation: exact │
│    ↗ View on SigID Wiki                           │
│  ─────────────────────────────────────────────    │
│  ○ RDS Data (FM subcarrier)        ████░░░░ 52    │
│    …                                              │
└────────────────────────────────────────────────── ┘
```

### Behaviour
- Frequency pre-filled from caller; blank if opened from nav
- Bandwidth field placeholder: "optional — improves matching", value in kHz (converted to Hz before API call)
- Modulation pre-filled from caller or defaults to `WFM`
- Search button disabled if frequency field is empty or invalid
- "Search" fires `POST /signalid/match` — results render inline, modal stays open
- Top result shows filled dot (●), rest show open dot (○)
- Score shown as a proportional bar + integer (0–100)
- `match_reasons` shown as a compact line of text below the signal name
- `sigidwiki_url` shown as "↗ View on SigID Wiki" link (opens new tab); not rendered if null
- `[×]` and clicking the backdrop close the modal
- Waterfall continues running behind the modal; no state is lost

### Error states in modal
| Condition | Display |
|---|---|
| Frequency empty/invalid | Search button disabled, field outlined red |
| No matches returned | "No signals match [X] MHz — try adjusting the frequency or leaving bandwidth blank" |
| Network/server error | "Search failed" + Retry button |
| `sigidwiki_url` null | Link not rendered |

---

## Testing

### `tests/test_signalid_match.py`
Unit tests for the matching algorithm as a pure function (no Flask test client needed for most):

- Frequency exactly at range boundary → included
- Frequency 1 Hz outside range → excluded
- Signal with multiple `frequency_ranges` → matched by whichever range contains the query
- Bandwidth within range → score 30; at 2× → score 15; outside 2× → score 0
- Bandwidth not provided → all signals score 15 neutral
- Modulation exact match → score 20; not provided → 10; mismatch → 0
- No matches → empty list, 200 response
- `limit` clamping (0 → 1, 25 → 20)

### `tests/test_signals_json.py`
Schema validation test — loads `data/signals.json` and asserts every entry has:
- Required fields present and correct types
- `min_hz < max_hz` in all frequency and bandwidth ranges
- `min_hz > 0` in all ranges
- `id` is unique across all entries
- `modulations` tokens are uppercase strings

### No mocking required
The matcher is a pure function over the in-memory database. Flask route tests use the real JSON file.

---

## Out of Scope

- Auto-bandwidth measurement from FFT
- Audio sample upload and DSP analysis
- Waterfall screenshot / image analysis
- Periodic database sync or remote update
- Audio sample or waterfall image hosting
- Region auto-detection from IP
- Standalone Signal Library browse mode (natural phase 2)
