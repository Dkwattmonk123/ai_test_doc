# CODE_CHANGES.md — exact OLD → NEW edits

> Every code change you need, as **file → function → OLD → NEW**. Paths are relative to the cloned repo root (`q-taskboard/`). Type these yourself during the recording. Nothing here is auto-applied.
>
> **PDF note:** the assignment PDF says "official `airtable` **npm** package" and mentions `src/lib/airtable-mock.ts`. The **actual repo is Django/Python** — `pyairtable` is already in `backend/requirements.txt`, the export endpoint is Django (`POST /api/projects/:id/export`), and the repo README references `backend/projects/airtable_mock.py`. Follow the **repo** (Python + `pyairtable`). If asked on camera, say exactly that: "the PDF is the generic template; this repo is the Python variant, so I use pyairtable as the README instructs."

---

# CHANGE 1 — Fix SQL injection (Part 2, the #1 critical fix)

**File:** `backend/projects/views.py`
**Function:** `TaskListCreateView.get`

### 1a. Add an import (top of the file)

**OLD** (lines 1-7):
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.db import connection
from users.serializers import UserSerializer
from .models import Project, Membership, Task
from .serializers import ProjectDetailSerializer, TaskSerializer
```

**NEW:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.db.models import Q
from users.serializers import UserSerializer
from .models import Project, Membership, Task
from .serializers import ProjectDetailSerializer, TaskSerializer
```
> We removed `from django.db import connection` (it was only used by the vulnerable raw-SQL branch) and added `from django.db.models import Q` for the safe ORM filter.

### 1b. Replace the vulnerable method body

**OLD** (`TaskListCreateView.get`, the whole method — lines ~104-131):
```python
class TaskListCreateView(APIView):
    def get(self, request, project_id):
        membership = _get_membership(request.user, project_id)
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)

        q = request.query_params.get('q')
        if q:
            with connection.cursor() as cursor:
                sql = (
                    f"SELECT id, project_id, title, description, status, assignee_id, created_by_id, position, created_at, updated_at "
                    f"FROM tasks "
                    f"WHERE project_id = '{project_id}' "
                    f"AND (title ILIKE '%{q}%' OR description ILIKE '%{q}%') "
                    f"ORDER BY position ASC"
                )
                cursor.execute(sql)
                columns = [col[0] for col in cursor.description]
                rows = [dict(zip(columns, row)) for row in cursor.fetchall()]
            return Response({'tasks': rows})

        tasks = (
            Task.objects
            .filter(project_id=project_id)
            .select_related('assignee')
            .order_by('status', 'position')
        )
        return Response({'tasks': TaskSerializer(tasks, many=True).data})
```

**NEW:**
```python
class TaskListCreateView(APIView):
    def get(self, request, project_id):
        membership = _get_membership(request.user, project_id)
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)

        tasks = (
            Task.objects
            .filter(project_id=project_id)
            .select_related('assignee')
            .order_by('status', 'position')
        )
        q = request.query_params.get('q')
        if q:
            tasks = tasks.filter(Q(title__icontains=q) | Q(description__icontains=q))
        return Response({'tasks': TaskSerializer(tasks, many=True).data})
```

**Why:** the ORM parameterizes values, so `q` can never break out of the string. `project_id` stays scoped. Search now also returns the same serialized shape as the non-search path (the old raw branch returned raw DB rows — a latent inconsistency).

### 1c. Regression test — ADD to `backend/projects/tests.py`

Add these two methods inside the existing `class TestTasks:` (indent at method level):
```python
    def test_search_is_not_sql_injectable(self, auth_client, user):
        project = Project.objects.create(name='P', owner=user)
        Membership.objects.create(user=user, project=project, role='admin')
        Task.objects.create(project=project, title='Record demo video',
                            created_by=user, status='todo')
        # a lone quote must NOT 500
        resp = auth_client.get(f'/api/projects/{project.id}/tasks', {'q': "'"})
        assert resp.status_code == 200
        # a UNION payload must leak no user data
        payload = ("x') UNION SELECT id,id,email,password,'todo',NULL::uuid,"
                   "id,0,created_at,updated_at FROM users--")
        resp = auth_client.get(f'/api/projects/{project.id}/tasks', {'q': payload})
        assert resp.status_code == 200
        titles = [t['title'] for t in resp.data['tasks']]
        assert not any('@' in t for t in titles)

    def test_search_matches_title(self, auth_client, user):
        project = Project.objects.create(name='P', owner=user)
        Membership.objects.create(user=user, project=project, role='admin')
        Task.objects.create(project=project, title='Record demo video', created_by=user)
        Task.objects.create(project=project, title='Draft press release', created_by=user)
        resp = auth_client.get(f'/api/projects/{project.id}/tasks', {'q': 'demo'})
        assert resp.status_code == 200
        assert [t['title'] for t in resp.data['tasks']] == ['Record demo video']
```
Run: `python -m pytest projects/tests.py -k search -v` → fails before Change 1, passes after.

---

# CHANGE 2 — (Recommended hardening) Fix broken auth on task update

> Optional but cheap and strengthens the submission. If you only fix one bug for Part 2, Change 1 is the one. This closes Issue #2.

**File:** `backend/projects/views.py`
**Function:** `TaskDetailView.patch`

**OLD** (lines ~164-170):
```python
class TaskDetailView(APIView):
    def patch(self, request, task_id):
        try:
            task = Task.objects.get(id=task_id)
        except Task.DoesNotExist:
            return Response({'error': 'not found'}, status=status.HTTP_404_NOT_FOUND)

        if 'title' in request.data:
```

**NEW:**
```python
class TaskDetailView(APIView):
    def patch(self, request, task_id):
        try:
            task = Task.objects.select_related('project').get(id=task_id)
        except Task.DoesNotExist:
            return Response({'error': 'not found'}, status=status.HTTP_404_NOT_FOUND)

        membership = _get_membership(request.user, str(task.project_id))
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)
        if not _can_edit_tasks(membership.role):
            return Response({'error': 'viewers cannot edit tasks'}, status=status.HTTP_403_FORBIDDEN)

        if 'title' in request.data:
```
> Leave the rest of `patch` unchanged. This mirrors `TaskDetailView.delete`, which already does the membership + role check.

**Test — ADD to `backend/projects/tests.py`** inside `class TestTasks:`:
```python
    def test_patch_task_requires_edit_membership(self, client, user):
        owner = User.objects.create_user(email='owner@example.com', name='Owner', password='password123')
        project = Project.objects.create(name='P', owner=owner)
        Membership.objects.create(user=owner, project=project, role='admin')
        task = Task.objects.create(project=project, title='A task', created_by=owner)
        # meera is NOT a member of this project
        resp = client.post('/api/auth/login', {'email': 'meera@taskboard.dev', 'password': 'password123'}, format='json')
        client.credentials(HTTP_AUTHORIZATION=f"Bearer {resp.data['token']}")
        r = client.patch(f'/api/tasks/{task.id}', {'title': 'HACK'}, format='json')
        assert r.status_code == 403
```

---

# CHANGE 3 — Part 3a: Task comments (append-only)

All additions. No existing lines change except `projects/urls.py`.

### 3a. ADD model — `backend/projects/models.py` (append at end of file)
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
        ordering = ['created_at']
```
> `uuid`, `models`, and `settings` are already imported at the top of this file.

### 3b. ADD serializer — `backend/projects/serializers.py` (append at end)
```python
class CommentSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)

    class Meta:
        model = Comment
        fields = ['id', 'body', 'author', 'created_at']
```
And update the model import at the top:

**OLD:**
```python
from .models import Project, Membership, Task
```
**NEW:**
```python
from .models import Project, Membership, Task, Comment
```

### 3c. ADD view — `backend/projects/views.py` (append at end of file)
```python
class CommentListCreateView(APIView):
    def get(self, request, task_id):
        task = Task.objects.select_related('project').filter(id=task_id).first()
        if not task:
            return Response({'error': 'not found'}, status=status.HTTP_404_NOT_FOUND)
        if not _get_membership(request.user, str(task.project_id)):
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)
        comments = task.comments.select_related('author').all()
        return Response({'comments': CommentSerializer(comments, many=True).data})

    def post(self, request, task_id):
        task = Task.objects.select_related('project').filter(id=task_id).first()
        if not task:
            return Response({'error': 'not found'}, status=status.HTTP_404_NOT_FOUND)
        membership = _get_membership(request.user, str(task.project_id))
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)
        if not _can_edit_tasks(membership.role):
            return Response({'error': 'viewers cannot comment'}, status=status.HTTP_403_FORBIDDEN)
        body = (request.data.get('body') or '').strip()
        if not body:
            return Response({'error': 'body is required'}, status=status.HTTP_400_BAD_REQUEST)
        c = Comment.objects.create(task=task, author=request.user, body=body)
        return Response({'comment': CommentSerializer(c).data}, status=status.HTTP_201_CREATED)
```
And update the serializer import at the top of `views.py`:

**OLD:**
```python
from .serializers import ProjectDetailSerializer, TaskSerializer
```
**NEW:**
```python
from .serializers import ProjectDetailSerializer, TaskSerializer, CommentSerializer
```

### 3d. ADD route — `backend/projects/urls.py`

**OLD:**
```python
from django.urls import path
from .views import ProjectListCreateView, ProjectDetailView, TaskListCreateView, TaskDetailView, ExportView, MemberAddView

urlpatterns = [
    path('projects', ProjectListCreateView.as_view()),
    path('projects/<uuid:project_id>', ProjectDetailView.as_view()),
    path('projects/<uuid:project_id>/tasks', TaskListCreateView.as_view()),
    path('projects/<uuid:project_id>/members', MemberAddView.as_view()),
    path('projects/<uuid:project_id>/export', ExportView.as_view()),
    path('tasks/<uuid:task_id>', TaskDetailView.as_view()),
]
```

**NEW:**
```python
from django.urls import path
from .views import (ProjectListCreateView, ProjectDetailView, TaskListCreateView,
                    TaskDetailView, ExportView, MemberAddView, CommentListCreateView)

urlpatterns = [
    path('projects', ProjectListCreateView.as_view()),
    path('projects/<uuid:project_id>', ProjectDetailView.as_view()),
    path('projects/<uuid:project_id>/tasks', TaskListCreateView.as_view()),
    path('projects/<uuid:project_id>/members', MemberAddView.as_view()),
    path('projects/<uuid:project_id>/export', ExportView.as_view()),
    path('tasks/<uuid:task_id>', TaskDetailView.as_view()),
    path('tasks/<uuid:task_id>/comments', CommentListCreateView.as_view()),
]
```

### 3e. Migration (run in terminal)
```bash
python manage.py makemigrations projects   # creates projects/migrations/0002_comment.py
python manage.py migrate
```
(Docker: `docker-compose exec backend python manage.py makemigrations projects` then `migrate`.)

### 3f. Tests — ADD to `backend/projects/tests.py` (new class at end)
```python
@pytest.mark.django_db
class TestComments:
    def _login(self, client, email):
        r = client.post('/api/auth/login', {'email': email, 'password': 'password123'}, format='json')
        client.credentials(HTTP_AUTHORIZATION=f"Bearer {r.data['token']}")

    def test_member_can_post_and_list_chronologically(self, client, user):
        project = Project.objects.create(name='P', owner=user)
        Membership.objects.create(user=user, project=project, role='admin')
        task = Task.objects.create(project=project, title='T', created_by=user)
        self._login(client, 'meera@taskboard.dev')
        assert client.post(f'/api/tasks/{task.id}/comments', {'body': 'first'}, format='json').status_code == 201
        assert client.post(f'/api/tasks/{task.id}/comments', {'body': 'second'}, format='json').status_code == 201
        resp = client.get(f'/api/tasks/{task.id}/comments')
        assert resp.status_code == 200
        assert [c['body'] for c in resp.data['comments']] == ['first', 'second']

    def test_viewer_can_read_but_not_post(self, client, user):
        owner = User.objects.create_user(email='owner@example.com', name='Owner', password='password123')
        project = Project.objects.create(name='P', owner=owner)
        Membership.objects.create(user=owner, project=project, role='admin')
        Membership.objects.create(user=user, project=project, role='viewer')
        task = Task.objects.create(project=project, title='T', created_by=owner)
        self._login(client, 'meera@taskboard.dev')
        assert client.get(f'/api/tasks/{task.id}/comments').status_code == 200
        assert client.post(f'/api/tasks/{task.id}/comments', {'body': 'x'}, format='json').status_code == 403

    def test_non_member_forbidden(self, client, user):
        owner = User.objects.create_user(email='owner@example.com', name='Owner', password='password123')
        project = Project.objects.create(name='P', owner=owner)
        Membership.objects.create(user=owner, project=project, role='admin')
        task = Task.objects.create(project=project, title='T', created_by=owner)
        self._login(client, 'meera@taskboard.dev')
        assert client.get(f'/api/tasks/{task.id}/comments').status_code == 403
        assert client.post(f'/api/tasks/{task.id}/comments', {'body': 'x'}, format='json').status_code == 403
```

### 3g. Frontend — `frontend/src/components/TaskDetail.tsx`

Add a comments panel. The `members` prop is already passed in, so you can tell if the current user is a viewer. Minimal addition — inside the component, after the existing mutations, add:

```tsx
// at top with other imports
import { useQuery } from "@tanstack/react-query";
import { getStoredUser } from "@/lib/api-client";

// inside TaskDetail(), after deleteTask mutation:
const me = getStoredUser();
const myRole = members.find((m) => m.user.id === me?.id)?.role;
const canComment = myRole === "admin" || myRole === "member";
const [commentBody, setCommentBody] = useState("");

const comments = useQuery({
  queryKey: ["comments", task.id],
  queryFn: () => apiFetch<{ comments: { id: string; body: string; author: { name: string }; created_at: string }[] }>(
    `/api/tasks/${task.id}/comments`),
});
const addComment = useMutation({
  mutationFn: (body: string) =>
    apiFetch(`/api/tasks/${task.id}/comments`, { method: "POST", body: JSON.stringify({ body }) }),
  onSuccess: () => { setCommentBody(""); queryClient.invalidateQueries({ queryKey: ["comments", task.id] }); },
  onError: (err) => setError(err instanceof Error ? err.message : "comment failed"),
});
```
Then render below the buttons (before the closing `</div>`s):
```tsx
<div className="mt-6 border-t border-border pt-4">
  <h3 className="text-sm font-medium mb-2">comments</h3>
  <ul className="space-y-2 mb-3">
    {comments.data?.comments.map((c) => (
      <li key={c.id} className="text-sm">
        <span className="text-muted">{c.author.name} · {new Date(c.created_at).toLocaleString()}</span>
        <p>{c.body}</p>
      </li>
    ))}
    {comments.data?.comments.length === 0 && <li className="text-xs text-muted italic">no comments yet</li>}
  </ul>
  {canComment ? (
    <form onSubmit={(e) => { e.preventDefault(); if (commentBody.trim()) addComment.mutate(commentBody.trim()); }} className="flex gap-2">
      <input value={commentBody} onChange={(e) => setCommentBody(e.target.value)} placeholder="add a comment"
        className="flex-1 rounded-md bg-bg border border-border px-3 py-2 text-sm focus:border-accent focus:outline-none" />
      <button type="submit" disabled={addComment.isPending} className="text-sm px-4 py-2 rounded-md bg-accent text-white disabled:opacity-50">post</button>
    </form>
  ) : (
    <p className="text-xs text-muted italic">viewers cannot comment</p>
  )}
</div>
```

---

# CHANGE 4 — Part 3c: Real Airtable export

### 4a. ADD settings — `backend/taskboard/settings.py` (append at end)
```python
AIRTABLE_API_KEY = os.environ.get('AIRTABLE_API_KEY', '')
AIRTABLE_BASE_ID = os.environ.get('AIRTABLE_BASE_ID', '')
AIRTABLE_TABLE_NAME = os.environ.get('AIRTABLE_TABLE_NAME', 'Tasks')
```
> `os` is already imported at the top of settings.py.

### 4b. ADD new file — `backend/projects/airtable_client.py`
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
    """Retry transient failures (429/5xx/network) with bounded backoff.
    Permanent failures (other 4xx) are re-raised immediately."""
    attempt = 0
    while True:
        try:
            return fn()
        except RequestException as exc:
            code = _status_of(exc)
            transient = code in TRANSIENT_STATUS or code is None
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
    """Idempotent upsert keyed on TaskId, with per-record failure isolation."""
    table = table or get_table()
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
        except Exception as exc:
            failed += 1
            errors.append({'taskId': str(task.id), 'error': str(exc)})
    return {'created': created, 'updated': updated, 'failed': failed,
            'exported': created + updated, 'errors': errors}
```

### 4c. ADD new file — `backend/projects/airtable_mock.py`
```python
class MockTable:
    """In-memory stand-in for a pyairtable Table (unit tests only).
    Pass fail_ids to simulate permanent per-record failures."""
    def __init__(self, fail_ids=None):
        self._rows = {}
        self._seq = 0
        self.fail_ids = set(fail_ids or [])

    def all(self, fields=None):
        return [{'id': rid, 'fields': f} for rid, f in self._rows.items()]

    def create(self, fields):
        if fields.get('TaskId') in self.fail_ids:
            raise RuntimeError('permanent 422 for ' + str(fields.get('TaskId')))
        self._seq += 1
        rid = f'rec{self._seq}'
        self._rows[rid] = fields
        return {'id': rid, 'fields': fields}

    def update(self, rid, fields):
        if fields.get('TaskId') in self.fail_ids:
            raise RuntimeError('permanent 422 for ' + str(fields.get('TaskId')))
        self._rows[rid] = {**self._rows.get(rid, {}), **fields}
        return {'id': rid, 'fields': self._rows[rid]}
```

### 4d. Replace the stub — `backend/projects/views.py`, `ExportView.post`

**OLD** (lines ~234-243):
```python
class ExportView(APIView):
    def post(self, request, project_id):
        membership = _get_membership(request.user, project_id)
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)
        if not _can_edit_tasks(membership.role):
            return Response({'error': 'only admins and members can export'}, status=status.HTTP_403_FORBIDDEN)

        tasks = Task.objects.filter(project_id=project_id).select_related('assignee', 'created_by')
        return Response({'exported': 0, 'tasks': TaskSerializer(tasks, many=True).data})
```

**NEW:**
```python
class ExportView(APIView):
    def post(self, request, project_id):
        membership = _get_membership(request.user, project_id)
        if not membership:
            return Response({'error': 'forbidden'}, status=status.HTTP_403_FORBIDDEN)
        if not _can_edit_tasks(membership.role):
            return Response({'error': 'only admins and members can export'}, status=status.HTTP_403_FORBIDDEN)
        if not settings.AIRTABLE_API_KEY or not settings.AIRTABLE_BASE_ID:
            return Response({'error': 'airtable not configured'}, status=status.HTTP_503_SERVICE_UNAVAILABLE)

        tasks = (Task.objects.filter(project_id=project_id)
                 .select_related('assignee', 'created_by').order_by('position'))
        from .airtable_client import export_tasks
        try:
            summary = export_tasks(tasks)
        except Exception as exc:
            return Response({'error': f'export failed: {exc}'}, status=status.HTTP_502_BAD_GATEWAY)
        return Response(summary)
```
And add the settings import at the top of `views.py`:

**OLD** (line 4, after Change 1 it's the `Q` import — add below it):
```python
from django.db.models import Q
```
**NEW:**
```python
from django.db.models import Q
from django.conf import settings
```

### 4e. Pass Airtable env into the backend container — `docker-compose.yml`

**OLD** (the `backend:` → `environment:` block):
```yaml
    environment:
      POSTGRES_DB: taskboard
      POSTGRES_USER: taskboard
      POSTGRES_PASSWORD: taskboard
      POSTGRES_HOST: db
      POSTGRES_PORT: "5432"
      DJANGO_SECRET_KEY: dev-secret-change-me
      DEBUG: "true"
```

**NEW:**
```yaml
    environment:
      POSTGRES_DB: taskboard
      POSTGRES_USER: taskboard
      POSTGRES_PASSWORD: taskboard
      POSTGRES_HOST: db
      POSTGRES_PORT: "5432"
      DJANGO_SECRET_KEY: dev-secret-change-me
      DEBUG: "true"
      AIRTABLE_API_KEY: ${AIRTABLE_API_KEY}
      AIRTABLE_BASE_ID: ${AIRTABLE_BASE_ID}
      AIRTABLE_TABLE_NAME: ${AIRTABLE_TABLE_NAME:-Tasks}
```
> Docker Compose reads `${...}` from the `.env` file in the repo root automatically. Fill `AIRTABLE_API_KEY` and `AIRTABLE_BASE_ID` in `.env` first (yours is missing `AIRTABLE_BASE_ID`).

### 4f. Airtable tests — ADD to `backend/projects/tests.py` (new class at end)
```python
from projects.airtable_client import export_tasks
from projects.airtable_mock import MockTable

@pytest.mark.django_db
class TestAirtableExport:
    def _project_with_tasks(self, user, n=3):
        project = Project.objects.create(name='P', owner=user)
        Membership.objects.create(user=user, project=project, role='admin')
        tasks = [Task.objects.create(project=project, title=f'T{i}', created_by=user, position=i)
                 for i in range(n)]
        return project, tasks

    def test_exports_all_tasks(self, user, db):
        _, tasks = self._project_with_tasks(user, 3)
        table = MockTable()
        summary = export_tasks(tasks, table=table)
        assert summary['created'] == 3 and summary['failed'] == 0
        assert len(table.all()) == 3

    def test_idempotent_second_run_updates_not_duplicates(self, user, db):
        _, tasks = self._project_with_tasks(user, 3)
        table = MockTable()
        export_tasks(tasks, table=table)
        summary = export_tasks(tasks, table=table)   # run again, same table
        assert summary['created'] == 0 and summary['updated'] == 3
        assert len(table.all()) == 3                   # no duplicates

    def test_partial_failure_does_not_abort(self, user, db):
        _, tasks = self._project_with_tasks(user, 3)
        table = MockTable(fail_ids={str(tasks[1].id)})  # middle task fails permanently
        summary = export_tasks(tasks, table=table)
        assert summary['failed'] == 1 and summary['created'] == 2
        assert len(table.all()) == 2                    # the two good ones landed

    def test_export_authz_viewer_forbidden(self, client, user):
        owner = User.objects.create_user(email='owner@example.com', name='Owner', password='password123')
        project = Project.objects.create(name='P', owner=owner)
        Membership.objects.create(user=owner, project=project, role='admin')
        Membership.objects.create(user=user, project=project, role='viewer')
        r = client.post('/api/auth/login', {'email': 'meera@taskboard.dev', 'password': 'password123'}, format='json')
        client.credentials(HTTP_AUTHORIZATION=f"Bearer {r.data['token']}")
        resp = client.post(f'/api/projects/{project.id}/export')
        assert resp.status_code == 403
```
> These use `MockTable`, so they never hit the network. Production uses the real `Api` via `get_table()`.

### 4g. Frontend export button — `frontend/src/pages/ProjectPage.tsx`

Add a mutation and a button. After the `createTask` mutation inside `ProjectPage()`:
```tsx
const [exportMsg, setExportMsg] = useState<string | null>(null);
const exportTasks = useMutation({
  mutationFn: () => apiFetch<{ created: number; updated: number; failed: number }>(
    `/api/projects/${id}/export`, { method: "POST" }),
  onSuccess: (r) => setExportMsg(`exported ${r.created + r.updated} (new ${r.created}, updated ${r.updated}, failed ${r.failed})`),
  onError: (err) => setExportMsg(err instanceof Error ? err.message : "export failed"),
});
```
Render a button near the project header (inside the `{project && (...)}` block):
```tsx
<button onClick={() => exportTasks.mutate()} disabled={exportTasks.isPending}
  className="text-sm px-4 py-2 rounded-md bg-accent text-white disabled:opacity-50">
  {exportTasks.isPending ? "exporting…" : "export to Airtable"}
</button>
{exportMsg && <p className="text-xs text-muted mt-2">{exportMsg}</p>}
```
> Only admins/members can trigger it server-side; a viewer gets 403 and sees the error message.

---

# CHANGE 5 — README recording link (required)

**File:** `q-taskboard/README.md` — add near the top (replace the Loom URL with your real one):
```markdown
## Screen recording
Full session walkthrough: https://www.loom.com/share/<your-recording-id>
```

---

## Commit sequence (do not squash)
```bash
git add backend/projects/views.py backend/projects/tests.py
git commit -m "fix: parameterize task search to close SQL injection (Part 2)"

git add backend/projects/models.py backend/projects/serializers.py backend/projects/views.py \
        backend/projects/urls.py backend/projects/migrations/ frontend/src/components/TaskDetail.tsx
git commit -m "feat: append-only task comments with member-gated authz (Part 3a)"

git add backend/projects/airtable_client.py backend/projects/airtable_mock.py \
        backend/taskboard/settings.py backend/projects/views.py backend/projects/tests.py \
        docker-compose.yml frontend/src/pages/ProjectPage.tsx
git commit -m "feat: real pyairtable export — idempotent, retry transient, isolate record failures (Part 3c)"

git add REVIEW.md TERMINAL_LOG.md DESIGN_NOTES.md README.md
git commit -m "docs: review, terminal log, design notes, recording link"
```
End commit messages with the attribution line your tooling requires.
