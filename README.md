# Mini-P7 — CPM Scheduler -Archive 

NOTE:This project is no longer in active maintaence and hence archived

A Critical Path Method (CPM) scheduler with **two interfaces over one engine**:
a scriptable command line (Typer + Rich) and a PySide6 desktop GUI.CLI included in stak

---

## Architecture

Both interfaces are thin wrappers over a single headless core (`service.py`),
so they can never drift apart in behaviour:

```
            +-------------+     +--------------+
            |   cli.py    |     | main_window  |   (Typer/Rich)   (PySide6 GUI)
            +------+------+     +------+-------+
                   +--------+----------+
                     +------v-------+
                     |  service.py  |   ProjectService - no Qt, fully testable
                     +------+-------+
          +------------+----+-----+--------------+
     scheduler.py    db.py   export_manager   p6_xml.py
      (CPM engine)  (SQLite)   (xlsx/pdf)      (P6 XML)
```

---

## Install

```bash
python -m venv .venv
# Windows (cmd):        .venv\Scripts\activate.bat
# Windows (PowerShell): .venv\Scripts\Activate.ps1
# macOS / Linux:        source .venv/bin/activate

pip install -e ".[dev]"     # installs the app + dev tools, and the `minip7` command
```

Requires Python 3.11+.

> **No-install option.** You don't need the editable install to run the app —
> it only creates the `minip7` shortcut. You can always run commands directly:
> ```
> pip install PySide6 SQLAlchemy openpyxl reportlab typer rich pytest
> python cli.py sample
> python cli.py schedule
> python main.py
> ```
> `python cli.py <command>` is identical to `minip7 <command>`.

---

## Command line

```bash
minip7 --help                              # list all commands
minip7 sample                              # load the demo project
minip7 schedule                            # run CPM, critical path in red
minip7 gantt                               # ASCII Gantt in the terminal
minip7 critical-path                       # just the path + total duration

minip7 add E2 "Inspection" 3 -p C,D -r QA  # id, name, duration, -p preds, -r resource
minip7 edit D --duration 5
minip7 delete D
minip7 list
minip7 clear -y

minip7 settings -n "Tower" -s 2026-01-05 -c Mon-Fri   # name / start date / calendar
minip7 settings                            # show current settings

minip7 export schedule.xlsx                # export: xlsx | pdf | xml | csv | json
minip7 import plan.csv                      # import: xml | csv | json

minip7 --project other.db list             # operate on any database file
```

Every command exits non-zero on error, so it composes cleanly in scripts and CI.
Run `minip7 <command> --help` for the options of any single command.

---

## Desktop GUI

```bash
minip7-gui      # or:  python main.py
```

---

## How to use (GUI)

1. **Load Sample** to load the demo dataset
2. **Schedule** (F5) to run the CPM engine
3. The table fills with ES / EF / LS / LF / TF / FF values
4. The **Gantt Chart** tab renders - red bars = critical path
5. **Resource Loading** tab shows per-resource workload
6. **File -> Project Settings** (Ctrl+,) sets name, start date, and calendar;
   these now persist across restarts

### Keyboard shortcuts

| Action | Shortcut |
|---|---|
| New Project | `Ctrl+N` |
| Project Settings | `Ctrl+,` |
| Import P6 XML | `Ctrl+I` |
| Add Activity | `Ctrl+A` |
| Schedule (CPM) | `F5` |
| Export to Excel | `Ctrl+E` |
| Export to PDF | `Ctrl+P` |

---

## Testing

```bash
pytest                 # run all 20 tests
pytest -v              # one line per test
pytest -k float        # tests matching "float"
```

Linting / typing (dev extras):

```bash
ruff check .
ruff format .
mypy service.py scheduler.py
```

---

## Project structure

| File | Role | Status |
|---|---|---|
| `service.py` | Headless `ProjectService` - shared core | **new** |
| `cli.py` | Typer + Rich command line | **new** |
| `pyproject.toml` | Packaging, entry points, tool config | **new** |
| `tests/test_scheduler.py` | CPM engine tests | **new** |
| `tests/test_service.py` | Service / persistence / calendar tests | **new** |
| `models.py` | ORM models + `ProjectMeta` table | changed |
| `db.py` | `Database` class + compatibility shim | changed |
| `main_window.py` | GUI, now backed by `ProjectService` | changed |
| `requirements.txt` | dropped `networkx`, added `typer`/`rich` | changed |
| `activity.py` | Activity dataclass (CPM fields) | unchanged |
| `scheduler.py` | CPM engine (forward/backward pass + floats) | unchanged |
| `export_manager.py` | Excel + PDF export | unchanged |
| `p6_xml.py` | Primavera P6 XML import/export | unchanged |
| `activity_dialog.py` | Add/Edit activity dialog | unchanged |
| `activity_table.py` | Activity grid | unchanged |
| `gantt_view.py` | Gantt chart (QGraphicsView) | unchanged |
| `resource_panel.py` | Resource loading chart | unchanged |
| `status_panel.py` | Bottom status bar | unchanged |
| `project_settings_dialog.py` | Project name + start date dialog | unchanged |
| `main.py` | GUI entry point | unchanged |

---

## Sample data

| ID | Name       | Dur | Predecessor | Resource |
|----|------------|-----|-------------|----------|
| A  | Start      | 2   | -           | -        |
| B  | Foundation | 4   | A           | -        |
| C  | Structure  | 6   | B           | -        |
| D  | Electrical | 3   | B           | Electrical Team |
| E  | Finish     | 2   | C, D        | -        |

**Critical path:** A -> B -> C -> E  (14 days)
**D** has TF = 3, FF = 3   *(the engine value; an earlier README listed 5/5 in error)*

---

## CPM terminology

| Term | Formula | Meaning |
|---|---|---|
| ES | max(pred EF) | Earliest the activity can start |
| EF | ES + Duration | Earliest it can finish |
| LS | LF - Duration | Latest it can start without delay |
| LF | min(succ LS) | Latest it must finish |
| TF | LS - ES | Total Float - slack before delaying project |
| FF | min(succ ES) - EF | Free Float - slack before delaying any successor |
| Critical | TF == 0 | On the critical path |

---

## What changed in this revision

- **New CLI** (`cli.py`) over a **new headless core** (`service.py`) - the GUI
  now uses the same core, so the two interfaces stay in lock-step.
- **Persistence fix:** project name / start date / calendar now survive restarts
  (new `ProjectMeta` table).
- **Working-day calendar** wired in (Mon-Fri skips weekends).
- **CSV / JSON** import & export added alongside the existing xlsx / pdf / P6 XML.
- **20-test pytest suite** added, locking in correct float values.
- README float values corrected; unused `networkx` dependency removed.

---

## Phase 3 ideas

- SS / FF / SF relationship types with lag
- Baseline vs. actual comparison
- WBS tree panel (hierarchical grouping)
- Resource levelling / smoothing
- Professional GUI restyle (centralized theme / Fluent widgets)
- P6 XER format import
