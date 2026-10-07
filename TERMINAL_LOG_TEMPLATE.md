# TERMINAL_LOG.md (template — keep this exact order)

> Capture real output on your machine. Tip: `script -a terminal_log.txt` records the whole session.

## 1. Setup output
```
$ docker-compose up --build
...
$ docker-compose exec backend python manage.py migrate
... Applying projects.0001_initial... OK
$ docker-compose exec backend python manage.py seed
seed complete.
```

## 2. Initial test run
```
$ docker-compose exec backend python -m pytest
... 15 passed
$ docker-compose exec frontend npm test
... (vitest) passed
```

## 3. Bug curl proof — BEFORE fix (SQL injection)
```
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://localhost:8000/api/projects/$PID/tasks?q=%27" -H "Authorization: Bearer $TOKEN"
HTTP 500
$ curl -s "http://localhost:8000/api/projects/$PID/tasks?q=<UNION payload>" -H "Authorization: Bearer $TOKEN"
# rows leaking user emails + password hashes
```

## 4. Fix curl proof — AFTER fix
```
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://localhost:8000/api/projects/$PID/tasks?q=%27" -H "Authorization: Bearer $TOKEN"
HTTP 200
$ curl -s "http://localhost:8000/api/projects/$PID/tasks?q=<UNION payload>" -H "Authorization: Bearer $TOKEN"
{"tasks": []}
$ docker-compose exec backend python -m pytest -k search -v
... PASSED
```

## 5. Airtable export demo (first run)
```
$ curl -s -X POST http://localhost:8000/api/projects/$PID/export -H "Authorization: Bearer $TOKEN"
{"created": 7, "updated": 0, "failed": 0, "exported": 7, "errors": []}
```
<Airtable screenshot or share link here — 7 rows>

## 6. Second Airtable run (idempotency)
```
$ curl -s -X POST http://localhost:8000/api/projects/$PID/export -H "Authorization: Bearer $TOKEN"
{"created": 0, "updated": 7, "failed": 0, "exported": 7, "errors": []}
```
<Airtable screenshot — still 7 rows, no duplicates>

## 7. Part 3a (or 3b) demo
```
# member posts a comment -> 201
$ curl -s -X POST http://localhost:8000/api/tasks/$TID/comments -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{"body":"looks good"}'
# viewer tries to post -> 403
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8000/api/tasks/$TID/comments -H "Authorization: Bearer $DEV_TOKEN" -d '{"body":"x"}'
403
# list is chronological
$ curl -s http://localhost:8000/api/tasks/$TID/comments -H "Authorization: Bearer $TOKEN"
```

## 8. Final test run
```
$ docker-compose exec backend python -m pytest
... all passed (now includes search + comments + airtable tests)
$ docker-compose exec frontend npm test
... passed
```
