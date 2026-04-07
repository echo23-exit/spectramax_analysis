# 96-Well Plate Analyzer

A browser-based Streamlit app for designing 96-well plate layouts and analyzing SpectraMax absorbance data.

## Features

- **Template Designer** — click or drag-select wells visually, assign colour-coded conditions, undo, export PNG + CSV
- **Data Analyzer** — upload SpectraMax `.xls` export + template CSV, blank subtraction, endpoint bar charts or kinetic line plots, download all results
- Works entirely in the browser — no coding required after setup

---

## Option A — Run locally on Windows (one-time setup)

1. Install Python from https://python.org (tick "Add Python to PATH" during install).

2. Open Command Prompt and run:
   ```
   pip install streamlit pandas matplotlib plotly numpy openpyxl
   ```

3. Double-click **launch.bat** — the app opens in your browser automatically.

---

## Option B — Deploy on Streamlit Community Cloud (shareable URL for your whole lab)

1. Push this repository to GitHub (public or private).

2. Go to **https://share.streamlit.io** and sign in with GitHub.

3. Click **New app**:
   - Repository: `your-username/your-repo-name`
   - Branch: `main`
   - Main file path: `app.py`

4. Click **Deploy** — Streamlit installs dependencies automatically from `requirements.txt`.

5. Share the generated URL (`https://your-app-name.streamlit.app`) with your lab.

---

## File structure

```
plate_app/
├── app.py              ← entire application (single file)
├── requirements.txt    ← Python dependencies for Streamlit Cloud
├── launch.bat          ← Windows double-click launcher
├── README.md
└── .streamlit/
    └── config.toml     ← theme and server settings
```

---

## SpectraMax export instructions

From SpectraMax Pro: **File → Export → Text** (use default settings).  
The resulting `.xls` file is UTF-16 tab-delimited text — the app handles this automatically.
