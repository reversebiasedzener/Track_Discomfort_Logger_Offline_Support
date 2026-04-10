# Track Ride Monitor — Setup Guide
**Solapur Division, Central Railway**

A mobile Progressive Web App for measuring railway track ride quality during inspection runs.
Records band-pass filtered accelerations (0.4–10 Hz) in three axes with GPS, at 5 readings/second, only when speed is above the configured threshold (default 80 km/h). All data saves locally first and syncs to Google Sheets when connected.

---

## Files in this package

| File | Purpose |
|------|---------|
| `index.html` | The complete mobile app (PWA) |
| `manifest.json` | PWA install manifest — enables home-screen installation |
| `sheet-backend.gs` | Google Apps Script — paste into your Google Sheet |
| `README.md` | This file |

---

## Step 1 — Deploy the Google Apps Script backend (~10 minutes)

1. Create a new Google Sheet (e.g. name it "Track Ride Data — Solapur Division")
2. Click **Extensions → Apps Script**
3. Delete any existing code in the editor
4. Open `sheet-backend.gs` and paste the entire contents
5. Click **Deploy → New Deployment**
6. Set:
   - **Type**: Web App
   - **Execute as**: Me
   - **Who has access**: Anyone
7. Click **Deploy** → grant permissions when prompted
8. Copy the **Web App URL** (looks like `https://script.google.com/macros/s/ABC123.../exec`)
9. Keep this URL — you'll paste it into the app's ⚙ Setup screen

### Optional: add conditional formatting
In the Apps Script editor: **Run → addConditionalFormatting**
This automatically colour-codes high filtered acceleration values in the RawLog sheet.

---

## Step 2 — Deploy the app (~5 minutes)

The app must be served over HTTPS for GPS and motion sensors to work.

### Option A: GitHub Pages (free, recommended)
1. Create a GitHub account if needed
2. Create a new repository (e.g. `track-monitor`)
3. Upload `index.html` and `manifest.json`
4. Go to **Settings → Pages → Source: main branch**
5. Your app URL: `https://yourusername.github.io/track-monitor`

### Option B: Any static web host
Upload `index.html` and `manifest.json` to any HTTPS web server.

### Option C: Local testing (Android only)
- Run `npx http-server` in the folder containing the files
- Connect the phone to the same WiFi network
- Access via the PC's local IP address — Android permits DeviceMotion on local networks

---

## Step 3 — Install on phone

### Android (Chrome) — recommended
1. Open the app URL in Chrome
2. Tap **⋮ → Add to Home Screen**
3. The app installs and behaves like a native application

### iPhone (Safari)
1. Open the app URL in **Safari** (not Chrome — iOS restricts motion sensors to Safari)
2. Tap the **Share** button → **Add to Home Screen**
3. When first opening the app, tap **⚙ Setup** — this triggers the iOS motion sensor permission dialog

---

## Step 4 — Configure the app

Open **⚙ Setup** in the app and set:

| Setting | Description | Default |
|---------|-------------|---------|
| Minimum speed for recording | Data only recorded at or above this speed | 80 km/h |
| Lateral & Vertical threshold — ≤110 km/h track | Exceedance level for peak counting | 1.96 m/s² (0.2g) |
| Lateral & Vertical threshold — >110 km/h track | Exceedance level for peak counting | 1.47 m/s² (0.15g) |
| Google Sheets URL | Web App URL from Step 1 | (blank = offline only) |

Thresholds match Indian Railways standards but can be overridden if your division maintains different tolerances.

---

## Using the app — step by step

### Starting a session
1. Tap **Start Session** — a pre-session screen appears
2. Select the **track class**:
   - **Up to 110 km/h** — threshold 1.96 m/s² (0.2g)
   - **Above 110 km/h** — threshold 1.47 m/s² (0.15g)
3. Enter the **run name** (e.g. "Solapur–Pune 15-Jan Run 1")
4. Tap **Confirm & Start**

### During recording

| Action | What happens |
|--------|-------------|
| Train below threshold speed | Strip shows "Waiting for 80 km/h" in amber. Sensor display remains live. No data recorded. |
| Train reaches threshold speed | Strip turns green "Recording". Data writes to local storage every 200 ms. |
| **⏸ Pause** button | Suspends exceedance counting for fX, fY, fZ while data continues recording normally. Use over known-bad junctions or yards to isolate track quality. Paused readings are marked in the log with ⏸. |
| **▶ Resume** button | Resumes exceedance counting. |
| **⚠ MARK DISCOMFORT** | Retroactively flags the previous 5 seconds of data (25 readings) plus the next 5 seconds. Inserts a PEAK_SUMMARY row with the worst filtered values from that window, along with GPS coordinates at the moment of pressing. Works offline. |
| Train drops below threshold | Recording pauses automatically. Resumes when speed returns above threshold. |
| **Stop Session** | Ends recording. All data remains safely in local storage and will sync to Sheets when connected. |

---

## What is displayed on screen

### Filtered acceleration cards (fX, fY, fZ)
All three acceleration values shown are **band-pass filtered (0.4–10 Hz)** — raw values are not displayed. This frequency range corresponds to ISO 2631-1 whole-body vibration sensitivity for passengers seated in a railway vehicle.

- **fX Lateral** — side-to-side sway (blue)
- **fY Vertical** — up-down bounce (green)
- **fZ Longitudinal** — fore-aft jerk from braking/acceleration (purple)

### Exceedance counts and quality rating

Counts increment each time the absolute filtered value crosses the track-class threshold. After the first full kilometre of recording, a **peaks per km** rate is calculated and the count cells are colour-coded:

| Track class | Threshold | Very Good (green) | Good (yellow) | Average (red) |
|-------------|-----------|-------------------|---------------|---------------|
| ≤110 km/h | 1.96 m/s² | < 1.5 peaks/km | 1.5 to 3 peaks/km | > 3 peaks/km |
| >110 km/h | 1.47 m/s² | < 1.0 peaks/km | 1.0 to 2 peaks/km | > 2 peaks/km |

Colour coding applies to fX (Lateral) and fY (Vertical) only. fZ (Longitudinal) is shown as a count without colour coding — longitudinal peaks are generated by rolling stock, not track geometry, and are not covered by the Indian Railways standard.

### Distance
Running distance is computed via Haversine formula from successive GPS coordinates and displayed in the strip and counts header.

---

## Offline behaviour

All data is written to **IndexedDB** on the device first, regardless of connectivity. A background sync loop checks for internet every 5 seconds and pushes unsynced rows to Google Sheets in batches of up to 500.

The storage card shows:
- **Unsynced (pending)** — rows saved locally, not yet in Sheets
- **Synced** — rows confirmed uploaded to Sheets

The **↓ CSV** button exports all locally stored data directly to a CSV file, fully offline. This is the recommended way to retrieve data if internet connectivity is unavailable throughout the run.

The screen stays on automatically during a session via the browser Wake Lock API, so manual screen timeout settings do not need to be changed.

---

## Google Sheets structure

### RawLog sheet
One row per 200 ms reading above the speed threshold:

| Column | Description |
|--------|-------------|
| Timestamp | ISO 8601 UTC |
| Date | yyyy-MM-dd |
| Time | HH:mm:ss.SSS |
| Session | Run name entered at session start |
| Track_Class | '110' (≤110 km/h) or '160' (>110 km/h) |
| Latitude / Longitude | GPS coordinates |
| Speed_kmh | GPS-derived speed |
| Dist_km | Running distance from session start (Haversine) |
| fAx_ms2 (BP 0.4–10Hz) | Filtered lateral acceleration |
| fAy_ms2 (BP 0.4–10Hz) | Filtered vertical acceleration |
| fAz_ms2 (BP 0.4–10Hz) | Filtered longitudinal acceleration |
| fResultant_ms2 | √(fAx²+fAz²) — combined filtered harshness |
| Count_Paused | 1 if exceedance counting was paused for this reading |
| Discomfort_Flag | 1 if within a marked discomfort window; PEAK_SUMMARY_5S for summary rows |

### Discomfort sheet
One row per discomfort event. Contains the peak filtered values from the ±5 second window, GPS location at the moment of button press, speed, and run name.

### Sessions sheet
One row per session. Records session name, first seen, last seen, and total discomfort events.

---

## Analysis in Google Sheets

### Compare runs on the same section
Filter by session name and chart fAx and fAy over distance (Dist_km column) to visualise ride quality along the route. Overlay discomfort markers to correlate with known rough patches.

### Quality threshold from your own data
Once you have a comfortable benchmark run, compute the 90th percentile of filtered values:
```
=PERCENTILE(FILTER(RawLog!I2:I, RawLog!D2:D="YourBenchmarkSessionName"), 0.9)
```
Use this as a custom override threshold in ⚙ Setup for subsequent runs.

### Conditional formatting on peaks per km
The Apps Script `addConditionalFormatting` function applies yellow/red highlighting to columns I–L (filtered values) automatically. Run it once from the Apps Script editor after the first data upload.

---

## Sensor orientation

Mount the phone in **portrait mode** in a holder, screen facing you, during all runs. Do not hold it by hand — body movement introduces noise.

- **fX (Lateral)**: side-to-side (coach sway, track twist)
- **fY (Vertical)**: up-down (dipped joints, corrugation)
- **fZ (Longitudinal)**: fore-aft (braking, acceleration — rolling stock, not track)

The band-pass filter removes gravity bias from fY and eliminates high-frequency noise above 10 Hz that is absorbed by the seat and body before reaching the passenger.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Sync fails | Check the Sheets URL in ⚙ Setup; ensure the Apps Script is deployed with "Anyone" access |
| Speed stays at 0.0 | Grant location permission; GPS needs a clear view of the sky — not through metal roofing |
| No accelerometer data on iPhone | Tap ⚙ Setup first to trigger the iOS motion permission dialog |
| Data not appearing in Sheets | Open Apps Script → **Executions** tab to see error logs |
| App won't install to home screen | Must be served over HTTPS; GitHub Pages is the easiest option |
| Counts not incrementing | Check that the session is running (green strip), speed is above threshold, and counting is not paused |
| Distance stuck at 0.00 km | GPS fix not yet acquired or accuracy too low — wait for "±N m" to appear in the GPS card |

---

## Privacy

All data stays within your Google account. The app communicates only with:
1. The Google Sheets Apps Script URL you configure in Setup
2. Google Fonts CDN (for typography — `fonts.googleapis.com`)

No third-party servers, analytics, or advertising. The app contains no tracking code.
