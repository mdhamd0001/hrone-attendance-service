# REVIEW.md

List every defect found in the starter `app/main.py` (helpers and endpoints).

| # | Where (function / line) | What is wrong | How you would notice it (test, input, or symptom) | How you fixed it |
|---|---|---|---|---|
| 1 | `health()` | Always returned 200 even when MongoDB is down. Spec says 503 when MongoDB does not answer. | Stop MongoDB, call GET /health — still gets 200. | Added `client.admin.command("ping")` in try/except; raise 503 on failure. |
| 2 | `create_employee()` | Race condition: used `find_one` then `insert_one` (check-then-act). Two concurrent identical creates could both pass the check. | Send two simultaneous POST /employees with the same `emp_code` under load — both get 201. | Removed the pre-check. Rely on the unique index on `emp_code` and catch `DuplicateKeyError`. |
| 3 | `create_employee()` | `datetime.now()` stores naive local-time datetime. On a non-UTC server the stored value and the epoch-ms returned in the response are both wrong. | Compare returned `created_at` ms with real UTC wall clock on a UTC+5:30 server. | Changed to `datetime.now(tz=timezone.utc)`. |
| 4 | `list_employees()` | `skip = page * page_size` is off by one page. Page 1 skips the first 20 docs entirely. | GET /employees?page=1 returns items 21-40 instead of 1-20. | Changed to `(page - 1) * page_size`. |
| 5 | `list_employees()` | `total = db.employees.count_documents({})` ignores the `department` filter. Total is always the whole collection size. | GET /employees?department=Engineering — total shows everyone, not just that department. | Changed to `db.employees.count_documents(q)`. |
| 6 | `list_employees()` | Results not sorted. Spec requires `emp_code` ascending. | GET /employees — order is insertion order. | Added `.sort("emp_code", ASCENDING)` to the cursor. |
| 7 | `punch_in()` | No 404 for unknown employee. `emp` is None so `emp["shift_start"]` raises `TypeError` (500). | POST /attendance/punch-in with unknown `emp_code`. | Added `if not emp: raise HTTPException(404, ...)`. |
| 8 | `punch_in()` | `datetime.fromtimestamp(ms/1000)` uses server local time instead of UTC. | On UTC+5:30 server, a UTC punch-in time is stored 5h30m ahead of reality. | Changed to `datetime.fromtimestamp(ms/1000, tz=timezone.utc)`. |
| 9 | `punch_in()` | Same check-then-insert race as #2. Two simultaneous punch-ins for same employee/date could both succeed. | Concurrent punch-ins under load — both get 201, two documents in the collection. | Removed pre-check. Rely on unique compound index `(emp_code, date)` + catch `DuplicateKeyError`. |
| 10 | `punch_in()` | `ts.date().isoformat()` uses server local time, not IST (R1). Overnight shifts not handled at all. | Punch at 23:45 UTC is 05:15 IST next day — wrong attendance date on any server not in IST. | Introduced `attendance_date()` that converts to IST and handles overnight shifts. |
| 11 | `compute_late_minutes()` | Built shift_start with `punch_in.replace(hour=h, minute=m)` which copies UTC tz onto an IST time. Comparison is UTC punch_in minus "UTC-branded" 09:30 IST — off by 5h30m. | Shift 09:30 IST: punch at 09:35 IST (04:05 UTC) shows diff ≈ -5h25m, so late_minutes = 0 always. | Rewrote helper to build `start_ist` as IST-aware datetime on the attendance date. |
| 12 | `list_attendance()` | Fetched ALL docs into Python then sorted + paginated in memory — O(N) on 100k docs, loads entire collection. | Load 100k docs, call GET /attendance — response time is proportional to collection size. | Moved sort + pagination to the database: `.sort(...).skip(skip).limit(page_size)`. |
| 13 | `list_attendance()` | `total = len(docs)` counts the already-paginated slice, always ≤ page_size and ignores filters. | GET /attendance?emp_code=EMP0001 — total shows 20, not the real count for that employee. | Changed to `db.attendance_logs.count_documents(q)` before pagination. |
| 14 | `list_attendance()` | `date_from > date_to` silently returns empty results instead of 422 (spec requirement). | GET /attendance?date_from=2026-07-10&date_to=2026-07-01 — 200 with empty items. | Added explicit check and `HTTPException(422, ...)`. |

### Things examined and decided not defects
- `$gte`/`$lte` string comparison on date fields is correct because `YYYY-MM-DD` strings sort lexicographically the same as calendar dates.
- `load_dotenv()` at the top is correct; real environment variables already win over `.env` values.
- `body.model_dump()` (Pydantic v2) is correct; `dict()` would be the v1 equivalent.
