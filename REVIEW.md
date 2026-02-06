# Code Review: backlog_db.py

Review of the SQLite Product Backlog module. Two review rounds completed; all findings resolved.

**Overall**: Well-structured module with solid fundamentals — good use of WAL mode, `BEGIN IMMEDIATE` for write serialization, parameterized queries throughout, and a clean API design. 144 tests passing. All findings from both review rounds have been addressed.

---

## Review Round 2 — Findings (all resolved)

### Bugs

| # | Finding | Fix |
|---|---|---|
| R2-1 | `close()` left stale connections in worker threads' `threading.local` — caused `ProgrammingError` | `_conn` now checks if connection is in `_all_conns` set; stale refs get replaced with fresh connections |
| R2-2 | `get_backlog_db()` race with `close()` could cause `KeyError` | Lock now held for the full get-or-create-and-return operation |
| R2-3 | `update_priority()` accepted floats, strings, booleans — no validation | Added `isinstance` check matching `add()`; also rejects `bool` in both methods |

### Recommendations

| # | Finding | Fix |
|---|---|---|
| R2-4 | `delete()` silently unparented children without audit trail or timestamp update | Now records `parent_change` event for each child and updates `updated_at` |
| R2-5 | `list_items(top_level_only=True, parent=5)` silently ignored `parent` | Now raises `ValueError` when both are specified |
| R2-6 | `backlog_events.item_id` has no FK constraint (intentional for deleted items) | Documented; no code change needed |

### Suggestions

| # | Finding | Fix |
|---|---|---|
| R2-7 | `_now()` and SQLite `DEFAULT` use different clock sources for timestamps | `add()` now explicitly passes `_now()` for both `created_at` and `updated_at` — single clock source |
| R2-8 | Concurrent status race test is theoretically fragile on overloaded CI | Accepted risk — skipped per user request |
| R2-9 | No test for `get_history()` on a deleted item via public API | Added `test_get_history_on_deleted_item` |
| R2-10 | No type validation on `item_id` parameters | Low priority — SQLite binding produces clear errors; not changed |
| R2-11 | No test for concurrent `delete()` of the same item | Added `test_concurrent_delete_same_item` |

---

## Review Round 1 — Findings (all resolved)

### HIGH

| # | Finding | Fix |
|---|---|---|
| 1 | `BacklogItem.comment()` timestamp drift — second `_now()` call produced a different value | `comment()` now returns `BacklogItem`; bound method reads `updated.updated_at` from it |
| 2 | `get_backlog_db()` silently ignored `agent=` on second call for same path | Singleton key now includes agent: `f"{resolved_path}::{agent or ''}"` |
| 3 | Self-referencing and circular parent hierarchies allowed | `_validate_parent()` walks the chain; rejects self-reference and cycles with `ValueError` |

### MEDIUM

| # | Finding | Fix |
|---|---|---|
| 4 | `close()` only closed calling thread's connection, leaked others | `_all_conns` set tracks all connections; `close()` closes them all |
| 5 | No input validation on title, priority, comment, result | Validates: empty title, non-int priority, empty comment, non-string result |
| 6 | `_init_schema` used `executescript` (implicit commit) | Replaced with individual `execute()` calls + explicit `commit()` |
| 7a | `add()`/`update_parent()` raised raw `IntegrityError` for missing parent | `_validate_parent()` checks existence, raises `LookupError` |
| 7b | `parent=None` in `list_items` had confusing semantics | Added `top_level_only=True` as clearer alternative; `parent=None` kept for backward compat |

### LOW

| # | Finding | Fix |
|---|---|---|
| 8 | `get_history()` returned `[]` for non-existent items | Raises `LookupError` if item never existed (but still returns events for deleted items) |
| 9 | `timeout=30` on `connect()` conflicted with `busy_timeout=5000` PRAGMA | Removed misleading `timeout=30` parameter |
| 10 | No `delete`, `update_title`, or `update_description` methods | Added all three with bound methods and audit events |
| 11 | Missing test coverage for edge cases | 144 tests total covering all gaps |
| 12 | `close()` docs didn't mention thread-local limitation | Fixed by fixing #4 — `close()` now closes all threads' connections |

---

## Positive Notes

- Parameterized queries throughout — no SQL injection risk
- `_UNSET` sentinel pattern cleanly distinguishes "not passed" from `None`
- `BEGIN IMMEDIATE` transactions prevent deadlocks between concurrent writers
- Comprehensive audit trail with agent identity on every mutation
- `BacklogItem` with optional backlog binding is a clean pattern
- 144 tests with good coverage including concurrency, validation, and edge cases
- `delete()` preserves full audit trail with JSON snapshot of final state
- `delete()` records `parent_change` events for unparented children
- Cycle detection walks the parent chain to prevent circular hierarchies
- Stale connection detection prevents `ProgrammingError` after `close()`
- Single clock source (`_now()`) for all timestamps
- API guide is thorough and accurate
