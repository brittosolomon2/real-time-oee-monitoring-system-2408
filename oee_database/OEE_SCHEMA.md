# OEE Database Schema (PostgreSQL) — Step 02.00

This database container uses PostgreSQL on port `5000` with connection command stored in:

- `oee_database/db_connection.txt` → `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

Per container conventions, schema and seed were applied by executing SQL **one statement at a time** via:

- `psql ... -c "SQL_STATEMENT"`

No `.sql` migration files were created.

---

## Objects created

Extension:
- `pgcrypto` (for `gen_random_uuid()`)

Tables (public schema):
- `lines` — manufacturing lines (master data)
- `shifts` — shift definitions (name + start/end time + timezone)
- `runs` — production runs per line (optionally linked to a shift)
- `events` — unified event log for `production | downtime | quality`
- `oee_snapshots` — computed OEE KPI snapshots (time series)
- `alert_thresholds` — per-line thresholds for metrics
- `alerts` — persisted alert instances (open/ack/closed)
- `handover_reports` — structured shift handover reports

Indexes:
- `idx_runs_line_started_at`
- `idx_runs_shift_started_at`
- `idx_events_run_started_at`
- `idx_events_line_started_at`
- `idx_events_type_started_at`
- `idx_oee_snapshots_line_snapshot_at`
- `idx_alerts_line_triggered_at`
- `idx_handover_reports_line_date`

---

## Minimal seed data inserted

- 2 lines: `LINE-1`, `LINE-2`
- 3 shifts: `Day`, `Swing`, `Night`
- 1 active run for `LINE-1`
- 3 events for the active run:
  - production
  - downtime
  - quality
- 2 alert thresholds for `LINE-1`:
  - `oee < 0.65` (warning)
  - `downtime_rate > 0.15` (critical)
- 1 OEE snapshot for `LINE-1`
- 1 handover report for `LINE-1`

---

## Re-applying / verifying (manual)

### Connect
```bash
# from real-time-oee-monitoring-system-2408/oee_database
$(cat db_connection.txt)
```

### Verify tables exist
```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "\dt"
```

### Verify seed counts
```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "SELECT 'lines' as table, count(*) FROM lines
UNION ALL SELECT 'shifts', count(*) FROM shifts
UNION ALL SELECT 'runs', count(*) FROM runs
UNION ALL SELECT 'events', count(*) FROM events
UNION ALL SELECT 'oee_snapshots', count(*) FROM oee_snapshots
UNION ALL SELECT 'alert_thresholds', count(*) FROM alert_thresholds
UNION ALL SELECT 'handover_reports', count(*) FROM handover_reports;"
```

---

## Notes for backend integration (Step 03.00)

- UUID PKs everywhere (`gen_random_uuid()`).
- `events.event_type` is constrained to: `production | downtime | quality`.
- `events.duration_seconds` is a stored generated column derived from `started_at/ended_at`.
- `oee_snapshots` is unique on `(line_id, snapshot_at, window_seconds)` to prevent duplicates for same point/window.
- `alert_thresholds` unique on `(line_id, metric)` to keep one threshold per line/metric.
- `handover_reports` unique on `(line_id, report_date, shift_id)`.

Task completed: PostgreSQL OEE domain schema and minimal seed data are in place; documentation added in `OEE_SCHEMA.md`.
