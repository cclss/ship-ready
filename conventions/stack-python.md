# Python backend (FastAPI / Django / Flask)

Minimal run/deploy contract for Python web backends. This is a **floor, not a
ceiling**: it does not constrain your code design, architecture, ORM choice, or
testing strategy. Build whatever you want on top — these notes only ensure the
preview runtime can detect, run, and deploy your app.

The preview runtime detects a Python app from **two files together**:

1. a dependency declaration — `requirements.txt` (or a standard
   `pyproject.toml`), and
2. a `Procfile` at the project root with a `web:` line — your **declared start
   command**. There is no framework or entrypoint guessing: no `Procfile web:`,
   no server.

## What works well

- **Declare the start command in `Procfile`.** One line, using a production
  server, binding the injected `PORT`:

  ```procfile
  web: uvicorn main:app --host 0.0.0.0 --port $PORT
  ```

  Framework equivalents:

  ```procfile
  web: gunicorn myproject.wsgi --bind 0.0.0.0:$PORT     # Django / WSGI
  web: gunicorn app:app --bind 0.0.0.0:$PORT            # Flask
  ```

  Use a **production server** (`uvicorn`, `gunicorn`) — never dev tooling
  (`flask run`, `manage.py runserver`, `uvicorn --reload`) as the `web:`
  command. Listen plain HTTP; TLS is terminated upstream.
- **Declare dependencies, let the platform install them.** List packages in
  `requirements.txt`. Loose version constraints are fine — they only need to be
  resolvable. Do not commit `.venv/` and do not run your own `pip install` from
  build scripts; the platform creates the environment and installs from your
  declaration. If you want reproducible resolution, commit a lockfile
  (compiled `requirements.txt` or `uv.lock`) — recommended, not required.
- **Pin the runtime in `.tool-versions`.** A line like `python 3.12.5` tells
  the platform which interpreter to provision. Without a pin you get the
  platform default.
- **Bind the injected `PORT`.** Read it from the environment (the `$PORT` in
  your `Procfile` line). `0.0.0.0` works for both local preview and a deployed
  container. Never hardcode a number.
- **Serve static assets from the same process, out of `static/`.** Mount them
  the framework-standard way (FastAPI `StaticFiles`, Django staticfiles /
  WhiteNoise, Flask's `static/`). If you build a JS frontend, point its build
  output at `static/` too (e.g. `vite build --outDir ../static`) so one process
  serves everything on one port.
- **Datastores come from environment URLs.** A SQL database is reached via
  `DATABASE_URL` and a cache via `REDIS_URL`; the platform provisions isolated
  instances and injects these when it sees the matching dependency
  (`psycopg` / `sqlalchemy` / `django` for Postgres, `redis` for cache). Keep
  migrations file-based and idempotent (Django migrations, Alembic) and declare
  the commands in `preview.toml` under `[db]` (e.g.
  `migrate = "alembic upgrade head"` or `"python manage.py migrate"`) — Python
  migration commands are not auto-inferred. See `datastores.md`.
- **Declare app-specific env in `.env.example`.** For Python this committed
  example file is the declaration mechanism the platform reads (source scanning
  of `os.environ` reads is not performed) — required keys empty/placeholder,
  optional keys with real defaults, per `env-and-secrets.md`.

## Auto-injected environment

The platform injects these and your app may rely on them; do not hardcode or
override them. (See `golden-rules.md` for the full table.)

- `PORT` — the port to listen on.
- `HOST=127.0.0.1`, `HOSTNAME=127.0.0.1` — loopback bind address in preview;
  bind `0.0.0.0` so a deployed container also works.
- `SECRET_KEY_BASE` — a per-app secret, stable across a run.
- `DATABASE_URL` — present when your app uses a SQL database.
- `REDIS_URL` — present when your app uses a cache.

## Common failures

1. **No `Procfile web:` line.** With no declared start command there is nothing
   to run — detection fails by design. Add the one-liner.
2. **Hardcoded port** (`uvicorn ... --port 8000`, `app.run(port=5000)`). The
   health check probes the injected `PORT`, so a fixed port never comes up.
3. **Dev server as the start command.** `flask run` / `manage.py runserver` /
   `--reload` are dev tooling: single-threaded, debug-mode, and unfit for the
   deploy image's read-only rootfs. Use `uvicorn`/`gunicorn` in both `Procfile`
   and the deploy `CMD`.
4. **Static-site misdetection.** An `index.html` at the repo root, or inside
   `dist/`, `build/`, `out/`, or `public/`, makes the repo look like a static
   site and your server never starts. Keep static assets in `static/` (and
   templates in `templates/`) — the framework-standard locations avoid the
   collision naturally.
5. **Committed `.venv/` or install-from-script.** The platform owns dependency
   installation; a committed virtualenv or a hardcoded `pip install` in a build
   script fights it. Declare in `requirements.txt` and leave installation to
   the platform.
6. **Hardcoded datastore host** (`localhost:5432`, `localhost:6379`). Read
   `DATABASE_URL` / `REDIS_URL` from the environment instead.

## Checklist

- [ ] `Procfile` has a `web:` line starting a production server on `$PORT`,
      bound to `0.0.0.0`.
- [ ] Dependencies declared in `requirements.txt` (lockfile optional but
      recommended); `.venv/` not committed.
- [ ] `.tool-versions` pins the Python version.
- [ ] Static assets live in `static/`; no stray `index.html` at the root or in
      `dist`/`build`/`out`/`public`.
- [ ] Database/cache reached via `DATABASE_URL` / `REDIS_URL`; migrations
      file-based + idempotent, declared in `preview.toml [db]`.
- [ ] App-specific env declared in a committed `.env.example`.

If detection is ambiguous (monorepo, custom build, non-standard layout),
declare it explicitly in `preview.toml` — see `preview-toml.md`.
