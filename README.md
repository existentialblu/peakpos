# peakpos

Single-file web tool for computing per-breath inspiratory shape metrics — `peak_pos` and `insp_skew` — from ResMed BRP `.edf` files exported by [OSCAR](https://www.sleepfiles.com/OSCAR/). Drop a folder, get a longitudinal chart, watch nightly medians evolve over months.

Built for personal flow-shape tracking around interventions like surgeries, mask changes, and PAP algorithm tweaks.

## What it does

- Parses ResMed BRP `.edf` (25 Hz inspiratory flow) entirely in the browser.
- Segments breaths via zero-crossing detection on smoothed flow.
- Computes two per-breath shape metrics:
  - **`peak_pos`** — fraction of inspiration at which peak inspiratory flow occurs (0 = onset, 1 = end).
  - **`insp_skew`** — third standardized moment of the inspiratory flow distribution over normalized time.
- Aggregates to nightly median + IQR, picking the longest BRP per night.
- Stacked time-series charts (peak_pos top, insp_skew bottom, shared x-axis) plus a phase-space scatter (peak_pos × insp_skew, color-graded by date).
- User-managed event markers (surgeries, settings changes, etc.) render as dashed vertical lines on both time charts and ring the first night-after on the phase scatter.
- Horizontal-only zoom (drag to select range, double-click / Esc / button to reset; "last 30 days" preset).

## Companion tool — `hjorth.html`

A sibling page in this repo for a different question: not the *shape* of individual breaths, but the *periodic-breathing / loop-gain instability* of the night as a whole.

- Builds a minute-ventilation proxy (mean |flow| in 0.5 s bins → uniform 2 Hz envelope), then a Welch PSD of it.
- Computes **Hjorth descriptors as normalized spectral moments over the loop-gain band (0.005–0.05 Hz, period 20–200 s)**:
  - **complexity** — 1.0 for a clean sinusoidal oscillation (textbook Cheyne–Stokes), higher for broadband / chaotic instability.
  - **period (s)** — the dominant loop-gain cycle length (1 / spectral centroid).
  - **band fraction** — share of envelope power in the band; a relative `csr_frac`-style anchor.
- Per-segment values → nightly median + IQR; click a row for that night's Welch PSD with the band shaded.
- Shares the same scaffold: BRP-only EDF parsing, longest-per-night aggregation, IndexedDB, event markers, horizontal zoom, JSON export — plus a rolling-median trend line, adaptive date axis, optional log-Y, and a hover value readout.

Caveat: the band fraction is *not* numerically comparable to a PLD/minute-ventilation-derived `csr_frac` — different source signal and normalization. Treat complexity and period (normalization-independent ratios) as the headline metrics and track them longitudinally against your event markers.

## What the metrics mean

Both metrics describe *where in inspiration* the flow concentrates. They sit on the same axis but read it differently.

### `peak_pos` — discrete

`argmax(flow) / (n − 1)` over the inspiratory segment.

- **~0.3–0.4**: front-loaded inspiration, characteristic of unobstructed airways.
- **~0.5**: symmetric / triangular.
- **>0.6**: late-loaded ("scooped"), the classic flow-limitation / UARS signature — airway slowly opening under negative pressure.

### `insp_skew` — continuous

Third standardized moment of the flow distribution treated as a probability mass over normalized inspiration time. Sign convention: positive = front-loaded, negative = late-loaded, zero = symmetric.

### Why both

At 25 Hz × ~1.5 s inspiration you have only ~37 inspiratory samples per breath, so `peak_pos = argmax/n` jumps in 2–3% chunks. Subtle reshaping of the inspiratory curve can move the centroid and skewness substantially without nudging argmax to a new bin. Skew is generally the more dynamic metric; peak_pos is the slower / structural readout. Tracking both lets you watch their interaction.

## How to use

1. **Get your data.** OSCAR's data tree lives somewhere like `…/SLEEP DATAS/DATALOG/YYYYMMDD/<files>_BRP.edf`.
2. **Open `index.html`** in a modern browser. For local file-system reads via drag-and-drop, just opening the file works; if you'd rather serve it, any static server does (e.g. `npx serve .` or `python -m http.server`).
3. **Drop your `DATALOG` folder onto the dropzone**, or click to pick a folder. The tool walks subdirectories, ingests every `*_BRP.edf` it finds, and silently skips non-BRP files (`STR.edf`, mask-off fragments, anything missing a `Flow` channel).
4. **Add date markers** in the events panel — surgeries, EPAP/PS changes, mask swaps, anything you want to see on the timeline. Markers persist across reloads.
5. **Drag horizontally** on either time chart to zoom; double-click (or Esc, or "reset zoom") to clear. "Last 30 days" jumps you to the recent end of the record.
6. **Click a row in the table** to expand a per-night histogram of both metrics with the median highlighted.

## Privacy

All parsing happens in the browser. EDF bytes never leave your device. Sessions and events live in IndexedDB (browser-local). "Clear all" wipes the session store; events are config and are kept (delete them individually). The "export json" button gives you a sidecar of your aggregates + events for backup; raw per-breath arrays are not exported (recoverable by re-ingesting the source files).

## Algorithm

Faithful JS port of the inspiratory-shape segmenter in the author's NightPrint Python pipeline. Specifically:

- 100 ms moving-average smoothing of the raw flow channel.
- Inspiration onset = negative→positive zero crossing.
- Exhale onset = first positive→negative crossing within an inspiration.
- Per-breath filters: total breath duration in [1.5 s, 12 s], peak inspiratory flow ≥ 0.05 L/s.
- `peak_pos` and `insp_skew` computed per breath, aggregated to nightly median / IQR.

Verified against the same data as the Python pipeline: median peak_pos values agree to 4 decimal places.

## Caveats

- **Longest BRP per night only.** OSCAR splits sessions on mask-off events; this tool plots the single longest segment per night to keep the timeline interpretable. Multi-segment-per-night is straightforward to add if needed.
- **ResMed only.** Parses standard EDF with a `Flow` signal. Philips DS2-encrypted files are not supported.
- **No clinical claims.** This is a personal-fit instrument, not a medical device. Don't make therapy changes from these numbers without your clinician.
- **Tongue-base assumption is implicit.** `peak_pos` and `insp_skew` are tongue-base / upper-airway readouts in spirit, but they're shape metrics — they reflect anything that reshapes the inspiratory flow profile (mask leak, position, mode changes, illness).

## Browser support

Tested on recent Chrome and Firefox. Folder-walk uses `webkitGetAsEntry`, which works in Chromium-family browsers and modern Firefox. Safari should also work but is untested.

## License

[MIT](LICENSE).

If you fork or build on this for a different project, attribution is appreciated — a link back to this repo or a credit line in your README is enough. The tool is open in spirit, not just in license.
