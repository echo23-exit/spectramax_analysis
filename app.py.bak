"""
96-Well Plate Analyzer — Streamlit Web App
==========================================
Tab 1 : Template Designer  — click wells on a visual plate, assign conditions & colours
Tab 2 : Data Analyzer      — upload SpectraMax .xls + template, blank-subtract, plot
Tab 3 : Help
"""

from __future__ import annotations
import csv
import io
import re
from pathlib import Path

import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from matplotlib.patches import Circle, FancyBboxPatch

import numpy as np
import pandas as pd
import plotly.graph_objects as go
import streamlit as st

# ─────────────────────────────────────────────────────────────────────────────
# CONSTANTS
# ─────────────────────────────────────────────────────────────────────────────
ROWS = list("ABCDEFGH")
COLS = list(range(1, 13))

DEFAULT_PALETTE = [
    "#E63946", "#457B9D", "#2DC653", "#F4A261", "#7B2D8B",
    "#06D6A0", "#FB8500", "#264653", "#C77DFF", "#0077B6",
    "#AACC00", "#FF006E", "#8338EC", "#3A86FF", "#FFBE0B",
    "#F72585", "#4CC9F0", "#80B918", "#E9C46A", "#6A4C93",
    "#1D3557", "#A8DADC", "#FF595E", "#6A994E", "#BC4749",
    "#5F0F40", "#9A031E", "#0F4C5C", "#E36414", "#386641",
    "#6B4226", "#344E41",
]

EMPTY_COLOR   = "#EBEBEB"
SELECTED_COLOR = "#FFD700"
PLATE_BG      = "#D8D8D8"

# ─────────────────────────────────────────────────────────────────────────────
# HELPERS
# ─────────────────────────────────────────────────────────────────────────────
def well_sort_key(well: str):
    m = re.match(r"^([A-Ha-h])(\d{1,2})$", str(well).strip())
    if not m:
        return (999, 999)
    return (ROWS.index(m.group(1).upper()), int(m.group(2)))


def hms_to_seconds(hms: str) -> int:
    h, m, s = (int(x) for x in str(hms).split(":"))
    return h * 3600 + m * 60 + s


def parse_well_range(text: str) -> list[str]:
    """
    Convert user text into a list of well IDs.  Supports:
      - Single well:          A1
      - Comma list:           A1, B2, C3
      - Same-row range:       A1:A6   →  A1 A2 A3 A4 A5 A6
      - Same-col range:       A1:D1   →  A1 B1 C1 D1
      - Rectangular block:    A1:C3   →  3×3 = 9 wells
    Tokens are case-insensitive and whitespace-tolerant.
    Returns wells sorted in plate order; silently skips invalid tokens.
    """
    result: set[str] = set()
    for token in re.split(r"[,\s]+", text.strip()):
        token = token.strip().upper()
        if not token:
            continue
        if ":" in token:
            parts = token.split(":")
            if len(parts) != 2:
                continue
            m1 = re.match(r"^([A-H])(\d{1,2})$", parts[0].strip())
            m2 = re.match(r"^([A-H])(\d{1,2})$", parts[1].strip())
            if not m1 or not m2:
                continue
            ri1 = ROWS.index(m1.group(1)); ci1 = int(m1.group(2))
            ri2 = ROWS.index(m2.group(1)); ci2 = int(m2.group(2))
            if ri1 > ri2: ri1, ri2 = ri2, ri1
            if ci1 > ci2: ci1, ci2 = ci2, ci1
            for ri in range(ri1, ri2 + 1):
                for ci in range(ci1, ci2 + 1):
                    if 1 <= ci <= 12:
                        result.add(f"{ROWS[ri]}{ci}")
        else:
            m = re.match(r"^([A-H])(\d{1,2})$", token)
            if m and 1 <= int(m.group(2)) <= 12:
                result.add(token)
    return sorted(result, key=well_sort_key)


# ─────────────────────────────────────────────────────────────────────────────
# PLATE VISUALISATION  (Plotly — display only, no drag-select)
# ─────────────────────────────────────────────────────────────────────────────
def make_plate_figure(assignments: dict, selected_wells: list | None = None) -> go.Figure:
    """Return a Plotly scatter figure representing the 96-well plate."""
    sel = set(selected_wells or [])

    x_vals, y_vals, fills, borders, bwidths = [], [], [], [], []
    hover, well_ids = [], []

    for ri, row in enumerate(ROWS):
        for ci, col in enumerate(COLS):
            wid = f"{row}{col}"
            info = assignments.get(wid)

            if wid in sel:
                fc, bc, bw = SELECTED_COLOR, "#CC8800", 3.0
                htxt = f"<b>{wid}</b><br>Selected — click again to deselect"
            elif info:
                fc, bc, bw = info["color"], "#404040", 1.5
                htxt = f"<b>{wid}</b><br>{info['name']}<br>Replicate {info.get('replicate','-')}"
            else:
                fc, bc, bw = EMPTY_COLOR, "#AAAAAA", 0.8
                htxt = f"<b>{wid}</b><br>Empty"

            x_vals.append(ci)
            y_vals.append(ri)
            fills.append(fc)
            borders.append(bc)
            bwidths.append(bw)
            hover.append(htxt)
            well_ids.append(wid)

    fig = go.Figure(go.Scatter(
        x=x_vals, y=y_vals,
        mode="markers",
        marker=dict(
            size=32,
            color=fills,
            line=dict(color=borders, width=bwidths),
            symbol="circle",
        ),
        customdata=well_ids,
        hovertext=hover,
        hoverinfo="text",
        selected=dict(marker=dict(opacity=1)),
        unselected=dict(marker=dict(opacity=1)),
    ))

    fig.update_layout(
        xaxis=dict(tickvals=list(range(12)), ticktext=[str(c) for c in COLS],
                   side="top", showgrid=False, zeroline=False,
                   range=[-0.8, 11.8], tickfont=dict(size=13)),
        yaxis=dict(tickvals=list(range(8)), ticktext=ROWS,
                   autorange="reversed", showgrid=False, zeroline=False,
                   range=[-0.8, 7.8], tickfont=dict(size=13)),
        plot_bgcolor=PLATE_BG,
        paper_bgcolor="white",
        margin=dict(l=36, r=10, t=52, b=10),
        height=360,
        showlegend=False,
        dragmode=False,
    )
    return fig


# ─────────────────────────────────────────────────────────────────────────────
# MATPLOTLIB PLATE EXPORT
# ─────────────────────────────────────────────────────────────────────────────
def draw_plate_png(assignments: dict, conditions_list: list, title: str = "Plate Layout") -> bytes:
    """Return PNG bytes of the print-ready plate layout."""
    A4_W, A4_H = 11.693, 8.268
    fig = plt.figure(figsize=(A4_W, A4_H), dpi=200)
    fig.patch.set_facecolor("white")

    ax = fig.add_axes([0.07, 0.20, 0.86, 0.70])
    ax.set_facecolor("white")
    ax.axis("off")
    ax.set_aspect("equal", adjustable="box")

    SP, R, PAD = 1.0, 0.38, 0.30
    ax.set_xlim(-(PAD + 0.6), 11 * SP + PAD + 0.1)
    ax.set_ylim(-(PAD + 0.1), 7 * SP + PAD + 0.9)

    # Plate body
    ax.add_patch(FancyBboxPatch(
        (-PAD, -PAD), 11 * SP + 2 * PAD, 7 * SP + 2 * PAD,
        boxstyle="round,pad=0.18", linewidth=1.5,
        edgecolor="#999999", facecolor="#E0E0E0", zorder=0,
    ))

    # Column labels
    for ci, col in enumerate(COLS):
        ax.text(ci, 7 * SP + PAD + 0.25, str(col),
                ha="center", va="bottom", fontsize=9, fontweight="bold", color="#444")

    # Row labels
    for ri, row in enumerate(ROWS):
        ax.text(-(PAD + 0.22), (7 - ri) * SP, row,
                ha="right", va="center", fontsize=9, fontweight="bold", color="#444")

    # Wells
    for ci, col in enumerate(COLS):
        for ri, row in enumerate(ROWS):
            wid = f"{row}{col}"
            info = assignments.get(wid)
            cx, cy = ci * SP, (7 - ri) * SP
            fc = info["color"] if info else "#F2F2F2"
            ax.add_patch(Circle((cx + 0.025, cy - 0.025), R,
                                 facecolor="#00000020", linewidth=0, zorder=1))
            ax.add_patch(Circle((cx, cy), R, facecolor=fc,
                                 edgecolor="#606060", linewidth=0.8, zorder=2))

    # Title
    fig.text(0.07 + 0.86 / 2, 0.935, title,
             ha="center", va="bottom", fontsize=14,
             fontweight="bold", color="#1A1A1A", transform=fig.transFigure)

    # Legend
    if conditions_list:
        leg = fig.add_axes([0.07, 0.01, 0.86, 0.17])
        leg.set_facecolor("white")
        leg.axis("off")
        n = len(conditions_list)
        ncols = min(n, 6)
        col_w = (0.86 * A4_W) / ncols
        for idx, cond in enumerate(conditions_list):
            cx2 = (idx % ncols) * col_w
            cy2 = 0.12 - (idx // ncols) * 0.055
            leg.add_patch(Circle((cx2 + 0.12, cy2), 0.09,
                                  facecolor=cond["color"], edgecolor="#606060",
                                  linewidth=0.6, zorder=3))
            leg.text(cx2 + 0.27, cy2 + 0.03, cond["name"],
                     ha="left", va="center", fontsize=8.5, fontweight="bold", color="#1A1A1A")
            wells_txt = ", ".join(cond.get("wells", []))
            leg.text(cx2 + 0.27, cy2 - 0.025, wells_txt,
                     ha="left", va="center", fontsize=6.5, color="#777")
        leg.set_xlim(0, 0.86 * A4_W)
        leg.set_ylim(-0.08, 0.20)

    buf = io.BytesIO()
    fig.savefig(buf, format="png", dpi=200, bbox_inches="tight", facecolor="white")
    plt.close(fig)
    buf.seek(0)
    return buf.getvalue()


# ─────────────────────────────────────────────────────────────────────────────
# CSV EXPORT  (template)
# ─────────────────────────────────────────────────────────────────────────────
def assignments_to_csv_bytes(assignments: dict) -> bytes:
    fields = ["Well", "Row", "Column", "Well_Number",
              "Condition", "Color_Hex", "Replicate", "Status",
              "Orientation", "Start_Well", "Layout_Mode"]
    rows = []
    wn = 1
    for row in ROWS:
        for col in COLS:
            wid = f"{row}{col}"
            info = assignments.get(wid)
            if info:
                rows.append(dict(Well=wid, Row=row, Column=col, Well_Number=wn,
                                 Condition=info["name"], Color_Hex=info["color"],
                                 Replicate=info.get("replicate", 1), Status="assigned",
                                 Orientation="manual", Start_Well=info.get("start_well", wid),
                                 Layout_Mode="manual_gui"))
            else:
                rows.append(dict(Well=wid, Row=row, Column=col, Well_Number=wn,
                                 Condition="", Color_Hex="", Replicate="",
                                 Status="empty", Orientation="", Start_Well="",
                                 Layout_Mode="manual_gui"))
            wn += 1
    buf = io.StringIO()
    w = csv.DictWriter(buf, fieldnames=fields)
    w.writeheader()
    w.writerows(rows)
    return buf.getvalue().encode()


# ─────────────────────────────────────────────────────────────────────────────
# SPECTRAMAX PARSER  (faithful port from notebook)
# ─────────────────────────────────────────────────────────────────────────────
def _decode(file_bytes: bytes) -> list[str]:
    for enc in ("utf-16", "utf-16-le", "utf-8-sig", "utf-8", "latin-1"):
        try:
            text = file_bytes.decode(enc)
            if text:
                return text.splitlines()
        except Exception:
            continue
    raise ValueError("Cannot decode file. Try exporting from SpectraMax as UTF-16 text.")


def _extract_plate_format(lines: list[str]) -> str:
    head = "\n".join(lines[:10]).lower()
    if "plateformat\tkinetic" in head or "kinetic" in head:
        return "kinetic"
    if "plateformat\tendpoint" in head or "endpoint" in head:
        return "endpoint"
    raise ValueError(
        "Cannot detect PlateFormat from file header. "
        "Please export from SpectraMax using File → Export → Text (default settings)."
    )


def _find_header_index(lines: list[str]) -> int:
    for i, line in enumerate(lines):
        if "Temperature" in line and re.search(r"\t1\t2\t3\t4\t5\t6\t7\t8\t9\t10\t11\t12", line):
            return i
    raise ValueError(
        "Cannot find plate header row (Temperature … 1…12). "
        "Make sure you are uploading a SpectraMax text export."
    )


def _parse_endpoint(lines: list[str]) -> pd.DataFrame:
    h = _find_header_index(lines)
    records, row_idx = [], 0
    for line in lines[h + 1:]:
        s = line.strip()
        if not s or s.startswith("~End") or s.startswith("Original"):
            if s.startswith("~End"):
                break
            continue
        fields = line.split("\t")
        vals = fields[2:14]
        if len(vals) < 12:
            continue
        row_letter = ROWS[row_idx]
        for ci, raw in enumerate(vals, start=1):
            try:
                v = float(raw.strip())
            except ValueError:
                v = np.nan
            records.append({"Well": f"{row_letter}{ci}", "Row": row_letter,
                             "Column": ci, "Value": v, "Assay_Type": "endpoint"})
        row_idx += 1
        if row_idx == 8:
            break
    if row_idx != 8:
        raise ValueError(f"Expected 8 plate rows, found {row_idx}. Is this an endpoint export?")
    return pd.DataFrame(records)


def _parse_kinetic(lines: list[str]) -> pd.DataFrame:
    h = _find_header_index(lines)
    records = []
    i = h + 1
    while i < len(lines):
        line = lines[i]
        s = line.strip()
        if not s:
            i += 1
            continue
        if s.startswith("~End") or s.startswith("Original"):
            break
        fields = line.split("\t")
        first = str(fields[0]).strip()
        if not re.match(r"^\d{2}:\d{2}:\d{2}$", first):
            i += 1
            continue
        time_hms = first
        time_s = hms_to_seconds(time_hms)
        time_min = time_s / 60.0
        block = [line] + lines[i + 1: i + 8]
        if len(block) < 8:
            raise ValueError(f"Incomplete kinetic block at {time_hms}.")
        for row_idx, bline in enumerate(block):
            bfields = bline.split("\t")
            vals = bfields[2:14]
            if len(vals) < 12:
                raise ValueError(f"Expected 12 columns in block at {time_hms}, row {ROWS[row_idx]}.")
            rl = ROWS[row_idx]
            for ci, raw in enumerate(vals, start=1):
                try:
                    v = float(raw.strip())
                except ValueError:
                    v = np.nan
                records.append({"Time_HMS": time_hms, "Time_s": time_s, "Time_min": time_min,
                                 "Well": f"{rl}{ci}", "Row": rl, "Column": ci,
                                 "Value": v, "Assay_Type": "kinetic"})
        i += 8
    if not records:
        raise ValueError("No kinetic blocks found. Is this a kinetic SpectraMax export?")
    return pd.DataFrame(records)


def parse_spectramax(file_bytes: bytes) -> tuple[str, pd.DataFrame]:
    """Return (assay_type, df).  assay_type is 'endpoint' or 'kinetic'."""
    lines = _decode(file_bytes)
    fmt = _extract_plate_format(lines)
    df = _parse_endpoint(lines) if fmt == "endpoint" else _parse_kinetic(lines)
    return fmt, df


# ─────────────────────────────────────────────────────────────────────────────
# TEMPLATE LOADER
# ─────────────────────────────────────────────────────────────────────────────
def load_template_csv(csv_bytes: bytes):
    """Return (assigned_df, cond_order, color_map)."""
    df = pd.read_csv(io.BytesIO(csv_bytes))
    missing = {"Well", "Condition", "Color_Hex"} - set(df.columns)
    if missing:
        raise ValueError(f"Template CSV is missing columns: {sorted(missing)}")

    df["Well"] = df["Well"].astype(str).str.strip().str.upper()
    df["Condition"] = df["Condition"].fillna("").astype(str).str.strip()
    df["Color_Hex"] = df["Color_Hex"].fillna("").astype(str).str.strip()

    if "Status" not in df.columns:
        df["Status"] = df["Condition"].apply(lambda x: "assigned" if x else "empty")
    else:
        df["Status"] = df["Status"].fillna("").astype(str).str.strip().str.lower()

    if "Replicate" not in df.columns:
        df["Replicate"] = np.nan
    else:
        df["Replicate"] = pd.to_numeric(df["Replicate"], errors="coerce")

    if "Well_Number" not in df.columns:
        df["Well_Number"] = df["Well"].map(lambda w: well_sort_key(w)[0] * 12 + well_sort_key(w)[1])

    assigned = df[(df["Status"] == "assigned") & (df["Condition"] != "")].copy()
    if assigned.empty:
        raise ValueError("Template CSV has no assigned wells.")

    cond_order = (
        assigned.groupby("Condition", sort=False)["Well_Number"]
        .min().sort_values().index.tolist()
    )
    color_map = (
        assigned.groupby("Condition", sort=False)["Color_Hex"]
        .first().to_dict()
    )
    return assigned, cond_order, color_map


def template_csv_to_assignments(csv_bytes: bytes) -> dict:
    """Quick conversion for the plate preview in Tab 2."""
    try:
        assigned, _, _ = load_template_csv(csv_bytes)
        out = {}
        for _, row in assigned.iterrows():
            wid = str(row["Well"]).strip()
            out[wid] = {"name": row["Condition"], "color": row["Color_Hex"],
                        "replicate": row.get("Replicate", 1)}
        return out
    except Exception:
        return {}


# ─────────────────────────────────────────────────────────────────────────────
# ANALYSIS
# ─────────────────────────────────────────────────────────────────────────────
def merge_with_template(raw_df: pd.DataFrame, assigned_df: pd.DataFrame) -> pd.DataFrame:
    keep = ["Well", "Condition", "Color_Hex", "Well_Number", "Replicate"]
    keep = [c for c in keep if c in assigned_df.columns]
    merged = raw_df.merge(assigned_df[keep], on="Well", how="inner")
    merged = merged[merged["Condition"].astype(str).str.strip() != ""].copy()
    return merged


def apply_blank_subtraction(merged_df: pd.DataFrame,
                             pairs: list[dict],
                             assay_type: str) -> tuple[pd.DataFrame, pd.DataFrame]:
    """
    pairs: list of {"Condition": ..., "Blank_Condition": ...}
    Returns (corrected_df, pair_detail_df).
    """
    out = merged_df.copy().reset_index(drop=True)
    out["_row_id"] = np.arange(len(out))
    out["Blank_Subtracted"] = False
    out["Raw_Value"]       = out["Value"]
    out["Corrected_Value"] = out["Value"]
    out["Blank_Mean"]      = np.nan
    out["Blank_Condition"] = ""

    pair_rows = []
    is_kinetic = (str(assay_type).lower() == "kinetic")

    for pair in pairs:
        data_c  = pair["Condition"]
        blank_c = pair["Blank_Condition"]
        dm = out["Condition"].astype(str) == str(data_c)
        bm = out["Condition"].astype(str) == str(blank_c)
        data_df  = out.loc[dm].copy()
        blank_df = out.loc[bm].copy()

        if data_df.empty:
            raise ValueError(f"No rows for data condition '{data_c}'.")
        if blank_df.empty:
            raise ValueError(f"No rows for blank condition '{blank_c}'.")

        if is_kinetic:
            blank_means = (blank_df.groupby(["Time_s", "Time_min", "Time_HMS"],
                                             sort=False)["Value"]
                           .mean().rename("Blank_Mean").reset_index())
            corrected = data_df.drop(columns=["Blank_Mean"], errors="ignore").merge(
                blank_means, on=["Time_s", "Time_min", "Time_HMS"], how="left")
        else:
            blank_mean = float(blank_df["Value"].mean())
            corrected = data_df.copy()
            corrected["Blank_Mean"] = blank_mean

        corrected["Raw_Value"]       = corrected["Value"]
        corrected["Corrected_Value"] = corrected["Raw_Value"] - corrected["Blank_Mean"]
        corrected["Blank_Subtracted"] = True
        corrected["Blank_Condition"]  = blank_c
        corrected["Value"]            = corrected["Corrected_Value"]

        for col in ["Value", "Raw_Value", "Corrected_Value",
                    "Blank_Mean", "Blank_Subtracted", "Blank_Condition"]:
            out.loc[corrected["_row_id"].values, col] = corrected[col].values

        pair_rows.append(corrected[
            ["Condition", "Blank_Condition", "Well", "Raw_Value",
             "Blank_Mean", "Corrected_Value"]
            + (["Time_s", "Time_min", "Time_HMS"] if is_kinetic else [])
        ].copy())

    out = out.drop(columns=["_row_id"], errors="ignore")
    pair_df = pd.concat(pair_rows, ignore_index=True) if pair_rows else pd.DataFrame()
    return out, pair_df


def endpoint_summary(merged_df: pd.DataFrame, cond_order: list) -> pd.DataFrame:
    s = (merged_df.groupby("Condition", sort=False)["Value"]
         .agg(["mean", "std", "count"])
         .rename(columns={"mean": "Mean", "std": "SD", "count": "N"})
         .reset_index())
    s["SD"] = s["SD"].fillna(0.0)
    s["Condition"] = pd.Categorical(s["Condition"], categories=cond_order, ordered=True)
    return s.sort_values("Condition").reset_index(drop=True)


def kinetic_summary(merged_df: pd.DataFrame, cond_order: list) -> pd.DataFrame:
    s = (merged_df.groupby(["Condition", "Time_s", "Time_min", "Time_HMS"], sort=False)["Value"]
         .agg(["mean", "std", "count"])
         .rename(columns={"mean": "Mean", "std": "SD", "count": "N"})
         .reset_index())
    s["SD"] = s["SD"].fillna(0.0)
    s["Condition"] = pd.Categorical(s["Condition"], categories=cond_order, ordered=True)
    return s.sort_values(["Condition", "Time_s"]).reset_index(drop=True)


# ─────────────────────────────────────────────────────────────────────────────
# PLOTS  (return PNG bytes — faithful port)
# ─────────────────────────────────────────────────────────────────────────────
def plot_endpoint(summary_df: pd.DataFrame, color_map: dict,
                  title: str, y_label: str) -> bytes:
    n = len(summary_df)
    fig, ax = plt.subplots(figsize=(max(8, 0.65 * n + 4), 6))
    x = np.arange(n)
    colors = [color_map.get(str(c), "#808080") for c in summary_df["Condition"]]
    ax.bar(x, summary_df["Mean"], yerr=summary_df["SD"],
           capsize=4, color=colors, edgecolor="black", linewidth=0.8)
    ax.set_xticks(x)
    ax.set_xticklabels(summary_df["Condition"].astype(str), rotation=45, ha="right")
    ax.set_ylabel(y_label)
    ax.set_title(title)
    ax.spines[["top", "right"]].set_visible(False)
    ax.grid(axis="y", alpha=0.25)
    fig.tight_layout()
    buf = io.BytesIO()
    fig.savefig(buf, format="png", dpi=300, bbox_inches="tight", facecolor="white")
    plt.close(fig)
    buf.seek(0)
    return buf.getvalue()


def plot_kinetic(summary_df: pd.DataFrame, color_map: dict,
                 title: str, y_label: str) -> bytes:
    n_cond = summary_df["Condition"].nunique()
    fig, ax = plt.subplots(figsize=(max(10, min(18, 8 + n_cond * 0.35)), 6.5))
    for cond, g in summary_df.groupby("Condition", sort=False, observed=False):
        g = g.sort_values("Time_s")
        x = g["Time_min"].to_numpy()
        y = g["Mean"].to_numpy()
        sd = g["SD"].fillna(0.0).to_numpy()
        color = color_map.get(str(cond), "#808080")
        ax.plot(x, y, label=str(cond), linewidth=2, color=color)
        ax.fill_between(x, y - sd, y + sd, alpha=0.18, color=color)
    ax.set_xlabel("Time (min)")
    ax.set_ylabel(y_label)
    ax.set_title(title)
    ax.spines[["top", "right"]].set_visible(False)
    ax.grid(alpha=0.25)
    ax.legend(title="Condition", bbox_to_anchor=(1.02, 1),
              loc="upper left", borderaxespad=0, frameon=False)
    fig.tight_layout()
    buf = io.BytesIO()
    fig.savefig(buf, format="png", dpi=300, bbox_inches="tight", facecolor="white")
    plt.close(fig)
    buf.seek(0)
    return buf.getvalue()


# ─────────────────────────────────────────────────────────────────────────────
# SESSION STATE
# ─────────────────────────────────────────────────────────────────────────────
def _init():
    defaults = {
        "assignments":     {},   # well → {name, color, replicate, start_well}
        "conditions_list": [],   # [{name, color, wells:[...]}]
        "selected_wells":  [],   # currently highlighted
        "undo_stack":      [],   # for undo
        "template_csv":    None, # bytes — shared with Tab 2
        "chart_ver":       0,    # bump to force chart re-render
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v


# ─────────────────────────────────────────────────────────────────────────────
# TAB 1 — TEMPLATE DESIGNER
# ─────────────────────────────────────────────────────────────────────────────
def tab_designer():
    left, right = st.columns([3, 2], gap="large")

    # ── Left: plate display + selection controls ─────────────────────────────
    with left:

        # Plate (display only — hover shows well info)
        fig = make_plate_figure(st.session_state.assignments,
                                st.session_state.selected_wells)
        st.plotly_chart(fig, use_container_width=True,
                        key=f"plate_{st.session_state.chart_ver}",
                        config={"displayModeBar": False})

        # ── Row selector buttons ──────────────────────────────────────────
        st.markdown("**Select by row** — click a row letter to toggle all 12 wells in that row")
        sel_set = set(st.session_state.selected_wells)
        row_btns = st.columns(8)
        for i, row in enumerate(ROWS):
            row_wells = [f"{row}{c}" for c in COLS]
            all_in = all(w in sel_set for w in row_wells)
            label = f"✓ {row}" if all_in else f"  {row}  "
            if row_btns[i].button(label, key=f"rowbtn_{row}", use_container_width=True):
                if all_in:
                    sel_set -= set(row_wells)
                else:
                    sel_set |= set(row_wells)
                st.session_state.selected_wells = sorted(sel_set, key=well_sort_key)
                st.rerun()

        # ── Column selector buttons ───────────────────────────────────────
        st.markdown("**Select by column** — click a column number to toggle all 8 wells in that column")
        col_btns = st.columns(12)
        for i, col in enumerate(COLS):
            col_wells = [f"{r}{col}" for r in ROWS]
            all_in = all(w in sel_set for w in col_wells)
            label = f"✓{col}" if all_in else str(col)
            if col_btns[i].button(label, key=f"colbtn_{col}", use_container_width=True):
                if all_in:
                    sel_set -= set(col_wells)
                else:
                    sel_set |= set(col_wells)
                st.session_state.selected_wells = sorted(sel_set, key=well_sort_key)
                st.rerun()

        # ── Text range input ──────────────────────────────────────────────
        st.markdown("**Or type well IDs / ranges**")
        st.caption("Single: `A1`  ·  List: `A1, B2, C3`  ·  Row range: `A1:A6`  ·  Block: `A1:C3`")

        ri1, ri2, ri3 = st.columns([5, 1, 1])
        range_txt = ri1.text_input("well_range", label_visibility="collapsed",
                                   placeholder="e.g.  A1, B1:B3, C1:D6",
                                   key="range_input")

        if ri2.button("Add", key="range_add", use_container_width=True, type="primary"):
            if range_txt.strip():
                parsed = parse_well_range(range_txt)
                if parsed:
                    sel_set = set(st.session_state.selected_wells) | set(parsed)
                    st.session_state.selected_wells = sorted(sel_set, key=well_sort_key)
                    st.rerun()
                else:
                    st.warning("No valid wells found. Check the format above.")

        if ri3.button("Remove", key="range_remove", use_container_width=True):
            if range_txt.strip():
                parsed = set(parse_well_range(range_txt))
                sel_set = set(st.session_state.selected_wells) - parsed
                st.session_state.selected_wells = sorted(sel_set, key=well_sort_key)
                st.rerun()

        # ── Selection status ──────────────────────────────────────────────
        if st.session_state.selected_wells:
            sel_sorted = sorted(st.session_state.selected_wells, key=well_sort_key)
            st.success(f"**{len(sel_sorted)}** well(s) selected:  "
                       f"{', '.join(sel_sorted)}")
        else:
            st.caption("No wells selected.  Use the row/column buttons or type a range above.")

    # ── Right: assignment controls + legend ──────────────────────────────────
    with right:
        st.subheader("Assign condition")

        used_colors = len(st.session_state.conditions_list)
        default_hex = DEFAULT_PALETTE[used_colors % len(DEFAULT_PALETTE)]

        cond_name  = st.text_input("Condition name", key="cond_name_input",
                                   placeholder="e.g.  E2 + CCCP + Glucose")
        cond_color = st.color_picker("Colour", value=default_hex, key="cond_color_input")

        can_assign = bool(st.session_state.selected_wells) and bool(cond_name.strip())

        c1, c2 = st.columns(2)
        assign_clicked = c1.button("Assign wells", type="primary", disabled=not can_assign)
        clear_clicked  = c2.button("Clear selection")

        if assign_clicked and can_assign:
            wells = sorted(st.session_state.selected_wells, key=well_sort_key)
            name  = cond_name.strip()
            existing = next((c for c in st.session_state.conditions_list
                             if c["name"] == name), None)
            if existing:
                color    = existing["color"]
                base_rep = len(existing["wells"]) + 1
                for rep, w in enumerate(wells, start=base_rep):
                    st.session_state.assignments[w] = {
                        "name": name, "color": color,
                        "replicate": rep, "start_well": existing["wells"][0],
                    }
                existing["wells"] += [w for w in wells if w not in existing["wells"]]
            else:
                for rep, w in enumerate(wells, start=1):
                    st.session_state.assignments[w] = {
                        "name": name, "color": cond_color,
                        "replicate": rep, "start_well": wells[0],
                    }
                st.session_state.conditions_list.append(
                    {"name": name, "color": cond_color, "wells": list(wells)}
                )
            st.session_state.undo_stack.append({"wells": wells, "name": name})
            st.session_state.selected_wells = []
            st.session_state.chart_ver += 1
            st.rerun()

        if clear_clicked:
            st.session_state.selected_wells = []
            st.session_state.chart_ver += 1
            st.rerun()

        # ── Legend ──────────────────────────────────────────────────────────
        st.divider()
        st.subheader("Legend")
        if st.session_state.conditions_list:
            for cond in st.session_state.conditions_list:
                a, b = st.columns([1, 10])
                a.markdown(
                    f'<div style="width:18px;height:18px;border-radius:50%;'
                    f'background:{cond["color"]};border:1px solid #888;margin-top:5px"></div>',
                    unsafe_allow_html=True,
                )
                b.markdown(f"**{cond['name']}** — {', '.join(cond['wells'])}")
        else:
            st.caption("No conditions assigned yet.")

        # ── Undo / Clear ─────────────────────────────────────────────────────
        st.divider()
        u1, u2 = st.columns(2)
        if u1.button("↩ Undo last assign",
                     disabled=not st.session_state.undo_stack):
            last = st.session_state.undo_stack.pop()
            removed = set(last["wells"])
            for w in removed:
                st.session_state.assignments.pop(w, None)
            st.session_state.conditions_list = [
                {**c, "wells": [w for w in c["wells"] if w not in removed]}
                for c in st.session_state.conditions_list
            ]
            st.session_state.conditions_list = [
                c for c in st.session_state.conditions_list if c["wells"]
            ]
            st.session_state.chart_ver += 1
            st.rerun()

        if u2.button("🗑 Clear all"):
            for k in ("assignments", "conditions_list", "selected_wells",
                      "undo_stack", "template_csv"):
                st.session_state[k] = {} if k == "assignments" else (
                    [] if k != "template_csv" else None)
            st.session_state.chart_ver += 1
            st.rerun()

        # ── Export ───────────────────────────────────────────────────────────
        st.divider()
        st.subheader("Export")
        if not st.session_state.assignments:
            st.caption("Assign at least one condition to export.")
        else:
            layout_title = st.text_input("Layout title", value="Plate Layout")

            csv_bytes = assignments_to_csv_bytes(st.session_state.assignments)
            st.session_state.template_csv = csv_bytes

            st.download_button("📥 plate_layout.csv", data=csv_bytes,
                               file_name="plate_layout.csv", mime="text/csv")

            png_bytes = draw_plate_png(st.session_state.assignments,
                                       st.session_state.conditions_list,
                                       title=layout_title)
            st.download_button("📥 plate_layout.png", data=png_bytes,
                               file_name="plate_layout.png", mime="image/png")


# ─────────────────────────────────────────────────────────────────────────────
# TAB 2 — DATA ANALYZER
# ─────────────────────────────────────────────────────────────────────────────
def tab_analyzer():
    # ── Step 1: file uploads ─────────────────────────────────────────────────
    st.subheader("Step 1 — Upload files")
    c1, c2 = st.columns(2)

    with c1:
        raw_file = st.file_uploader(
            "SpectraMax export  (.xls · .txt · .csv · .tsv)",
            type=["xls", "txt", "csv", "tsv"],
            key="raw_uploader",
        )

    with c2:
        use_designer = st.checkbox(
            "Use template from Designer tab",
            value=(st.session_state.template_csv is not None),
            disabled=(st.session_state.template_csv is None),
        )
        if use_designer and st.session_state.template_csv:
            template_bytes = st.session_state.template_csv
            st.success("Template loaded from Designer tab.")
        else:
            tpl_file = st.file_uploader("plate_layout.csv", type=["csv"],
                                        key="template_uploader")
            template_bytes = tpl_file.read() if tpl_file else None

    # ── Parse ────────────────────────────────────────────────────────────────
    raw_df = assay_type = assigned_df = cond_order = color_map = None

    if raw_file:
        try:
            assay_type, raw_df = parse_spectramax(raw_file.read())
            st.success(f"SpectraMax file parsed — **{assay_type}** assay · "
                       f"{raw_df['Well'].nunique()} wells · "
                       f"{len(raw_df):,} records")
        except Exception as e:
            st.error(f"Could not parse SpectraMax file: {e}")

    if template_bytes:
        try:
            assigned_df, cond_order, color_map = load_template_csv(template_bytes)
            with st.expander("Preview loaded plate layout"):
                asgn = template_csv_to_assignments(template_bytes)
                st.plotly_chart(make_plate_figure(asgn),
                                use_container_width=True, key="tpl_preview")
            st.success(f"Template loaded — **{len(cond_order)}** condition(s): "
                       f"{', '.join(cond_order)}")
        except Exception as e:
            st.error(f"Could not load template: {e}")

    if raw_df is None or assigned_df is None:
        st.info("Upload both files above to continue.")
        return

    # ── Step 2: Merge & configure ─────────────────────────────────────────────
    try:
        merged = merge_with_template(raw_df, assigned_df)
    except Exception as e:
        st.error(f"Merge error: {e}")
        return

    if merged.empty:
        st.error("No overlap between data file wells and template wells. "
                 "Check that you uploaded the matching files.")
        return

    st.divider()
    st.subheader("Step 2 — Configure analysis")

    col_a, col_b, col_c = st.columns(3)

    with col_a:
        if assay_type == "kinetic":
            assay_choice = st.radio("Assay type", ["kinetic", "endpoint"],
                                    horizontal=True)
        else:
            assay_choice = "endpoint"
            st.radio("Assay type", ["endpoint"], horizontal=True)

    with col_b:
        do_blank = st.checkbox("Apply blank subtraction")

    with col_c:
        y_label = st.text_input("Y-axis label", value="Absorbance (a.u.)")

    # ── Blank subtraction pairs ───────────────────────────────────────────────
    blank_pairs: list[dict] = []
    if do_blank:
        st.markdown("#### Blank subtraction pairs")
        st.caption("Pair each data condition with its corresponding blank condition. "
                   "The blank mean will be subtracted.")
        n_pairs = st.number_input("Number of pairs", min_value=1,
                                  max_value=len(cond_order), value=1, step=1)
        for i in range(int(n_pairs)):
            p1, _, p2 = st.columns([5, 1, 5])
            data_c  = p1.selectbox(f"Data condition {i+1}", cond_order,
                                   key=f"dc_{i}")
            blank_c = p2.selectbox(f"Blank condition {i+1}", cond_order,
                                   key=f"bc_{i}")
            blank_pairs.append({"Condition": data_c, "Blank_Condition": blank_c})

    # ── Kinetic time range ────────────────────────────────────────────────────
    time_filter_s = None
    if assay_choice == "kinetic":
        all_s = sorted(merged["Time_s"].dropna().astype(int).unique())
        mins_all = [round(s / 60, 2) for s in all_s]
        st.caption(f"Kinetic data: {len(all_s)} time points · "
                   f"{mins_all[0]}–{mins_all[-1]} min")
        t_range = st.slider("Plot time range (min)",
                            float(mins_all[0]), float(mins_all[-1]),
                            (float(mins_all[0]), float(mins_all[-1])))
        time_filter_s = (t_range[0] * 60, t_range[1] * 60)

    # ── Condition & title ──────────────────────────────────────────────────────
    sel_conds = st.multiselect("Conditions to plot", cond_order, default=cond_order)
    plot_title_default = (
        f"{'Endpoint' if assay_choice == 'endpoint' else 'Kinetic'} summary "
        f"({'blank-subtracted, ' if do_blank else ''}mean ± SD)"
    )
    plot_title = st.text_input("Plot title", value=plot_title_default)

    # ── Run ───────────────────────────────────────────────────────────────────
    st.divider()
    if not st.button("Run analysis", type="primary"):
        return
    if not sel_conds:
        st.warning("Select at least one condition.")
        return

    with st.spinner("Analysing…"):
        try:
            analysis_df = merged.copy()

            # Blank subtraction
            if do_blank and blank_pairs:
                analysis_df, pair_df = apply_blank_subtraction(
                    analysis_df, blank_pairs, assay_choice)
            else:
                pair_df = pd.DataFrame()

            # Filter conditions
            analysis_df = analysis_df[
                analysis_df["Condition"].astype(str).isin(sel_conds)
            ].copy()

            # Time filter (kinetic)
            if assay_choice == "kinetic" and time_filter_s:
                t0, t1 = time_filter_s
                analysis_df = analysis_df[
                    (analysis_df["Time_s"] >= t0) &
                    (analysis_df["Time_s"] <= t1)
                ].copy()

            sel_order = [c for c in cond_order if c in sel_conds]
            active_colors = {c: color_map.get(c, "#808080") for c in sel_order}

            if assay_choice == "endpoint":
                summary = endpoint_summary(analysis_df, sel_order)
                png = plot_endpoint(summary, active_colors, plot_title, y_label)

                st.subheader("Results")
                st.image(png, use_container_width=True)

                d1, d2, d3, d4 = st.columns(4)
                d1.download_button("📥 Plot PNG", png,
                                   "endpoint_plot.png", "image/png")
                d2.download_button("📥 Summary CSV",
                                   summary.to_csv(index=False).encode(),
                                   "endpoint_summary.csv", "text/csv")
                d3.download_button("📥 Wells CSV",
                                   analysis_df.to_csv(index=False).encode(),
                                   "endpoint_wells.csv", "text/csv")
                if not pair_df.empty:
                    d4.download_button("📥 Blank pairs CSV",
                                       pair_df.to_csv(index=False).encode(),
                                       "blank_pairs.csv", "text/csv")
                st.dataframe(
                    summary.style.format({"Mean": "{:.4f}", "SD": "{:.4f}"}),
                    use_container_width=True,
                )

            else:  # kinetic
                summary = kinetic_summary(analysis_df, sel_order)
                png = plot_kinetic(summary, active_colors, plot_title, y_label)

                st.subheader("Results")
                st.image(png, use_container_width=True)

                d1, d2, d3, d4 = st.columns(4)
                d1.download_button("📥 Plot PNG", png,
                                   "kinetic_plot.png", "image/png")
                d2.download_button("📥 Summary CSV",
                                   summary.to_csv(index=False).encode(),
                                   "kinetic_summary.csv", "text/csv")
                d3.download_button("📥 Wells CSV",
                                   analysis_df.to_csv(index=False).encode(),
                                   "kinetic_wells.csv", "text/csv")
                if not pair_df.empty:
                    d4.download_button("📥 Blank pairs CSV",
                                       pair_df.to_csv(index=False).encode(),
                                       "blank_pairs.csv", "text/csv")

                pivot = summary.pivot_table(
                    index="Time_min", columns="Condition", values="Mean"
                ).reset_index()
                st.dataframe(
                    pivot.style.format({c: "{:.4f}" for c in pivot.columns
                                        if c != "Time_min"}),
                    use_container_width=True,
                )

        except Exception as e:
            st.error(f"Analysis error: {e}")


# ─────────────────────────────────────────────────────────────────────────────
# TAB 3 — HELP
# ─────────────────────────────────────────────────────────────────────────────
def tab_help():
    st.markdown("""
### Template Designer

**How to select wells:**

- **Row buttons** (A–H) — click any letter to select all 12 wells in that row.  Click again to deselect the whole row.
- **Column buttons** (1–12) — click any number to toggle all 8 wells in that column.
- **Range text input** — type a range and click **Add** to add those wells to your selection, or **Remove** to deselect them.  Supported formats:
  - Single well: `A1`
  - Comma list: `A1, B2, C3`
  - Row range: `A1:A6` → selects A1 A2 A3 A4 A5 A6
  - Column range: `A1:D1` → selects A1 B1 C1 D1
  - Rectangular block: `A1:C3` → selects a 3 × 3 block
- Mix methods freely — selections **accumulate** across all three methods.

**Assigning a condition:**

1. Select any combination of wells using the controls above.
2. Type a condition name and pick a colour on the right.
3. Click **Assign wells** — wells turn to that colour on the plate and appear in the legend.
4. Re-use the same condition name to **add more wells** to an existing condition.
5. **Undo last assign** reverses the most recent assignment block.
6. When finished, download **plate_layout.csv** and **plate_layout.png**.

---

### Data Analyzer

1. Upload the **SpectraMax .xls** export from your reader.
2. Either tick *"Use template from Designer tab"* or upload your saved **plate_layout.csv**.
3. Choose **endpoint** or **kinetic** — the app auto-detects from the file.
4. Optionally enable **blank subtraction** and pair each data condition to its blank.
5. Choose conditions to include, set your axis label and title.
6. Click **Run analysis** to generate plots and download all results.

---

### SpectraMax file format

Export from SpectraMax Pro using **File → Export → Text** with default settings.  
The resulting `.xls` file is UTF-16 tab-delimited text — the app reads it correctly.

---

### Running locally on Windows

```
pip install streamlit pandas matplotlib plotly numpy openpyxl
streamlit run app.py
```

Or double-click **launch.bat** — it opens the app in your browser automatically.

---

### Deploying on Streamlit Community Cloud

1. Push this folder to a GitHub repository.
2. Go to **share.streamlit.io** → *New app* → select your repo and `app.py`.
3. Click **Deploy** — done. Share the URL with your whole lab.
    """)


# ─────────────────────────────────────────────────────────────────────────────
# ENTRY POINT
# ─────────────────────────────────────────────────────────────────────────────
def main():
    st.set_page_config(
        page_title="96-Well Plate Analyzer",
        page_icon="🧫",
        layout="wide",
        initial_sidebar_state="collapsed",
    )
    _init()

    st.title("🧫  96-Well Plate Analyzer")
    st.caption("Design your plate layout · analyze your SpectraMax data · download publication-ready plots")

    t1, t2, t3 = st.tabs(["🎨  Template Designer", "📊  Data Analyzer", "ℹ️  Help"])
    with t1:
        tab_designer()
    with t2:
        tab_analyzer()
    with t3:
        tab_help()


if __name__ == "__main__":
    main()
