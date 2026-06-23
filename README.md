# Titagarh-Daily-MIS-Report-Automation
Automated daily generation of the **Titagarh (TRSL) MIS Report** for two customers — **TRSL (679)** and **TRSL HED (682)** — as print-ready PDFs.

It replaces the old manual workflow (run SQL → run 3 notebooks → paste into Excel →
generate HTML via Gemini → export PDF) with a **single command** that collects all
data, renders the report, and saves the PDF automatically.

---

## What it does

A single parameterized script (`generate_titagarh_report.py`):

1. Pulls data from every source (Postgres + 3 MongoDB collections)
2. Computes the date windows automatically (**Last Day / Week / MTD**)
3. Renders a self-contained HTML report (exact report layout, logos embedded)
4. Exports it to **PDF via headless Chrome**
5. Auto-increments the **Report No. (RRnn)** per customer
6. Saves both `.html` and `.pdf` into the project folder

Runs **daily at 12:00 PM** via Windows Task Scheduler for both customers.

---

## Data sources

| Section | Source | Detail |
|---|---|---|
| Melting + Quality & Testing | **PostgreSQL** (`ebdb`) | filtered by `customer_id` |
| Inspection stages | **MongoDB** `dlms_customer_<id>` | Inward, Mould, Shot Blasting AHT, Final, Load test (+ Accepted/Rejected) |
| Pouring (AI) + Safety Shield | **MongoDB** `agniss_db` | collections `<id>_POURING`, `<id>_SS` |
| WIP & Production (Section 8) | **MongoDB** `dlms_customer_<id>` | `dashboard_traces` |

---

## Key features

- **No manual date edits** — report is dated *yesterday*; windows are computed from the run day.
- **Multi-customer** — one script serves both customers via `--customer`:
  - `679` → title **TRSL – Daily MIS Report**
  - `682` → title **TRSL (HED) – Daily MIS Report**
- **Auto report numbering** — `RRnn` increments per customer (stored in state files).
- **Manual-only sections** (Furnace Utilization, Fresh/Reclaimed Sand, Rejection reasons) render as placeholders (`0` / `-`).
- **Backfill** support for any past date.
- **Logging** — every run appends to `report_log.txt`.

---

## Requirements

- **Python 3.9+**
- Python packages:
  ```bash
  pip install psycopg2 pymongo
  ```
- **Google Chrome** (or Microsoft Edge) installed — used to render the PDF.

---

## Usage

### Generate today's reports (dated yesterday)

```bash
python generate_titagarh_report.py --customer 679    # TRSL
python generate_titagarh_report.py --customer 682    # TRSL HED
```

### Run from a Jupyter notebook cell

```python
import sys
sys.argv = ["report", "--customer", "682"]                          # HED, yesterday
# sys.argv = ["report", "--customer", "679"]                         # TRSL, yesterday
# sys.argv = ["report", "--customer", "682", "--date", "21.06.2026"] # backfill a date
exec(open(r"D:\Titagarh Mis project\generate_titagarh_report.py", encoding="utf-8").read())
```

### Backfill a specific date

```bash
python generate_titagarh_report.py --customer 679 --date 21.06.2026
```
> `--date` is the **report date** (the "yesterday" being reported on), format `dd.mm.yyyy`.

### Output

Files are written to the project folder:

```
Titagarh Mis report DD.MM.YYYY.pdf        # 679 (TRSL)
Titagarh_HED Mis report DD.MM.YYYY.pdf    # 682 (TRSL HED)
```
(plus matching `.html` files)

---

## Scheduling (Windows Task Scheduler)

A daily task named **"Titagarh Daily MIS Report"** runs `run_report.bat` at **12:00 PM**,
which generates both customers' reports.

Recreate the task if needed (PowerShell):

```powershell
$action  = New-ScheduledTaskAction -Execute "D:\Titagarh Mis project\run_report.bat"
$trigger = New-ScheduledTaskTrigger -Daily -At 12:00PM
$set     = New-ScheduledTaskSettingsSet -StartWhenAvailable -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName "Titagarh Daily MIS Report" -Action $action -Trigger $trigger -Settings $set -Force
```

> The PC must be on at noon. If asleep/off, the task runs as soon as the machine is next available.

---

## Project files

| File | Purpose |
|---|---|
| `generate_titagarh_report.py` | Main report generator |
| `run_report.bat` | Scheduler launcher (runs 679 then 682) |
| `report_state.json` | RR counter for 679 |
| `report_state_682.json` | RR counter for 682 |
| `report_log.txt` | Run log |

---

## Configuration

Per-customer settings live in the `CUSTOMERS` dictionary near the top of
`generate_titagarh_report.py` (title, customer id, database names, file prefix,
RR seed). To onboard another customer, add a new entry there.

---

## ⚠️ Security note (read before pushing to GitHub)

The script currently contains **hardcoded database credentials** (PostgreSQL password
and MongoDB connection URIs with passwords). **Do not commit secrets to a public
repository.** Before pushing, move credentials to environment variables or a local
config file that is git-ignored, and **rotate any credentials that were already
committed**.

A starter `.gitignore`:

```gitignore
# Secrets / local config
.env
config.local.py

# Generated output
*.pdf
*.html
report_log.txt
report_state*.json

# Python
__pycache__/
*.pyc
```
