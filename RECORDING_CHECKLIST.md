# RECORDING_CHECKLIST.md

> Record the **full session**, terminal visible throughout, narrate your thinking. Put the link in `README.md` or `RECORDING.md`. **No recording = not evaluated.**

## Checklist (in order)
```
[ ] Show the repo + `git status` (clean clone) and `git log --oneline`
[ ] Show setup: docker-compose up, migrate, seed
[ ] Open http://localhost:3000, log in as meera@taskboard.dev
[ ] Open Q3 Launch board (7 tasks)
[ ] Run baseline tests (pytest 15 passed, npm test)
[ ] Walk through REVIEW.md — the 4 issues
[ ] DEMONSTRATE the SQL injection bug with curl (500 + leaked users)
[ ] Show the responsible code (projects/views.py, the f-string SQL)
[ ] Implement the fix (ORM) live, or show the diff
[ ] Run the regression test (fails before / passes after)
[ ] Re-run the same curl → 200 + no leak
[ ] Commit the fix (show the message)
[ ] Demonstrate Part 3a comments: member posts (201), viewer 403, chronological list
[ ] Show Airtable base config (fields) and .env (blur the token)
[ ] Trigger export → show created: 7
[ ] Open Airtable → show 7 rows
[ ] Run export AGAIN → updated: 7, created: 0
[ ] Refresh Airtable → still 7 rows (idempotency)
[ ] (Optional) show partial-failure / retry via the unit test
[ ] Run the FULL test suite → all green
[ ] Show git history (not squashed) and the docs
```

## Suggested narration (per key step)
**SQL injection demo**
- WHAT I'M SHOWING: the `?q=` search endpoint and a crafted payload.
- WHAT I SAY: "The search builds raw SQL by string interpolation. A lone quote 500s, and this UNION payload returns the users table — emails and password hashes — proving arbitrary read. This is my #1 issue."
- VIEWER SEES: terminal 500, then rows where titles are emails and descriptions are `pbkdf2_sha256$...` hashes.

**The fix**
- WHAT I'M SHOWING: replacing raw SQL with the ORM `Q(...)` filter.
- WHAT I SAY: "The ORM parameterizes automatically, so the same payload now returns zero rows and the quote returns 200. My regression test asserts no email ever appears in results."
- VIEWER SEES: failing test → passing test, curl 500 → 200.

**Airtable idempotency**
- WHAT I'M SHOWING: two consecutive exports + the Airtable grid.
- WHAT I SAY: "The client indexes existing records by TaskId, so the first run creates 7 and the second updates the same 7 — no duplicates. Transient errors retry with backoff; one bad record doesn't abort the batch."
- VIEWER SEES: `created: 7` then `updated: 7, created: 0`; Airtable still 7 rows.
