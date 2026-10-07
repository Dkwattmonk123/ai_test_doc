# REVIEW.md — Code Review (template)

> Fill the `curl` outputs with your **own** captured runs. Issues below were verified live against the seeded app during analysis; prioritized by business impact. Keep the top 4.

## Issue #1 — SQL injection in task search (CRITICAL)
- **Category:** Security · **Severity:** Critical
- **File:** `backend/projects/views.py:110-123` — `TaskListCreateView.get`, `if q:` branch
- **Description:** The `?q=` search interpolates the user-supplied `q` (and `project_id`) directly into raw SQL via f-strings and runs it with `cursor.execute(sql)`. Any authenticated user can read arbitrary tables, including every user's email and password hash. A lone quote returns HTTP 500, confirming unsanitized concatenation.
- **Recommended fix:** Replace the raw SQL with the ORM `Task.objects.filter(project_id=...).filter(Q(title__icontains=q) | Q(description__icontains=q))`, or use a parameterized query with `%s` placeholders.
- **Proof (curl):**
  ```bash
  # single quote -> 500
  curl -s -o /dev/null -w "HTTP %{http_code}\n" \
    "http://localhost:8000/api/projects/$PID/tasks?q=%27" -H "Authorization: Bearer $TOKEN"
  # UNION dumps users (emails + password hashes)
  Q="x%27%29%20UNION%20SELECT%20id%2C%20id%2C%20email%2C%20password%2C%20%27todo%27%2C%20NULL%3A%3Auuid%2C%20id%2C%200%2C%20created_at%2C%20updated_at%20FROM%20users--%20"
  curl -s "http://localhost:8000/api/projects/$PID/tasks?q=$Q" -H "Authorization: Bearer $TOKEN"
  ```
  **Output (paste your real run here):**
  ```
  HTTP 500
  ... rows where title = user email, description = pbkdf2_sha256$... hash ...
  ```

## Issue #2 — Broken object-level authorization on task update (CRITICAL)
- **Category:** Security · **Severity:** Critical
- **File:** `backend/projects/views.py:164-185` — `TaskDetailView.patch`
- **Description:** `patch` loads a task by id and saves changes without any membership or role check (unlike `delete` directly below it). A non-member or a viewer can rename tasks and change status/assignee in any project (IDOR / cross-tenant tampering).
- **Recommended fix:** After loading the task, call `_get_membership(request.user, task.project_id)`; return 403 if `None` or if `not _can_edit_tasks(membership.role)` — mirror `TaskDetailView.delete`.
- **Proof (curl):**
  ```bash
  # lina (not a member of Q3 Launch) edits a Q3 task -> 200 (should be 403)
  curl -s -o /dev/null -w "HTTP %{http_code}\n" -X PATCH http://localhost:8000/api/tasks/$TID \
    -H "Authorization: Bearer $LINA_TOKEN" -H 'Content-Type: application/json' \
    -d '{"title":"HACKED by non-member"}'
  ```

## Issue #3 — Insecure authentication & global config (HIGH)
- **Category:** Security · **Severity:** High
- **File:** `backend/taskboard/settings.py` (lines 9, 51-54, 56) + `users/serializers.py:14`
- **Description:** 30-day JWT access tokens with no refresh/rotation/revocation; `ALLOWED_HOSTS=['*']`; `CORS_ALLOW_ALL_ORIGINS=True`; no password validators (only `min_length=8`); no rate limiting on login/register.
- **Recommended fix:** Shorten access tokens + add refresh rotation & blacklist; env-driven `ALLOWED_HOSTS`/CORS; add `AUTH_PASSWORD_VALIDATORS`; add DRF throttling.

## Issue #4 — Unvalidated assignee + N+1 on dashboard (MEDIUM)
- **Category:** Data Integrity / Performance · **Severity:** Medium
- **File:** `backend/projects/views.py:156,180-181` (assignee) and `:39` (N+1)
- **Description:** Task create/update accept any `assigneeId` without verifying the user is a project member. Separately, `ProjectListCreateView.get` calls `p.tasks.count()` per project, ignoring the prefetch and issuing one extra COUNT query per project.
- **Recommended fix:** Validate `assigneeId` against the project's memberships (400/422 otherwise). Replace `.count()` with a `Count('tasks')` annotation or `len(p.tasks.all())`.
