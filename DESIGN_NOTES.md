# DESIGN_NOTES.md (template)

## Part 2 — critical fix choice
I fixed **SQL injection in task search** (`projects/views.py`) as the #1 issue because it is the highest business impact: any authenticated user can exfiltrate the entire database, including every user's email and password hash, through the `?q=` parameter. I replaced the raw f-string SQL with the Django ORM (`Q(title__icontains=q) | Q(description__icontains=q)`), which is injection-proof by construction and returns the same serialized shape as the non-search path.

## Part 3a — Comments: append-only design
Comments expose only `GET` (list) and `POST` (create). There are intentionally **no** update or delete routes, so append-only is enforced at the routing layer rather than relying on per-request checks. Read access requires any project membership; posting requires `admin`/`member` (viewers are 403), reusing the existing `_get_membership` / `_can_edit_tasks` helpers.

## Part 3b — Activity feed: transaction decision (required reasoning)
**Decision: best-effort, non-blocking.** If writing the activity/audit record fails, the original change (e.g. status update) still commits.

**Reasoning (2–3 sentences):** The primary user action is the source of truth and must not fail because a secondary audit write failed — rolling back a legitimate task update because logging hiccuped would be worse for the user and the data than a missing audit row. The activity feed is advisory/observability, not a correctness invariant, so I wrap the activity write in a try/except (or `transaction.on_commit`) that logs the failure without aborting the main operation. The trade-off is that the audit log can have gaps under failure; that is acceptable for a feed, and if strict completeness were required I would instead make it a hard dependency inside one atomic transaction.

## Part 3c — Airtable export
- **Idempotency:** keyed on `TaskId` (the task UUID). The client reads existing records once, indexes by `TaskId`, then updates matches and creates the rest — re-running produces no duplicates.
- **Error handling:** transient failures (429, 5xx, network/timeout) retry with bounded exponential backoff; permanent failures (4xx except 429) are not retried. Each record is processed independently so one failing record does not abort the batch; failures are returned in an `errors` array with counts.
- **Production vs tests:** production uses the real `pyairtable` `Api`; unit tests use `airtable_mock.py` so they never touch the network.
