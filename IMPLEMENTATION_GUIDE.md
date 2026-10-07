# TaskBoard Assignment — Implementation Guide

> **Status:** This is an **engineering manual**, not a finished submission. It was produced by analyzing a live clone of `https://github.com/ajackus/q-taskboard`. Every bug and command below was reproduced against a running instance of the app. Follow it top-to-bottom from a clean clone and you can complete the assignment without an AI assistant.
>
> **You still must, yourself:** clone the repo, run the steps, implement the code, run the tests, verify Airtable, commit, and record the screen capture. This guide does not do any of that for you.

---

## 0. What the assignment actually asks for

Source: `ASSIGNMENT.md` (present in commit `84cb1a6`, removed in `5f58e52` — recover it with `git show 84cb1a6:ASSIGNMENT.md`).

| # | Deliverable | Required |
|---|---|---|
| 1 | `REVIEW.md` — top 4 issues, prioritized by business impact, ≥1 with a real `curl` proof | ✅ |
| 2 | Fix the #1 issue — commit + tests + before/after `curl` | ✅ |
| 3a **or** 3b | Comments **or** Activity feed, with tests | ✅ (one) |
| 3c | Real Airtable export via **`pyairtable`** — idempotent, retry transient, skip permanent, per-record isolation | ✅ (mandatory) |
| — | `TERMINAL_LOG.md` in a **specific order** (see §12) | ✅ |
| — | Screen recording (Loom etc.), terminal visible, narrated; link in `README.md` or `RECORDING.md` | ✅ |
| — | Full commit history, **do not squash**, **do not modify seed data** | ✅ |

Bonus: do **both** 3a and 3b; make the export async.

> ⚠️ **Pre-commit hook.** `.git-hooks/pre-commit` auto-captures AI tool logs into `.ai-conversations/`. `bin/setup` wires it via `git config core.hooksPath .git-hooks`. This is disclosed and intentional — leave it on; it is part of the evaluation. It never blocks a commit (`exit 0`).

---

## 1. Repository architecture

```
q-taskboard/
├── docker-compose.yml        # db (postgres:16) + backend (8000) + frontend (3000)
├── .env.example              # copy to .env; POSTGRES_*, DJANGO_SECRET_KEY, AIRTABLE_*
├── bin/setup                 # non-Docker bootstrap; also sets core.hooksPath
├── .git-hooks/pre-commit     # captures AI logs into .ai-conversations/
├── backend/                  # Django 5 + DRF, pytest-django
│   ├── taskboard/
│   │   ├── settings.py       # ⚠ ALLOWED_HOSTS=['*'], CORS_ALLOW_ALL, 30-day JWT, no pw validators
│   │   └── urls.py           # /api/health + users + projects
│   ├── users/                # custom User (email login, UUID pk), JWT auth
│   │   ├── models.py  views.py  serializers.py  urls.py  tests.py (8 tests)
│   ├── projects/
│   │   ├── models.py         # Project, Membership(role: admin/member/viewer), Task
│   │   ├── views.py          # ⚠ ALL the bugs live here
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   ├── tests.py          # 7 tests
│   │   └── management/commands/seed.py   # DO NOT EDIT
│   ├── requirements.txt      # pyairtable already pinned (2.3–3.0)
│   └── pytest.ini
└── frontend/                 # React 18 + Vite 5 + TS strict + TanStack Query 5 + Tailwind
    └── src/
        ├── lib/api-client.ts # apiFetch wrapper; token in localStorage
        ├── pages/            # Login, Register, Dashboard, Project
        ├── components/       # Header, StatusColumn, TaskCard, TaskDetail
        ├── types/index.ts
        └── tests/            # TaskCard.test.tsx, schemas.test.ts (vitest)
```

**Request flow:** React (`apiFetch`, token from `localStorage`) → Vite dev proxy `/api` → Django DRF (`JWTAuthentication`, `IsAuthenticated` default) → PostgreSQL.

**Authorization model:** `Membership(user, project, role)` where role ∈ `admin|member|viewer`. Helpers in `projects/views.py`: `_get_membership(user, project_id)` and `_can_edit_tasks(role)` (true for admin/member). **The bugs are places where these helpers are not called.**

**What does NOT exist (despite being referenced):**
- `backend/projects/airtable_mock.py` — README §"Airtable Export" claims it exists. It does **not**. You will create it (§10).
- `POST /api/projects/:id/export` real logic — it is a stub returning `{'exported': 0, ...}` (`views.py:234-243`).

---

## 2. Prerequisites

Verify before starting:

```bash
git --version        # any recent
docker --version     # Docker + compose v2 — recommended path
python3 --version    # 3.12+  (Docker image uses 3.12)
node --version       # 20+    (repo targets Node 20; 18 may warn)
npm --version
```

You also need an **Airtable account**, a **base**, and a **Personal Access Token** (see §9) before the recording session. Set this up in advance — it is the mandatory part.

---

## 3. Clean clone + setup (Docker — recommended)

```bash
git clone https://github.com/ajackus/q-taskboard
cd q-taskboard
cp .env.example .env            # then edit AIRTABLE_* (see §9)
git config core.hooksPath .git-hooks   # ensure the eval hook is active

docker-compose up --build       # terminal 1 — leave running
# in terminal 2:
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py seed
```

- App: **http://localhost:3000**  · API: **http://localhost:8000**
- Health check: `curl http://localhost:8000/api/health` → `{"ok": true}`
- Login (seed): `meera@taskboard.dev` / `password123`

**Expected `migrate` output:** a list of `Applying ... OK` lines ending with `projects.0001_initial... OK`.
**Expected `seed` output:** `seed complete.` followed by the 5 login accounts.

### Non-Docker (fallback)

Requires local PostgreSQL 15+ on `localhost:5432` with db/user/password all `taskboard` (matching `.env`).

```bash
# Postgres (one-time): createuser/createdb named taskboard, password taskboard
cd backend
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp ../.env.example ../.env
python manage.py migrate
python manage.py seed
python -m pytest            # expect: 15 passed
python manage.py runserver  # :8000

cd ../frontend
npm install
npm test                    # vitest
npm run dev                 # :3000
```

> **Verified baseline (on a clean clone):** `pytest` → **15 passed** (7 projects + 8 users). `npm test` → vitest suites in `TaskCard.test.tsx` and `schemas.test.ts` pass. If a migration check complains about a renamed index (`tasks_project_status_idx`), that is cosmetic from a newer Django and not a bug to fix — leave migrations as-is.

---

## 4. Initial verification (before changing anything)

1. `docker-compose up` → open http://localhost:3000 → login as `meera@taskboard.dev` / `password123`.
2. Dashboard shows 3 projects (Q3 Launch, Customer Onboarding Revamp, Internal Tools Cleanup).
3. Open **Q3 Launch** → Kanban board with 7 tasks across 4 columns; members list at the bottom.
4. Run the test baseline and **record the output** — it becomes the first block of `TERMINAL_LOG.md`:
   ```bash
   docker-compose exec backend python -m pytest
   docker-compose exec frontend npm test
   ```

**Get a token for curl** (used throughout):
```bash
TOKEN=$(curl -s -X POST http://localhost:8000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
echo "$TOKEN"
```
Get a project id:
```bash
curl -s http://localhost:8000/api/projects -H "Authorization: Bearer $TOKEN" \
  | python3 -c 'import sys,json;[print(p["id"],p["name"],p["role"]) for p in json.load(sys.stdin)["projects"]]'
```

---

## 5. Part 1 — Code review (`REVIEW.md`)

Create `REVIEW.md` listing the **top 4 issues**, prioritized by business impact, each with: file:line · category · severity · 2–3 sentence description · recommended fix. At least one needs a real `curl` proof. Use `REVIEW_TEMPLATE.md` (provided) as the skeleton.

Below are the four issues **verified live** during analysis, already prioritized. **#1 and #2 are both real, exploitable security holes** — #1 is the Part 2 fix.

### Issue #1 — SQL injection in task search (CRITICAL · Security) ← fix this in Part 2

- **File:** [`backend/projects/views.py:110-123`](backend/projects/views.py#L110-L123), `TaskListCreateView.get`, the `if q:` branch.
- **Root cause:** the `?q=` search builds raw SQL by f-string interpolation of `project_id` **and** the user-controlled `q`:
  ```python
  sql = (f"... FROM tasks WHERE project_id = '{project_id}' "
         f"AND (title ILIKE '%{q}%' OR description ILIKE '%{q}%') ORDER BY position ASC")
  cursor.execute(sql)
  ```
- **Impact:** any authenticated user can read the entire database. **Verified exfiltration of every user's email + password hash** through the search response (see §6). A single quote (`?q='`) returns **HTTP 500**, confirming unsanitized concatenation.
- **Category:** Security · **Severity:** Critical.
- **Fix:** use a parameterized query (`cursor.execute(sql, [project_id, like, like])` with `%s` placeholders), or — cleaner — drop the raw SQL entirely and use the ORM: `Task.objects.filter(project_id=project_id).filter(Q(title__icontains=q) | Q(description__icontains=q))`. See §7.

### Issue #2 — Broken object-level authorization on task update (CRITICAL · Security)

- **File:** [`backend/projects/views.py:164-185`](backend/projects/views.py#L164-L185), `TaskDetailView.patch`.
- **Root cause:** `patch` loads the task by id and saves changes **without ever calling `_get_membership` or `_can_edit_tasks`**. (Compare `TaskDetailView.delete` right below it, which *does* check — so the omission is clearly a bug, not a design choice.)
- **Impact:** **Verified** — a user who is *not a member* of the project, and a user who is only a *viewer*, can both rename a task and change its status/assignee via `PATCH /api/tasks/:id`. Cross-tenant data tampering (IDOR).
- **Category:** Security · **Severity:** Critical.
- **Fix:** after loading the task, resolve membership on `task.project_id`, 403 if `None`, and 403 if `not _can_edit_tasks(role)` — mirroring `delete`. (If you fix this instead of #1 as your "critical", that is defensible; but #1 leaks all credentials, so #1 ranks higher.)

### Issue #3 — Insecure authentication & global config (HIGH · Security)

- **File:** [`backend/taskboard/settings.py`](backend/taskboard/settings.py) — `ACCESS_TOKEN_LIFETIME = 30 days` with no refresh/rotation/revocation (`SIMPLE_JWT`, line 51); `ALLOWED_HOSTS = ['*']` (line 9); `CORS_ALLOW_ALL_ORIGINS = True` (line 56); no `AUTH_PASSWORD_VALIDATORS`; register only enforces `min_length=8` (`users/serializers.py:14`). No throttling anywhere.
- **Impact:** a leaked token is valid for 30 days and cannot be revoked; permissive CORS/hosts widen attack surface; weak passwords allowed; login/register are brute-forceable (no rate limit).
- **Category:** Security · **Severity:** High.
- **Fix (recommend, not required to implement):** shorten access token + add refresh rotation + blacklist; restrict `ALLOWED_HOSTS`/CORS via env; add Django password validators; add DRF throttling.

### Issue #4 — Assignee not validated + N+1 on dashboard (MEDIUM · Data Integrity / Performance)

- **Data integrity:** `TaskListCreateView.post` ([`views.py:156`](backend/projects/views.py#L156)) and `TaskDetailView.patch` ([`views.py:180-181`](backend/projects/views.py#L180-L181)) accept any `assigneeId` without checking the user is a member of the project — you can assign tasks to arbitrary users. (Note: `on_delete=SET_NULL` means a stale id just needs validating against membership.)
- **Performance:** `ProjectListCreateView.get` ([`views.py:39`](backend/projects/views.py#L39)) computes `p.tasks.count()` per project, which **ignores** the `prefetch_related('project__tasks')` cache and issues one extra `COUNT` query per project (N+1). Use `Count('tasks')` annotation or `len(p.tasks.all())`.
- **Category:** Data Integrity + Performance · **Severity:** Medium.

> You only need 4 issues. If you prefer a different 4th, other real candidates: no pagination on task/project lists; `PATCH /api/tasks/:id` also lacks the project-membership check for assignee (same as #2); `description` f-string in `seed` is fine. Don't pad — pick the highest-impact four.

---

## 6. Part 1 — the required `curl` proof (SQL injection, #1)

These were **run live** during analysis against the seeded DB. Re-run them on your machine and paste the real output into `REVIEW.md` and `TERMINAL_LOG.md`. (`$TOKEN` and `$PID` from §4; use the **Q3 Launch** project id.)

**Proof A — 500 on a lone quote (confirms concatenation):**
```bash
curl -s -o /dev/null -w "q=' -> HTTP %{http_code}\n" \
  "http://localhost:8000/api/projects/$PID/tasks?q=%27" \
  -H "Authorization: Bearer $TOKEN"
# Observed: HTTP 500
```

**Proof B — UNION dumps every user's email + password hash** (the dramatic one). Decoded payload:
```
x') UNION SELECT id, id, email, password, 'todo', NULL::uuid, id, 0, created_at, updated_at FROM users--
```
URL-encoded request:
```bash
Q="x%27%29%20UNION%20SELECT%20id%2C%20id%2C%20email%2C%20password%2C%20%27todo%27%2C%20NULL%3A%3Auuid%2C%20id%2C%200%2C%20created_at%2C%20updated_at%20FROM%20users--%20"
curl -s "http://localhost:8000/api/projects/$PID/tasks?q=$Q" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c 'import sys,json;[print(t["title"],"|",t["description"][:50]) for t in json.load(sys.stdin)["tasks"]]'
```
**Observed (real):** 5 rows, each a user — e.g. `meera@taskboard.dev | pbkdf2_sha256$1000000$...`. The emails land in the `title` field and the password hashes in `description`. This proves arbitrary table read via the search endpoint.

> Why 10 columns / those casts: the original `SELECT` lists 10 columns with types `uuid, uuid, varchar, text, varchar, uuid, uuid, int, timestamptz, timestamptz`. The UNION must match them, hence `id, id, email, password, 'todo', NULL::uuid, id, 0, created_at, updated_at`. The trailing `--` comments out the rest of the original WHERE/ORDER BY.

---

## 7. Part 2 — fix the #1 issue (SQL injection) + regression test

### The fix

Edit `TaskListCreateView.get` in [`backend/projects/views.py`](backend/projects/views.py#L110-L123). Replace the raw-SQL `if q:` branch with the ORM (removes the `from django.db import connection` dependency for this path):

```python
from django.db.models import Q   # add near the top

# inside TaskListCreateView.get, replacing the whole `if q:` block:
q = request.query_params.get('q')
tasks = (
    Task.objects
    .filter(project_id=project_id)
    .select_related('assignee')
    .order_by('status', 'position')
)
if q:
    tasks = tasks.filter(Q(title__icontains=q) | Q(description__icontains=q))
return Response({'tasks': TaskSerializer(tasks, many=True).data})
```

Why the ORM over parameterized raw SQL: it is injection-proof by construction, returns the same serialized shape as the non-search path (the raw branch returned raw DB rows with a different shape — a latent inconsistency), and keeps `project_id` scoping intact. If you must keep raw SQL, the minimum fix is:
```python
sql = ("SELECT ... FROM tasks WHERE project_id = %s "
       "AND (title ILIKE %s OR description ILIKE %s) ORDER BY position ASC")
cursor.execute(sql, [project_id, f'%{q}%', f'%{q}%'])
```

### Regression test

Add to `backend/projects/tests.py` inside `TestTasks` (or a new `TestTaskSearch`):

```python
def test_search_is_not_sql_injectable(self, auth_client, user):
    project = Project.objects.create(name='P', owner=user)
    Membership.objects.create(user=user, project=project, role='admin')
    Task.objects.create(project=project, title='Record demo video',
                        created_by=user, status='todo')
    # a lone quote must NOT 500
    resp = auth_client.get(f'/api/projects/{project.id}/tasks', {'q': "'"})
    assert resp.status_code == 200
    # a UNION payload must return no rows (not leak users)
    payload = ("x') UNION SELECT id,id,email,password,'todo',NULL::uuid,"
               "id,0,created_at,updated_at FROM users--")
    resp = auth_client.get(f'/api/projects/{project.id}/tasks', {'q': payload})
    assert resp.status_code == 200
    titles = [t['title'] for t in resp.data['tasks']]
    assert not any('@' in t for t in titles)   # no emails leaked

def test_search_matches_title(self, auth_client, user):
    project = Project.objects.create(name='P', owner=user)
    Membership.objects.create(user=user, project=project, role='admin')
    Task.objects.create(project=project, title='Record demo video', created_by=user)
    Task.objects.create(project=project, title='Draft press release', created_by=user)
    resp = auth_client.get(f'/api/projects/{project.id}/tasks', {'q': 'demo'})
    assert resp.status_code == 200
    assert [t['title'] for t in resp.data['tasks']] == ['Record demo video']
```

Run: `pytest backend/projects/tests.py -k search -v`. **Before the fix**, `test_search_is_not_sql_injectable` fails (500 / emails present); **after**, it passes.

### Before/after curl

- **Before:** on the unfixed code, run §6 Proof A + B, capture the 500 and the leaked rows.
- **After:** apply the fix, restart backend, re-run the *same* commands → Proof A returns `HTTP 200`, Proof B returns `rows: 0` (no user data). Put both blocks, labeled, into `TERMINAL_LOG.md`.

> If instead you choose **Issue #2** as your critical fix, mirror `delete` in `patch`: load task, `membership=_get_membership(request.user, task.project_id)`, 403 if none or `not _can_edit_tasks(membership.role)`; test with lina (non-member) and dev (viewer) both getting 403.

---

## 8. Part 3 — pick Comments (3a) or Activity feed (3b)

**Recommendation: implement 3a (Comments).** It is smaller, self-contained, append-only (no update/delete endpoints to secure), has a crisp authorization rule that reuses existing helpers, and is easy to prove in the recording. Do 3b too for bonus if time allows; 3b forces the transaction-decision write-up which is extra surface area.

### 3a — Comments

**Model** — new file `backend/projects/` additions to `models.py`:
```python
class Comment(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    task = models.ForeignKey(Task, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                               related_name='comments')
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'comments'
        ordering = ['created_at']        # chronological
```
Then: `python manage.py makemigrations projects && python manage.py migrate`.

**Serializer** (`serializers.py`):
```python
class CommentSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)
    class Meta:
        model = Comment
        fields = ['id', 'body', 'author', 'created_at']
```

**View** (`views.py`) — append-only: only `GET` (list) and `POST` (create). No PUT/PATCH/DELETE routes at all, which enforces append-only at the routing layer.
```python
class CommentListCreateView(APIView):
    def get(self, request, task_id):
        task = Task.objects.select_related('project').filter(id=task_id).first()
        if not task:
            return Response({'error': 'not found'}, status=404)
        if not _get_membership(request.user, task.project_id):   # any member can read
            return Response({'error': 'forbidden'}, status=403)
        comments = task.comments.select_related('author').all()  # ordered by Meta
        return Response({'comments': CommentSerializer(comments, many=True).data})

    def post(self, request, task_id):
        task = Task.objects.select_related('project').filter(id=task_id).first()
        if not task:
            return Response({'error': 'not found'}, status=404)
        membership = _get_membership(request.user, task.project_id)
        if not membership:
            return Response({'error': 'forbidden'}, status=403)
        if not _can_edit_tasks(membership.role):      # viewers can read, not post
            return Response({'error': 'viewers cannot comment'}, status=403)
        body = (request.data.get('body') or '').strip()
        if not body:
            return Response({'error': 'body is required'}, status=400)
        c = Comment.objects.create(task=task, author=request.user, body=body)
        return Response({'comment': CommentSerializer(c).data}, status=201)
```

**URLs** (`projects/urls.py`): `path('tasks/<uuid:task_id>/comments', CommentListCreateView.as_view())`.

**Authorization contract to prove:** member/admin → can read + post; viewer (dev@example.com on Q3) → can read, **403** on post; non-member (lina@example.com on Q3) → **403** on both.

**Frontend** (`components/TaskDetail.tsx`): add a comments section — `useQuery(['comments', task.id], () => apiFetch('/api/tasks/'+task.id+'/comments'))`, render chronologically (author · body · `new Date(created_at).toLocaleString()`), a post form wired to `useMutation` POST that invalidates `['comments', task.id]`; hide the form when the current user's role is `viewer`.

**Tests** (`projects/tests.py`): member can post (201) and list; viewer gets 403 on post but 200 on list; non-member gets 403; comments come back in chronological order; there is **no** edit/delete route (append-only).

### 3b — Activity feed (if you also do it / bonus)

Model `Activity(id, project FK, actor FK, verb, task FK null, metadata JSONField, created_at)`; write a record on task create, status change, assignee change, comment added; endpoint `GET /api/projects/:id/activity` (members only, `order_by('-created_at')`, most-recent-first). See `DESIGN_NOTES.md` for the required **transaction decision** (if the activity write fails, does the original change roll back?) — decide, implement, and justify in 2–3 sentences. Recommended stance documented there: **best-effort, non-blocking** (the primary mutation commits even if the audit write fails) — justified in `DESIGN_NOTES.md`.

---

## 9. Part 3c — Airtable setup (mandatory, do before recording)

1. Create a free **Airtable** account.
2. Create a **Base** (e.g. "TaskBoard Export").
3. Create a **Table** named `Tasks` (matches `AIRTABLE_TABLE_NAME`).
4. Add fields (names are case-sensitive, must match your mapping in §10):
   - `TaskId` (Single line text) — the idempotency key
   - `Title` (Single line text)
   - `Description` (Long text)
   - `Status` (Single line text, or Single select with todo/in_progress/review/done)
   - `Assignee` (Single line text)
   - `Position` (Number)
   - `ProjectId` (Single line text)
5. Create a **Personal Access Token** at https://airtable.com/create/tokens with scopes `data.records:read` + `data.records:write`, and access to your base.
6. Find the **Base ID** (`app…`) — it's in the base URL or via the API docs for your base.
7. Put them in `.env` (never commit real secrets; `.env` is gitignored):
   ```
   AIRTABLE_API_KEY=pat...your token...
   AIRTABLE_BASE_ID=appXXXXXXXXXXXXXX
   AIRTABLE_TABLE_NAME=Tasks
   ```
   For Docker, also add these three to the `backend.environment:` block in `docker-compose.yml` (the compose file currently does **not** pass them through — add them, or use `env_file: .env`).

---

## 10. Part 3c — implement the real export

`pyairtable` is already in `requirements.txt`. The endpoint stub is `ExportView.post` ([`views.py:234-243`](backend/projects/views.py#L234-L243)) and authorization (admin/member only) is already correct there — keep it, replace the body.

**Data mapping** (`Task` → Airtable `Tasks`):

| Task field | Airtable field | Transformation |
|---|---|---|
| `id` (UUID) | `TaskId` | `str(id)` — **idempotency key** |
| `title` | `Title` | direct |
| `description` | `Description` | `or ''` |
| `status` | `Status` | direct |
| `assignee` | `Assignee` | `assignee.name if assignee else ''` |
| `position` | `Position` | int |
| `project_id` | `ProjectId` | `str(project_id)` |

**New file `backend/projects/airtable_client.py`:**
```python
import time
from django.conf import settings
from pyairtable import Api
from requests.exceptions import RequestException

TRANSIENT_STATUS = {429, 500, 502, 503, 504}

def _status_of(exc):
    resp = getattr(exc, 'response', None)
    return getattr(resp, 'status_code', None)

def _with_retry(fn, *, retries=3, base_delay=0.5):
    """Retry transient failures with bounded exponential backoff.
    Re-raise permanent failures (4xx except 429) immediately."""
    attempt = 0
    while True:
        try:
            return fn()
        except RequestException as exc:
            code = _status_of(exc)
            transient = code in TRANSIENT_STATUS or code is None  # None = network/timeout
            if not transient or attempt >= retries:
                raise
            time.sleep(base_delay * (2 ** attempt))
            attempt += 1

def get_table():
    api = Api(settings.AIRTABLE_API_KEY)
    return api.table(settings.AIRTABLE_BASE_ID, settings.AIRTABLE_TABLE_NAME)

def fields_for(task):
    return {
        'TaskId': str(task.id),
        'Title': task.title,
        'Description': task.description or '',
        'Status': task.status,
        'Assignee': task.assignee.name if task.assignee else '',
        'Position': task.position,
        'ProjectId': str(task.project_id),
    }

def export_tasks(tasks, table=None):
    """Idempotent upsert keyed on TaskId. Per-record isolation: one bad
    record does not abort the run. Returns a summary dict."""
    table = table or get_table()
    # one read to build an index (idempotency): TaskId -> airtable record id
    existing = _with_retry(lambda: table.all(fields=['TaskId']))
    index = {r['fields'].get('TaskId'): r['id'] for r in existing}
    created = updated = failed = 0
    errors = []
    for task in tasks:
        f = fields_for(task)
        try:
            rec_id = index.get(str(task.id))
            if rec_id:
                _with_retry(lambda: table.update(rec_id, f))
                updated += 1
            else:
                _with_retry(lambda: table.create(f))
                created += 1
        except Exception as exc:                      # per-record isolation
            failed += 1
            errors.append({'taskId': str(task.id), 'error': str(exc)})
    return {'created': created, 'updated': updated, 'failed': failed,
            'exported': created + updated, 'errors': errors}
```

Add to `settings.py`:
```python
AIRTABLE_API_KEY = os.environ.get('AIRTABLE_API_KEY', '')
AIRTABLE_BASE_ID = os.environ.get('AIRTABLE_BASE_ID', '')
AIRTABLE_TABLE_NAME = os.environ.get('AIRTABLE_TABLE_NAME', 'Tasks')
```

**Rewrite `ExportView.post`** (keep the existing auth checks, lines 235-240):
```python
class ExportView(APIView):
    def post(self, request, project_id):
        membership = _get_membership(request.user, project_id)
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)
        if not _can_edit_tasks(membership.role):
            return Response({'error': 'only admins and members can export'},
                            status=status.HTTP_403_FORBIDDEN)
        if not settings.AIRTABLE_API_KEY or not settings.AIRTABLE_BASE_ID:
            return Response({'error': 'airtable not configured'}, status=503)
        tasks = (Task.objects.filter(project_id=project_id)
                 .select_related('assignee', 'created_by').order_by('position'))
        from .airtable_client import export_tasks
        try:
            summary = export_tasks(tasks)
        except Exception as exc:
            return Response({'error': f'export failed: {exc}'}, status=502)
        return Response(summary)
```
(Add `from django.conf import settings` at the top of `views.py`.)

**Test double `backend/projects/airtable_mock.py`** (README references this; use it in unit tests so tests never hit the network — production uses the real `Api`):
```python
class MockTable:
    """In-memory stand-in for a pyairtable Table. Optionally fail chosen TaskIds."""
    def __init__(self, fail_ids=None):
        self._rows = {}          # airtable record id -> fields
        self._seq = 0
        self.fail_ids = set(fail_ids or [])
    def all(self, fields=None):
        return [{'id': rid, 'fields': f} for rid, f in self._rows.items()]
    def create(self, fields):
        if fields.get('TaskId') in self.fail_ids:
            raise RuntimeError('permanent 422 for ' + fields['TaskId'])
        self._seq += 1
        rid = f'rec{self._seq}'
        self._rows[rid] = fields
        return {'id': rid, 'fields': fields}
    def update(self, rid, fields):
        if fields.get('TaskId') in self.fail_ids:
            raise RuntimeError('permanent 422 for ' + fields['TaskId'])
        self._rows[rid] = {**self._rows.get(rid, {}), **fields}
        return {'id': rid, 'fields': self._rows[rid]}
```

**Airtable tests** (`projects/tests.py`), using `export_tasks(tasks, table=MockTable(...))`:
- **Export writes all tasks:** N tasks → `created == N`, mock has N rows.
- **Idempotency:** run twice on the same `MockTable` → second run `created == 0`, `updated == N`, still N rows (no duplicates).
- **Partial failure isolation:** `MockTable(fail_ids={str(task_b.id)})` → `failed == 1`, A and C still `created`/present; one bad record didn't abort the batch.
- **Authorization:** viewer/non-member → 403 on `POST /api/projects/:id/export` (test the view, not the client).

> **Retry/transient note for the recording:** retry logic lives in `_with_retry` (retries 429/5xx/network with bounded backoff, re-raises 4xx). You can unit-test it by passing a fake that raises a `RequestException` with a `.response.status_code` of 503 twice then succeeds; assert it eventually returns. Keep real retries out of the live demo (they're slow) — demonstrate them with a test.

---

## 11. Manual Airtable verification (for the recording)

1. **First export:** from the Project page trigger (add an "Export to Airtable" button calling `POST /api/projects/:id/export`, members only), or `curl`:
   ```bash
   curl -s -X POST http://localhost:8000/api/projects/$PID/export \
     -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
   # expect: {"created": 7, "updated": 0, "failed": 0, "exported": 7, "errors": []}
   ```
   Open Airtable → the `Tasks` table shows 7 rows (for Q3 Launch). **Screenshot or share link.**
2. **Second export (idempotency):** run the exact same curl again →
   ```
   {"created": 0, "updated": 7, "failed": 0, "exported": 7, ...}
   ```
   Refresh Airtable → **still 7 rows**, no duplicates. Screenshot both the terminal and Airtable.

---

## 12. `TERMINAL_LOG.md` — required order

The assignment fixes this order. Capture each block (tip: `script -a terminal_log.txt` records your whole session):

1. **Setup output** — `docker-compose up` / migrate / seed.
2. **Initial test run** — `pytest` (15 passed) + `npm test`.
3. **Bug curl proof (before)** — §6 Proof A (HTTP 500) + Proof B (leaked users).
4. **Fix curl proof (after)** — same curls → 200 + 0 rows; plus `pytest -k search` green.
5. **Airtable export demo** — first export, `created: 7`; Airtable screenshot/link.
6. **Second Airtable run** — `updated: 7`, `created: 0`; Airtable still 7 rows (idempotency).
7. **Part 3a (or 3b) demo** — member posts a comment (201), viewer gets 403, list is chronological.
8. **Final test run** — full `pytest` + `npm test`, all green (now includes your new tests).

Use `TERMINAL_LOG_TEMPLATE.md` as the scaffold.

---

## 13. Recommended implementation order

```
01 clone + cp .env.example .env + git config core.hooksPath .git-hooks
02 docker-compose up --build ; migrate ; seed
03 verify login + board ; capture baseline pytest/npm test     (LOG block 1-2)
04 write REVIEW.md (4 issues)                                   (Part 1)
05 reproduce SQL injection via curl                             (LOG block 3)
06 fix injection (ORM) + regression tests ; re-run curl        (Part 2, LOG block 4)
07 commit: "fix: parameterize task search (SQL injection)"
08 implement Comments (model/migration/view/urls/frontend/tests)(Part 3a)
09 commit: "feat: task comments (append-only, member-gated)"
10 Airtable: base/token/fields + .env + compose env            (setup)
11 implement airtable_client.py + airtable_mock.py + ExportView (Part 3c)
12 Airtable tests (all-export, idempotency, partial-failure, authz)
13 commit: "feat: real Airtable export (idempotent, retry, isolated)"
14 run export twice ; screenshot Airtable                      (LOG block 5-6)
15 (optional) Activity feed + DESIGN_NOTES transaction decision (3b bonus)
16 full test suite green                                        (LOG block 8)
17 assemble TERMINAL_LOG.md in required order
18 write DESIGN_NOTES.md ; add recording link to README.md
19 record the full session (see RECORDING_CHECKLIST.md)
20 push ; submit
```

Commit after each logical unit — the evaluators read commit history; **do not squash**.

---

## 14. Troubleshooting (high-value cases)

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop not running | start Docker Desktop; re-run `docker-compose up` |
| Port 5432/8000/3000 already in use | local Postgres / stale server | `lsof -nP -iTCP:8000 -sTCP:LISTEN`; stop it or change the published port |
| `migrate` → `connection refused` | db container not ready / wrong `POSTGRES_HOST` | wait for db healthy; in Docker `POSTGRES_HOST=db`, locally `localhost` |
| `pytest` can't reach DB (non-Docker) | local Postgres down or role/db missing | create role+db `taskboard`/`taskboard`; `pg_ctl ... start` |
| `initdb: invalid locale` (macOS) | unset `LC_*` | `export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8` or `initdb --locale=C` |
| `?q='` returns 500 **after** your fix | still hitting raw SQL | confirm the ORM branch replaced the `if q:` raw block; restart backend |
| Airtable `401`/`403` | bad token / no base access | re-mint PAT with `data.records:read+write` scoped to the base |
| Airtable `422 INVALID_VALUE_FOR_COLUMN` | field name/type mismatch | field names are case-sensitive; match §10 exactly; Status select options must include your values |
| Airtable `429` | rate limit (5 req/s/base) | handled by `_with_retry`; for ~1000 tasks consider `batch_create`/`batch_upsert` (10/req) |
| Export curl → `airtable not configured` (503) | `.env` not loaded into backend | add `AIRTABLE_*` to `docker-compose.yml` backend env or `env_file: .env`; restart |
| Frontend can't reach API | proxy target | Docker sets `API_TARGET=http://backend:8000`; locally Vite defaults to `http://localhost:8000` |
| Duplicates appear in Airtable on 2nd run | idempotency key not found | ensure `TaskId` field exists and `export_tasks` indexes on it before create |

**Debugging a backend API issue:** reproduce with `curl` → check the backend container logs (`docker-compose logs -f backend`) → find the route in `projects/urls.py` → the view in `projects/views.py` → the query. **Frontend:** DevTools Network tab → failed request + response → `src/lib/api-client.ts` (`apiFetch`) → the calling component.

---

## 15. Command reference

```bash
# Setup
git clone https://github.com/ajackus/q-taskboard && cd q-taskboard
cp .env.example .env ; git config core.hooksPath .git-hooks
docker-compose up --build
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py seed
# Tests
docker-compose exec backend python -m pytest            # 15 passed baseline
docker-compose exec backend python -m pytest -k search -v
docker-compose exec frontend npm test
# Auth token
TOKEN=$(curl -s -X POST localhost:8000/api/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
# Projects / tasks
curl -s localhost:8000/api/projects -H "Authorization: Bearer $TOKEN"
curl -s "localhost:8000/api/projects/$PID/tasks" -H "Authorization: Bearer $TOKEN"
# Export (after Part 3c)
curl -s -X POST localhost:8000/api/projects/$PID/export -H "Authorization: Bearer $TOKEN"
# Django
docker-compose exec backend python manage.py makemigrations projects
docker-compose exec backend python manage.py migrate
# Session capture
script -a terminal_log.txt
```

---

## 16. Final checklist

```
Setup       [ ] clean clone runs  [ ] migrate+seed OK  [ ] login works  [ ] baseline 15 pass
Part 1      [ ] REVIEW.md  [ ] 4 issues prioritized  [ ] ≥1 curl proof (SQLi)
Part 2      [ ] #1 fixed  [ ] regression test  [ ] before curl  [ ] after curl  [ ] commit
Part 3a/3b  [ ] comments (or activity) works  [ ] authz enforced  [ ] tests
Part 3c     [ ] real pyairtable  [ ] member-only  [ ] all tasks exported
            [ ] idempotent (2nd run no dupes)  [ ] retry transient  [ ] partial-failure isolated  [ ] tests
Docs        [ ] TERMINAL_LOG.md (correct order)  [ ] DESIGN_NOTES.md  [ ] recording link in README
Git         [ ] full history, not squashed  [ ] seed data untouched
Recording   [ ] full session, terminal visible, narrated, Airtable shown, final tests green
```

---

## 17. Uncertainties / requires human verification

- **`REQUIRES HUMAN VERIFICATION`** — real Airtable field names/types. §10 assumes the fields in §9; if you name them differently, update `fields_for`. A `422` means a mismatch.
- **`REQUIRES HUMAN VERIFICATION`** — exact `pyairtable` 2.3 method names on your installed version. This guide uses `Api(key).table(base, name)` + `.all()/.create()/.update()`; if your version differs, check `python -c "import pyairtable, inspect; print(pyairtable.__version__)"` and the pyairtable docs. `batch_upsert` (if present) can replace the read-index-then-create loop for idempotency.
- **Node version:** repo targets Node 20; if you're on 18 and `npm install`/vitest warns, use Node 20 via nvm.
- **Which critical fix:** this guide fixes **SQL injection (#1)** for Part 2 because it leaks all credentials. Fixing the **PATCH authorization (#2)** instead is a valid alternative — §7 gives both.
- Everything marked "Observed" was reproduced live against a seeded instance during analysis; **re-capture the real output on your own machine** for the submission — do not paste these as if they were your run.
```
