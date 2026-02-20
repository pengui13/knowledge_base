# Docker Compose — Explained

## Image vs Container

**Image is a class. Container is an instance of that class.**

**Image** is a static, read-only blueprint — it contains your code, dependencies, OS files, environment variables, everything baked in at build time. It sits on disk doing nothing. You can share it, push it to Docker Hub, or put it on another machine.

**Container** is a running instance of that image — the image brought to life. It has its own process, its own filesystem layer on top of the image, its own network.

```yaml
backend:
  build: .                      # builds the IMAGE from Dockerfile
  image: cryphos-app:latest     # names/tags that image

liquidations:
  image: cryphos-app:latest        # reuses the same image
  container_name: cryphos-liquidations  # but creates a separate container
```

---

## Dockerfile vs docker-compose.yaml

The Dockerfile defines everything baked into the image — OS, Python version, dependencies, env variables, working directory. Every container that runs from that image inherits it all automatically.

docker-compose.yaml can then **add or override** on top per container:

```yaml
celery:
  image: cryphos-app:latest      # inherits everything from Dockerfile
  environment:
    PYTHONPATH: /app              # adds/overrides env var
  working_dir: /app/apis         # overrides WORKDIR from Dockerfile
  command: celery -A apis worker # overrides CMD from Dockerfile
```

| | Purpose |
|---|---|
| **Dockerfile** | What's the same for everyone — base image, installed packages, copied code |
| **docker-compose.yaml** | What's different per container — command, extra env vars, volumes, ports |

- **Dockerfile** = class definition
- **docker-compose.yaml** = how you instantiate each object with different arguments

---

## container_name is optional

Without it, Docker auto-generates a name like `cryphos_liquidations_1`. With it you get a predictable name — purely for convenience when debugging:

```bash
docker logs cryphos-liquidations   # clean
docker logs cryphos_liquidations_1 # auto-generated, harder to remember
```

---

## Top level — `services:`

Everything under `services:` defines the containers that will run. Each key (`db`, `redis`, `backend`, etc.) is a service name — Docker Compose uses these names as **internal DNS hostnames** too. So inside any container, `redis` resolves to the Redis container's IP automatically. No hardcoded IPs needed.

---

## `db` service

```yaml
image: postgres:16-alpine
```
Official Postgres 16, alpine variant — minimal Linux base, smaller image.

```yaml
restart: always
```
If the container crashes or the host reboots, Docker restarts it automatically.

```yaml
environment:
  POSTGRES_DB: cryphos
  POSTGRES_USER: cryphos_user
  POSTGRES_PASSWORD: ${DB_PASSWORD}
```
Passes env vars into the container. Postgres reads these on first startup to create the database and user. `${DB_PASSWORD}` pulls the value from your `.env` file on the host — never hardcoded.

```yaml
volumes:
  - cryphos_pgdata:/var/lib/postgresql/data
```
Mounts a named volume to the path where Postgres stores its data. Without this, all your data disappears every time the container restarts. Named volumes (`cryphos_pgdata`) are managed by Docker and persist on the host.

```yaml
ports:
  - "127.0.0.1:5432:5432"
```
Format is `host:container`.

Without `ports`, Postgres is completely isolated — only other containers on the same Docker network can reach it. Your `backend`, `celery` etc. connect fine because they share the same Docker network. So for production you don't need to expose it at all.

With `ports` bound to `127.0.0.1`, you can connect from your **host machine** using tools like DBeaver, pgAdmin, `psql`, or Django shell locally. Binding to `127.0.0.1` specifically means only you on localhost can access it — not the public internet.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U cryphos_user -d cryphos"]
  interval: 5s
  timeout: 3s
  retries: 20
```
Runs `pg_isready` every 5 seconds to check if Postgres is actually accepting connections (not just running). This is what `condition: service_healthy` in other services checks against. Without this, a service could try to connect to Postgres before it's ready and crash.

---

## `backend` service

```yaml
build: .
```
Builds the image from your Dockerfile in the current directory. Only this service has `build:` — all others just reuse the built image.

```yaml
env_file: .env
```
Loads all variables from your `.env` file into the container. Different from `environment:` — that's for individual inline vars, `env_file` loads the whole file at once.

```yaml
depends_on:
  db:
    condition: service_healthy
  redis:
    condition: service_healthy
```
Waits for both `db` and `redis` to pass their healthchecks before starting. Without `condition: service_healthy` it would just wait for the container to start, not for the service inside to be ready — that's the important distinction.

```yaml
working_dir: /app/apis
environment:
  PYTHONPATH: /app
```
- `working_dir` — all commands run from `/app/apis` inside the container
- `PYTHONPATH: /app` — tells Python to look in `/app` when importing modules, so `from apis.something import x` resolves correctly

```yaml
volumes:
  - ./staticfiles:/app/staticfiles
  - .:/app
  - venv_data:/app/.venv
```
Three mounts:
- `./staticfiles:/app/staticfiles` — bind mount, syncs your local `staticfiles/` with the container. Django's `collectstatic` output persists on host
- `.:/app` — bind mount of your entire project into `/app`. Code changes on the host are reflected instantly in the container without rebuilding — great for development
- `venv_data:/app/.venv` — named volume for the virtualenv, persists across restarts and isn't overwritten by the bind mount above

```yaml
command: >
  sh -c "python manage.py migrate &&
         exec daphne -b 0.0.0.0 -p 8000 apis.asgi:application"
```
Runs migrations first, then starts Daphne (ASGI server for Django Channels/WebSockets). `exec` replaces the shell process with Daphne so it becomes PID 1 and receives signals properly — important for graceful shutdown.

---

## `liquidations`, `collector`, `klines_socket` services

```yaml
environment:
  REDIS_URL: redis://redis:6379/1
```
`redis` here is the service name — Docker's internal DNS resolves it to the Redis container automatically. `/1` means Redis database number 1, separate from the default `/0` used by Celery and Django.

These services override `REDIS_URL` because they're data collectors that need their own isolated Redis database, separate from the main app.

---

## `celery` service

```yaml
command: celery -A apis worker -l info --concurrency=2
```
- `-A apis` — the Celery app is defined in the `apis` module
- `worker` — starts a Celery worker
- `--concurrency=2` — runs 2 parallel worker processes. Low number, appropriate for a VPS with limited RAM

---

## `celery-beat` service

```yaml
volumes:
  - celerybeat_data:/data
command: celery -A apis beat -l info --schedule=/data/celerybeat-schedule
```
Celery Beat is the scheduler — it sends periodic tasks to workers on a schedule. The `--schedule` file tracks what was last run. Stored in a named volume so it persists across restarts — otherwise Beat would re-run tasks on every restart thinking they've never run.

---

## `tg` service

```yaml
command: python manage.py tg
stop_grace_period: 20s
```
Runs your Telegram bot as a Django management command. `stop_grace_period: 20s` gives the container 20 seconds to shut down gracefully before Docker force-kills it — so the bot can finish handling the current update cleanly.

---

## `volumes:` (top level)

```yaml
volumes:
  cryphos_pgdata:
  celerybeat_data:
  venv_data:
```
Declares the named volumes Docker manages. Must be listed here for Docker Compose to create them. Without this declaration, the volume references inside services would fail at startup.

---

## Quick reference

| Concept | What it does |
|---|---|
| `image:` | Which image to run |
| `build:` | Build image from Dockerfile |
| `container_name:` | Fixed name instead of auto-generated |
| `restart: always` | Auto-restart on crash or reboot |
| `env_file:` | Load whole `.env` file |
| `environment:` | Set individual env vars |
| `depends_on:` | Start order + health conditions |
| `ports:` | Expose container port to host |
| `volumes:` | Mount persistent storage |
| `working_dir:` | Override default working directory |
| `command:` | Override default CMD |
| `stop_grace_period:` | Graceful shutdown timeout |
| `healthcheck:` | How to verify service is ready |
