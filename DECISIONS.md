# DECISIONS.md

1. **Indexes.**
   I created five indexes. A unique index on `employees.emp_code` enforces the business key and makes all employee lookups O(1). A unique compound index on `attendance_logs.(emp_code, date)` enforces the one-record-per-day constraint and is the race-safe replacement for the check-then-insert pattern. A non-unique index on `(date, emp_code)` covers the attendance list query (filter by date range, sort by date + emp_code). A compound index on `(emp_code, date, status)` covers per-employee queries and the monthly aggregation match stage. An index on `(date, status, late_minutes)` covers the leaderboard match. I considered a single-field index on `joined_on` for the department summary headcount lookup but rejected it because the $lookup pipeline filters by both `department` and `joined_on` and the collection is small relative to logs.

2. **Punch-in race.**
   Both requests compute the attendance date and build the document concurrently. Both call `insert_one`. MongoDB's unique index on `(emp_code, date)` makes exactly one succeed; the other raises `DuplicateKeyError` at the storage layer with no gap. The winning request gets 201; the loser catches the error and returns 409. There is no window where a second document can slip through.

3. **Ties at the leaderboard cutoff.**
   I use `$setWindowFields` with `$rank`, which assigns the same rank to employees with equal `total_late_minutes` and skips the next ranks (1, 2, 2, 4). The `$match {rank: {$lte: limit}}` stage runs after ranking, so if two employees are tied at rank `limit`, both are included. The response can contain more rows than `limit`.

4. **Headcount with zero-log employees.**
   The department summary pipeline starts from the `employees` collection (`$match` on `joined_on <= last_day_of_month`), then does a `$lookup` to `attendance_logs` with `preserveNullAndEmptyArrays: true` and `$unwind`. Employees with no logs produce one row where all log fields are null. The outer `$group` counts each distinct `emp_code` once for headcount, and the null-log rows contribute 0 to all sums.

5. **At 100x the data.**
   The main bottleneck would be the `$densify` + per-day `$lookup` in the trend endpoint, which is O(days × employees) in the database. I would pre-aggregate a daily stats collection keyed by `(department, date)` using a change-stream worker or a scheduled job, and the trend endpoint would read from that collection instead of computing on the fly.
